---
{"dg-publish":true,"permalink":"/choco-s-wiki/notes/java-agent/"}
---

Java Agent 是一种强大的工具，允许开发者在 Java 应用程序运行时对其字节码进行修改。借助 Java Agent，我们可以动态拦截和增强程序的行为，而无需手动修改源代码或重新编译。
由此可见，这确实是一种非常方便、
强大的注入内存马的方式。

> 本文章大部分内容来自 [Java Agent 从入门到内存马](https://xz.aliyun.com/t/9450)，再加上个人的实践经验
## Premain 方式（启动时加载）
在 JVM 启动时通过 -javaagent 参数加载。Premain 方法会在主程序的 main 方法之前执行，用于对类加载过程进行初始化或增强。

**特性**
- 在 JVM 启动时通过 -javaagent 参数注入
- 在 main 方法执行之前执行
- 可以修改已加载和未加载的类字节码

**preagent**代码：
```java
package com.choco.demo;  
  
import java.lang.instrument.Instrumentation;  
  
public class PreDemo {  
    public static void premain(String args, Instrumentation inst) throws Exception{  
        for (int i = 0; i < 10; i++) {  
            System.out.println("hello I`m premain agent!!!");  
        }    
    }
}
```

**被注入程序**代码：
```java
package org.example;  
 
public class Main {  
    public static void main(String[] args) {  
 
        System.out.printf("Hello and welcome!");  
  
        for (int i = 1; i <= 5; i++) {    
            System.out.println("i = " + i);  
        }
    }
}
```

这里需要将两段代码编译jar包运行。
如果不会的话，可以往下看，会了直接跳第二段。
编译jar包，朴素一点可以用命令，但命令写起来有点复杂，还是用强大的IDEA吧！从头开始：
1. 新建项目-> 构建系统选maven -> 选一个你有的jdk -> 添加示例代码
2. 在`src/main/java`下新建类，名称随意，比如`com/choco/demo/PreDemo.java`
3. 在`src/main/resources/META-INF/MANIFEST.MF`文件中添加:
```
Manifest-Version: 1.0  
Premain-Class: com.choco.demo.PreDemo

```
4. 点击IDEA工具栏 选择`Project Structure` -> `Artifacts` -> `JAR` -> `From modules with dependencies` 默认即可
5. 选择`Build` -> `Build Artifacts` -> `Build`。
这样可以看到在out目录中编译出来的jar包，可以在`MANIFEST.MF`中通过`Main-Class: org.example.Main`参数指定打包main.class文件


**运行结果**
![Pasted image 20241219145153.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241219145153.png)
流程如下：
![Pasted image 20241219145301.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241219145301.png)


有没有发现，Premain 只能在启动时通过 -javaagent 参数指定，在实际环境中我们总不能先把人家的服务停掉再注入内存马吧，所以需要在程序运行过程中动态注入 Agent，这就是 Agentmain 的用武之地。

## Agentmain 方式（运行时加载）
与 Premain 相比，Agentmain 更加灵活。它允许在 JVM 运行时动态加载 Agent，并修改已加载的类字节码。

**特性**
• 可在 JVM 运行时加载 Agent。
• 依赖 Attach 机制注入。
• 只能修改已加载的类的字节码。

### Attach 机制：

Attach 机制是 JVM 提供的一种进程间通信能力，允许一个 JVM 进程附加到另一个 JVM 进程上，并注入包含 Agentmain 方法的 JAR 文件。可以理解为，Attach 是 Agentmain 的运载工具。

1. **创建 Java Agent**：编写 agentmain 方法，并打包为 JAR。
```java
package com.choco.demo;  
  
import java.lang.instrument.Instrumentation;  
  
public class AgentDemo {  
    public static void agentmain(String agentArgs, Instrumentation inst) {  
        for (int i = 0; i < 10; i++) {  
            System.out.println("hello I`m agentMain!!!");  
        }    
    }
}
```
2. **创建 AttachAgent**：编写用于动态注入代理的 AttachAgent 类。
```java
package com.choco.demo;  
  
import com.sun.tools.attach.AgentInitializationException;  
import com.sun.tools.attach.AgentLoadException;  
import com.sun.tools.attach.AttachNotSupportedException;  
import com.sun.tools.attach.VirtualMachine;  
  
import java.io.IOException;  
  
public class AgentMain {  
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {  
        // 获取目标 JVM 的 PID 和 Agent JAR 文件路径  
        String pid = args[0];  
        // 目标 JVM 进程的 PID        
        String jarPath = args[1];  
        // 包含 agentmain 方法的 JAR 文件路径  
  
        System.out.println("Target JVM PID: " + pid);  
        System.out.println("Agent JAR Path: " + jarPath);  
  
        // 附加到目标 JVM        
        VirtualMachine vm = VirtualMachine.attach(pid);  
  
        // 加载 agent 到目标 JVM 中  
        vm.loadAgent(jarPath);  
  
        // 断开连接  
        vm.detach();  
  
        System.out.println("Agent loaded successfully!");  
    }
}
```

需修改`MANIFEST.MF`文件内容为：
```
Manifest-Version: 1.0  
Premain-Class: com.choco.demo.PreDemo  
Agent-Class:  com.choco.demo.AgentDemo

