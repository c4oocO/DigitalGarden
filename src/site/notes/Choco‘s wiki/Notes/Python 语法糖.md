---
{"dg-publish":true,"permalink":"/choco-s-wiki/notes/python/"}
---

[19 Sweet Python Syntax Sugar for Improving Your Coding Experience | by Yang Zhou | TechToFreedom | Medium](https://medium.com/techtofreedom/19-sweet-python-syntax-sugar-for-improving-your-coding-experience-37c4118fc6b1)
# 1. 联合运算符：合并 Python 字典的最优雅方式

在 Python 中合并多个字典的方法有很多种，但在 Python 3.9 发布之前，没有一种方法可以被描述为优雅。

例如，我们如何在 Python 3.9 之前合并以下三个字典？

其中一种方法是使用 for 循环：

```python
cities_us = {'New York City': 'US', 'Los Angeles': 'US'}  
cities_uk = {'London': 'UK', 'Birmingham': 'UK'}  
cities_jp = {'Tokyo': 'JP'}  
  
cities = {}  
  
for city_dict in [cities_us, cities_uk, cities_jp]:  
    for city, country in city_dict.items():  
        cities[city] = country  
  
print(cities)  
# {'New York City': 'US', 'Los Angeles': 'US', 'London': 'UK', 'Birmingham': 'UK', 'Tokyo': 'JP'}
```

它很体面，但远非优雅和 Pythonic。

Python 3.9 引入了联合运算符，这是一种语法糖，使合并任务变得超级简单：
```python
cities_us = {'New York City': 'US', 'Los Angeles': 'US'}  
cities_uk = {'London': 'UK', 'Birmingham': 'UK'}  
cities_jp = {'Tokyo': 'JP'}  
  
cities = cities_us | cities_uk | cities_jp  
  
print(cities)  
# {'New York City': 'US', 'Los Angeles': 'US', 'London': 'UK', 'Birmingham': 'UK', 'Tokyo': 'JP'}

如上面的程序所示，我们可以使用一些管道符号，在这种情况下称为联合运算符，来合并任意数量的 Python 字典。

是否有可能通过工会运营商进行就地合并？

cities_us = {'New York City': 'US', 'Los Angeles': 'US'}  
cities_uk = {'London': 'UK', 'Birmingham': 'UK'}  
cities_jp = {'Tokyo': 'JP'}  
  
cities_us |= cities_uk | cities_jp  
print(cities_us)  
# {'New York City': 'US', 'Los Angeles': 'US', 'London': 'UK', 'Birmingham': 'UK', 'Tokyo': 'JP'}
```
当然，只需将联合运算符移动到等号的左侧，如上面的代码所示。

# 2. 类型提示：使您的 Python 程序类型安全

动态类型化，即在运行时确定变量的类型，是使 Python 灵活方便的关键特性。但是，如果变量类型不正确，也可能导致隐藏的错误和错误。

为了解决这个问题，Python 在 3.5 版本中引入了键入提示功能。它提供了一种在代码中注释变量类型的方法，现代 IDE 可以在开发过程中为开发人员及早发现类型错误。

例如，如果我们将一个变量定义为整数，但将其更改为字符串，如下所示，IDE（在本例中为 PyCharm）将为我们突出显示意外的代码：

![A sreenshot to show how PyCharm highlights the unexpected assignment based-on Python type hints](https://miro.medium.com/v2/resize:fit:700/0*sAtpn9ukuIBb21A2.png)

PyCharm 根据类型提示突出显示意外赋值

除了原始类型之外，还有一些高级类型提示技巧。

==例如，在 Python 中使用所有大写字母定义常量是一种常见的约定。==

DATABASE = 'MySQL'

但是，这只是一种约定，没有人可以阻止您为这个“常量”分配新值。

为了改进它，我们可以使用 `Final` 类型提示，它指示变量旨在成为常量值，不应重新赋值：
```python
from typing import Final  
DATABASE: Final = "MySQL"
```
如果我们真的更改了“常量”，IDE肯定会提醒我们：

![An screenshot to show how PyCharm highlights the code that changes a constant](https://miro.medium.com/v2/resize:fit:700/1*Xp6dVpf-PX8a4XioFZU42A.png)

PyCharm 突出显示更改常量的代码

# 3. F-Strings：一种 Pythonic 字符串格式化方法

Python 支持几种不同的字符串格式化技术，例如使用 `%` 符号的 C 样式格式化、内置 `format()` 函数和 f 字符串。

除非您仍在使用比 Python 3.6 更旧的版本，否则 f-strings 绝对是执行字符串格式的最 Python 方式。因为它们可以用最少的代码完成所有格式化任务，甚至可以在字符串中运行表达式。
```python
from datetime import datetime  
  
today = datetime.today()  
  
print(f"Today is {today}")  
# Today is 2023-03-22 21:52:29.623619  
  
print(f"Today is {today:%B %d, %Y}")  
# Today is March 22, 2023  
  
print(f"Today is {today:%m-%d-%Y}")  
# Today is 03-22-2023
```
如上面的代码所示，使用 f 字符串只需做两件事：

1. 在字符串前添加字母“f”以指示它是 f 字符串。
2. 使用带有变量名称的大括号和字符串 （ `{variable_name:format}` ） 内的可选格式说明符，以特定格式插值变量的值。

“Simple is better than complex.”f 字符串很好地反映了 Python 禅宗中的这句话。

更重要的是，我们可以直接在 f 字符串中执行一个表达式：
```python
from datetime import datetime  
  
print(f"Today is {datetime.today()}")  
# Today is 2023-03-22 22:00:32.405462
```
# 4. 使用省略号作为不成文代码的占位符

在 Python 中，我们通常将 `pass` 关键字作为未编写代码的占位符。但是我们也可以使用省略号来实现此目的。
```python
def write_an_article():  
    ...  
  
  
class Author:  
    ...
```
Python 之父 Guido van Rossum 将这种语法糖添加到 Python 中，因为他认为它很可爱。

# 5. Python 中的装饰器：一种模块化功能和分离关注点的方法

Python 装饰器的想法是允许开发人员在不修改其原始逻辑的情况下向现有对象添加新功能。

我们可以自己定义装饰器。并且还有许多精彩的内置装饰器可供使用。

例如，Python 类中的静态方法不绑定到实例或类。它们包含在类中只是因为它们在逻辑上属于该类。

要定义静态方法，我们只需要使用 `@staticmethod` 装饰器，如下所示：
```python
class Student:  
    def __init__(self, first_name, last_name):  
        self.first_name = first_name  
        self.last_name = last_name  
        self.nickname = None  
  
    def set_nickname(self, name):  
        self.nickname = name  
  
    @staticmethod  
    def suitable_age(age):  
        return 6 <= age <= 70  
  
  
print(Student.suitable_age(99)) # False  
print(Student.suitable_age(27)) # True  
print(Student('yang', 'zhou').suitable_age(27)) # True
```
# 6.列表推导：在一行代码中列出一个列表

Python 以其简洁性而闻名，这在很大程度上归功于其精心设计的语法糖，例如列表推导。

通过列表推导，我们可以将 for 循环和 if 条件都放在一行代码中来生成 Python 列表：
```python
Genius = ["Yang", "Tom", "Jerry", "Jack", "tom", "yang"]  
L1 = [name for name in Genius if name.startswith('Y')]  
L2 = [name for name in Genius if name.startswith('Y') or len(name) < 4]  
L3 = [name for name in Genius if len(name) < 4 and name.islower()]  
print(L1, L2, L3)  
# ['Yang'] ['Yang', 'Tom', 'tom'] ['tom']
```
此外，Python 中还有集合、字典和生成器推导式。它们的语法类似于列表推导。

例如，以下程序在字典推导的帮助下，根据某些条件生成一个字典：
```python
Entrepreneurs = ["Yang", "Mark", "steve", "jack", "tom"]  
D1 = {id: name for id, name in enumerate(Entrepreneurs) if name[0].isupper()}  
print(D1)  
# {0: 'Yang', 1: 'Mark'}
```
# 7. 用于定义小型匿名函数的 Lambda 函数

lambda 函数，或称为匿名函数，是语法糖，用于在 Python 中轻松定义一个小函数，使您的代码更整洁、更短。

lambda 函数的一个常见应用是使用它来定义内置 `sort()` 函数的比较方法：
```python
leaders = ["Warren Buffett", "Yang Zhou", "Tim Cook", "Elon Musk"]  
leaders.sort(key=lambda x: len(x))  
print(leaders)  
# ['Tim Cook', 'Yang Zhou', 'Elon Musk', 'Warren Buffett']
```
在上面的示例中，一个 lambda 函数被定义为用作对列表进行排序的比较方法，该函数接收变量并返回其长度。当然，我们可以在这里以正常的方式编写一个完整的函数。但鉴于这个函数非常简单，把它写成 lambda 函数肯定更短更整洁。

# 8. 三元条件运算符：将 If 和 Else 放入一行代码中

许多编程语言都有三元条件运算符。Python 对此的语法只是将 `if` 和 `else` 放入同一行：
```python
short_one = a if len(a) < len(b) else b
```
如果我们在没有三元条件语法的情况下实现与上述相同的逻辑，则需要几行代码：
```python
short_one = ''  
if len(a) < len(b):  
    short_one=a  
else:  
    short_one=b
```
# 9. 使用“枚举”方法优雅地迭代列表

在某些情况下，我们需要在迭代时同时使用列表中元素的索引和值。

经典的 C 风格方法如下所示：
```python
for (int i = 0; i < len_of_list; i++) {  
        printf("%d %s\n", i, my_list[i]);  
    }
```
我们可以在 Python 中编写类似的逻辑，但 `my_list[i]` 这似乎有点难看，尤其是当我们需要多次调用元素的值时。

真正的 Pythonic 方法是使用函数 `enumerate()` 直接获取索引和值：
```python
leaders = ["Warren", "Yang", "Tim", "Elon"]  
for i,v in enumerate(leaders):  
    print(i, v)  
# 0 Warren  
# 1 Yang  
# 2 Tim  
# 3 Elon
```
# 10. 上下文管理器：自动关闭资源

众所周知，一旦文件被打开和处理，立即关闭它以释放内存资源是很重要的。忽视这样做可能会导致内存泄漏，甚至使我们的系统崩溃。
```python
f = open("test.txt", 'w')  
f.write("Hi,Yang!")  
# some logic here  
f.close()
```
在 Python 中处理文件相当容易。如上面的代码所示，我们只需要始终记住调用该 `f.close()` 方法来释放内存即可。

但是，当程序变得越来越复杂和庞大时，没有人能总是记住一些东西。这就是 Python 提供上下文管理器语法糖的原因。
```python
with open("test.txt", 'w') as f:  
    f.write("Hi, Yang!")
```
如上图所示，“with”语句是 Python 上下文管理器的关键。只要我们通过它打开一个文件，处理它下面的文件，文件处理后就会自动关闭。

# 11. Python 列表的花式切片技巧

从列表中获取部分项目是常见的要求。在 Python 中，slice 运算符由三个组件组成：
```python
a_list[start:end:step]
```
- “start”：起始索引（默认值为 0）。
- “end”：结束索引（默认值为列表的长度）。
- “step”：定义遍历列表时的步长（默认值为 1）。

基于这些，有一些技巧可以使我们的代码如此整洁。

## 使用切片技巧反转列表

由于切片运算符可以是负数（-1 是最后一项，依此类推），我们可以利用此功能以这种方式反转列表：
```python
a = [1,2,3,4]  
print(a[::-1])  
# [4, 3, 2, 1]

## 获取列表的浅副本

>>> a = [1, 2, 3, 4, 5, 6]  
>>> b = a[:]  
>>> b[0]=100  
>>> b  
[100, 2, 3, 4, 5, 6]  
>>> a  
[1, 2, 3, 4, 5, 6]
```
与 `b=a[:]` 不同 `b=a` ，因为它分配的是 的浅拷贝 `a` ，而不是 `a` 本身。因此，如上所示，这些 `b` 更改根本不会影响 `a` 。

# 12. Walrus 运算符：表达式中的赋值

walrus 运算符，也称为赋值表达式运算符，是在 Python 3.8 中引入的，由 `:=` 符号表示。

它用于为表达式中的变量赋值，而不是单独赋值。

例如，请考虑以下代码：
```python
while (line := input()) != "stop":  
    print(line)
```
在此代码中，walrus 运算符用于将用户输入的值分配给变量 `line` ，同时还检查输入是否为“停止”。只要用户输入不是“停止”，while 循环就会继续运行。

顺便说一句，walrus 运算符 `:=` ，，从海象的眼睛和獠牙中得名：

![A walrus](https://miro.medium.com/v2/resize:fit:700/0*a-m6VUGnATX-a8vW.jpeg)


# 13. 连续比较：一种更自然的写出 if 条件的方法

在 Java 或 C 等语言中，有时需要编写如下条件：

```python
if (a > 1 && a < 10){  
  //do somthing  
}
```
但是，如果您不使用 Python，则无法像下面这样优雅地编写它：

```python
if 1 < a < 10:  
    ...
```

是的，Python 允许我们编写连续比较。它使我们的代码看起来就像我们在数学中编写代码一样自然。

# 14. Zip 功能：轻松组合多个可迭代对象

Python 有一个内置 `zip()` 函数，该函数将两个或多个可迭代对象作为参数，并返回一个聚合可迭代对象中的元素的迭代器。

例如，在 `zip()` 函数的帮助下，以下代码将 3 个列表聚合为一个列表，没有任何循环：

```
id = [1, 2, 3, 4]  
leaders = ['Elon Mask', 'Tim Cook', 'Bill Gates', 'Yang Zhou']  
sex = ['male', 'male', 'male', 'male']  
record = zip(id, leaders, sex)  
  
print(list(record))  
# [(1, 'Elon Mask', 'male'), (2, 'Tim Cook', 'male'), (3, 'Bill Gates', 'male'), (4, 'Yang Zhou', 'male')]
```

# 15. 直接交换两个变量

交换两个变量通常是初学者在打印“Hello world！”后编写的第一个程序。

在许多编程语言中，执行此操作的经典方法需要一个临时变量来存储其中一个变量的值。

例如，您可以在 Java 中交换两个整数，如下所示：

```python
int a = 5;  
int b = 10;  
int temp = a;  
a = b;  
b = temp;  
System.out.println("a = " + a); // Output: a = 10  
System.out.println("b = " + b); // Output: b = 5
```

在 Python 中，语法非常直观和优雅：

```python
a = 10  
b = 5  
a, b = b, a  
print(a, b)  
# 5 10
```

# 16. 解构赋值技巧

在 Python 中解构赋值是一种将迭代器或字典的元素分配给单个变量的方法。它允许您编写更短、更易读的代码，避免使用索引或键访问单个元素。

```python
person = {'name': 'Yang', 'age': 30, 'location': 'Mars'}  
name, age, loc = person.values()  
print(name, age, loc)  
# Yang 30 Mars
```


如上例所示，我们可以在一行代码中直接将字典的值分配给 3 个单独的变量。

但是，如果左侧只有两个变量，如何接收assignments？

Python 为此提供了另一种语法糖：

```python
person = {'name': 'Yang', 'age': 30, 'location': 'Mars'}  
name, *others = person.values()  
print(name, others)  
# Yang [30, 'Mars']
```

如上所示，我们可以在变量之前添加一个星号，让它从`person.values()` 接收所有剩余的变量。

简单而优雅，不是吗？

# 17. 带星号的 Iterables 解包

除了解构赋值之外，Python 中的星号也是可迭代解包的关键。

```python
A = [1, 2, 3]  
B = (4, 5, 6)  
C = {7, 8, 9}  
L = [*A, *B, *C]  
print(L)  
# [1, 2, 3, 4, 5, 6, 8, 9, 7]
```

如上图所示，将列表、集合和元组合并为一个列表的最简单方法是通过新列表中的星号解压缩它们。

# 18. Any（） 和 All（） 函数

在某些情况下，我们需要检查可迭代对象（例如列表、元组或集合）中的任何或所有元素是否为 true。

当然，我们可以使用 for 循环来逐个检查它们。但是 Python 提供了两个内置函数 `any()` ，并 `all()` 简化了这两个操作的代码。

例如，以下程序用于 `all()` 确定列表的所有元素是否都是奇数：
```python
my_list = [3, 5, 7, 8, 11]  
all_odd = all(num % 2 == 1 for num in my_list)  
print(all_odd)  
# False
```

以下代码使用该 `any()` 函数检查是否存在名称以“Y”开头的领导者：
```python
leaders = ['Yang', 'Elon', 'Sam', 'Tim']  
starts_with_Y = any(name.startswith('Y') for name in leaders)  
print(starts_with_Y)  
# True
```

# 19. 数字下划线

计算一个大数字中有多少个零是一件令人头疼的事情。

幸运的是，Python 允许在数字中包含下划线以提高可读性。

例如，我们可以用 Python 编写，而不是编写 `10000000000` `10_000_000_000` ，这更容易阅读。
