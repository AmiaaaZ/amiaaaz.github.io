---
title: "Java学习笔记ⅩⅢ"
slug: "java-study-notes-13"
description: "对于易混淆点的纠正"
date: 2023-07-20T01:47:58+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true

---

太久不看java，已经忘得一干二净 手感全无了，最近重新拾起来复建发现了一些从未发现的盲点或一些当时没有彻底理解透彻的地方，遂整理一篇；内容相当基础，新手友好

----

## 动态代理

在学习他人文章、跟着一步步复现时我们经常会不自觉地把大脑交给所看的文章来接管，还会自动略过文章中未详细说明的细节，这样不求甚解很容易造成偏听则暗的后果，此处动态代理就是一例

### CC1的LazyMap

- **Map版CC1**：AnnotationInvocationHandler#readObject-> memberValue.setValue(Map)-> RCE
- **LazyMap版CC1**：AnnotationInvocationHandler#readObject-> memberValue.setValue(Map)-> AnnotationInvocationHandler#invoke(proxy)-> LazyMap#get-> RCE

这里反驳一下p牛在java安全漫谈11中的这两段话

![image-20230713235555818](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230713235555818.png)

![image-20230713235613928](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230713235613928.png)

1. 被代理的不是“对象”而是“接口”

动态代理指的是利用反射在运行时创建一个实现某些给定接口的新类（即动态代理类）及其实例，举例：

```java
Foo f = (Foo)Proxy.newProxyInstance(Foo.class.getClassLoader(), new Class[]{Foo.class}, handler);
```

这里我们对`Foo`接口进行代理，同时在没有实现类的情况下动态创建了一个接口对象`f`，当调用`f.func("abc")`时会移交给`handler.invoke()`进行处理

2. 被代理的是`Map` 并不是`AnnotationInvocationHandler`

最直观的判别方法：被代理的接口是谁，创建的接口实例就会被强制转型成谁，传入的参数也会是与被代理接口强相关，这里是

```java
Map proxyMap = (Map)Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler)
```

毫无疑问是对`Map`接口进行代理

3. LazyMap的触发与setValue有关

Map被代理后需要有调用Map.xxx的地方才会触发`AnnotationInvocationHandler#invoke`，调用Map.xxx的地方就在`AnnotationInvocationHandler#readObject`中，也就是Map版CC1的触发点

### JDK7u21的Templates

理由同上，p牛在Java安全漫谈18中的表述我认为也是存在错误的

![image-20230718180825094](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230718180825094.png)

这里并不是对`AnnotationInvocationHandler`做代理，而是对`Templates`做代理

**实际反序列化执行顺序**是这样的：

LinkedHashSet被反序列化-> 其中包含的TemplatesImpl对象和被代理的Templates对象分别被反序列化，AnnotationInvocationHandler也被反序列化并初始化基本信息-> Templates类型代理对象被反序列化后，加入LinkedHashSet内部的LinkedHashMap时会进行比较`key.eqauls(k)`，其中key为代理对象、k为含payload的TemplatesImpl对象-> 该equals方法也交由AnnotationInvocationHandler#equalsImpl进行实现，中间会执行Templates.getOutputProperties().invoke(TemplatesImpl)，最终rce

## 语法相关

### 引用类型的传递

`String`和`String[]`都是引用类型 存储对象引用，但`String`类型的对象是不可变的，当它们作为参数传递时会有不同的表现

```java
public class main {
    public static void main(String[] args) {
        Person p = new Person();
        String[] fullname = new String[] { "Homer", "Simpson" };
        p.setName(fullname);
        System.out.println(p.getName()); // "Homer Simpson"
        fullname[0] = "Bart"; // 修改String[]
        System.out.println(p.getName()); // "Bart Simpson"
    }

}

class Person {
    private String[] name;

    public String getName() {
        return this.name[0] + " " + this.name[1];
    }

    public void setName(String[] name) {
        this.name = name;
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        Person p = new Person();
        String bob = "Bob";
        p.setName(bob);
        System.out.println(p.getName()); // "Bob"
        bob = "Alice"; // 修改String
        System.out.println(p.getName()); // "Bob"
    }
}

class Person {
    private String name;

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

在例2中对`String`对象bob的修改相当于创建一个新的对象，所以当再次`getName`时会返回修改前的旧值；而`String[]`类型的对象是可变的，当它作为参数传递给方法时 实际传递的是该变量所引用的数组对象在内存中的地址（指针），当我们修改它时就会影响到使用它的地方

基本类型的参数传递不用考虑这一情况，所有修改不会有后续影响

### 构造方法

构造方法没有返回值，调用它的方式就是用`new`关键字；默认的构造方法为`(){}`，如果存在自定义构造方法将不再创建默认的构造方法，需要手动写出

构造方法可以参数、返回值均为空，但写其它的方法一般不可以 定义方法时必须存在语句，如果定义了方法 又没有具体代码 那一定是抽象方法，所在类无法被实例化

一个构造方法中可以调用其它构造方法，语法是`this(...)`

任何类构造方法的第一行语句都是调用父类的构造方法`super()`，所以在类的继承中 如果父类没有默认的构造方法，子类必须显式调用`super`并传入适当的参数调用父类自定义的构造方法；未指定的新建类都默认继承自Object，所以无此困扰

### 静态xx

类中的静态字段由所有实例共享

类的静态方法不需要通过实例即可调用，通过类名即可调用，它属于类而不属于实例 因此也无法访问`this`变量 只可以访问静态字段（入口的`main`函数就是静态方法！

类中可以嵌套静态内部类`static class Inner{}`，有访问类中`private`字段/方法的权限，但因为是完全独立的类 也无法引用`Outer.this`

类中可以有静态代码块`static{}`，在类的实例化（调用构造函数）或类的初始化（反射`forName`）时均会被调用