```
注意最后一定要空一行。
然后 IDEA菜单栏【构建】【编译 Artifacts..】获得agent的jar包。

3. **注入与验证**：

使用 AttachAgent 指定目标 JVM 的 PID 和 Agent JAR 路径，完成注入。
jar包要复制到被注入进程的同一目录下，要不然被注入进程不知道从哪里加载agent。
这是IDEA中的运行配置，直接运行attch的java文件，便于操作：
![Pasted image 20241219182429.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241219182429.png)
结果输出：
- 被注入的进程（tomcat）

![Pasted image 20241219182358.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241219182358.png)
- Attch的输出
![Pasted image 20241219182418.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241219182418.png)


> tips: 可通过命令JPS来查看Java进程的pid

## `Instrumentation`：Agent 的核心能力
无论是 `Premain` 还是 `Agentmain`，它们的核心都离不开 `Instrumentation` 接口。这是 Java 提供的一个强大机制，用于动态修改和检测类字节码。

**主要功能**
- 动态修改已加载类的字节码。
- 定义新的类。
- 获取 JVM 中所有已加载类的信息。
- 获取对象的大小。
- 添加类转换器（`ClassFileTransformer`）。

**主要 API**
```java
// 获取所有已加载的类
Class<?>[] getAllLoadedClasses();

// 重定义类
redefineClasses(ClassDefinition... definitions);

// 添加类转换器
addTransformer(ClassFileTransformer transformer);

// 获取对象大小
long getObjectSize(Object objectToSize);
```
通过 Instrumentation，我们可以对程序进行深度观察和控制。

## `Javassist`：字节码增强
在实现字节码增强时，直接操作字节码指令往往显得过于复杂。这时，我们可以借助 Javassist——一个操作字节码的高层次库，它允许我们用接近 Java 源代码的方式修改类。
在 Java Agent 中，通过 Instrumentation 拦截到类加载的事件后，可以利用 Javassist 修改字节码。
Javassist 提供了一个高级 API，允许开发者以类似 Java 源代码的方式来修改字节码，而不需要直接理解或操作字节码的低级指令。它在 Agent 中通常与 Instrumentation 对象配合使用。

**Javassist 的功能：**
- 动态修改类的方法、字段等。
- 添加新方法、字段或注解。
- 替换方法的具体实现。
- 动态生成新类。

**示例：动态修改方法**
1. **被注入的类**：

```java
package org.example;  
  
import java.util.Scanner;  
  
 
public class Main {  
    public static void main(String[] args) throws InterruptedException {  
        //打印当前路径  
        System.out.println(System.getProperty("user.dir"));  
        System.out.println("PID: " + ProcessHandle.current().pid());  
        out();  
  
        Thread.sleep(30000);  
        out();  
    }  
    public static void out() {  
        System.out.println("Hello, World!");  
    }}
```

2. **Agent 使用 Javassist 修改方法**：
```java
package com.choco.demo;  
  
import javassist.*;  
import java.lang.instrument.ClassFileTransformer;  
import java.lang.instrument.Instrumentation;  
import java.security.ProtectionDomain;  
  
public class AgentDemo {  
    public static void agentmain(String agentArgs, Instrumentation inst) {  
        System.out.println("Agent agentmain is running...");  
  
        inst.addTransformer(new ClassFileTransformer() {  
            @Override  
            public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,  
                                    ProtectionDomain protectionDomain, byte[] classfileBuffer) {  
                System.out.println("Attempting to transform: " + className);  
                if ("org/example/Main".equals(className)) {  
                    System.out.println("Transforming class: " + className);  
                    try {  
                        ClassPool classPool = ClassPool.getDefault();  
                        classPool.insertClassPath(new LoaderClassPath(loader));  
                        CtClass ctClass = classPool.get("org.example.Main");  
  
                        // 修改 main 方法  
                        CtMethod mainMethod = ctClass.getDeclaredMethod("out");  
                        mainMethod.insertBefore("{ System.out.println(\"Hello, Agent injected successfully before execution!\"); }");  
                        mainMethod.insertAfter("{ System.out.println(\"Hello, Agent injected successfully after execution!\"); }");  
                        byte[] byteCode = ctClass.toBytecode();  
                        System.out.println("Bytecode modified. Length: " + byteCode.length);  
                        return byteCode;  
                    } catch (Exception e) {  
                        e.printStackTrace();  
                    }                }                return null; // 不修改其他类  
            }  
        }, true);  
  
        try {  
            Class<?> targetClass = Class.forName("org.example.Main");  
            System.out.println("Retransforming class: " + targetClass.getName());  
            inst.retransformClasses(targetClass);  
        } catch (Exception e) {  
            e.printStackTrace();  
        }   
    }
}

```

这个方法需要引入 Javassist 库。两种方法，一个是构建工具Maven或者gradle添加依赖；还有就是直接在idea中添加：
1. 打开 `File` > `Project Structure` > `Libraries`。
2. 点击 + 添加一个新的库（我选的`来自maven` 没啥问题），选择下载的` javassist-<version>.jar` 文件。
3. 点击 OK 并应用更改。
注意`MANIFEST.MF`需要添加
```
Can-Redefine-Classes: true  
Can-Retransform-Classes: true
```

**运行结果**
相同的，先修改AgentMain的代码，【构建】->【编译 Artifacts..】-> 复制jar包到运行目录（当然写jar包的路径也是可以的）
运行main程序，然后眼疾手快复制pid，作为attch的的运行参数：
![Pasted image 20241225175743.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241225175743.png)
main中的结果输出：
![Pasted image 20241225153442.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020241225153442.png)
可以看第二次调用main中的out方法，输出了三个`Hello, ...`，修改类方法成功。


## 总结

1. **Premain 和 Agentmain**：分别适合启动时和运行时的 Agent 加载。
2. **Attach 机制**：为动态注入 Agent 提供可能。
3. **Instrumentation 和 Javassist**：一个是能力核心，一个是操作工具。