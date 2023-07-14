---
title: "Java学习笔记ⅩⅢ"
slug: "java-study-notes-13"
description: "基础拾遗，查漏补缺"
date: 2023-07-14T23:31:58+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

太久不看java，已经忘得一干二净 手感全无了，最近重新拾起来复建发现了一些从未发现的盲点或一些当时没有彻底理解透彻的地方，遂整理一篇；内容相当基础，新手友好

----

## 关于TransformedMap

*LazyMap也可以用类似的`LazyMap.decorate`

```java
Transformer[] transformers = new Transformer[]{
       new ConstantTransformer(Runtime.class),
       new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
       new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
       new InvokerTransformer("exec", new Class[]{String.class}, new String[]{"calc.exe"})
};
Transformer[] fake = new Transformer[]{new ConstantTransformer(1)};
Transformer transformerChain = new ChainedTransformer(fake);
Map m1 = new HashMap();
Map m2 = TransformedMap.decorate(m1, null, transformerChain);
```

`TransformedMap.decorate`是对`Map`对象的修饰，可传入`keyTransformer`和`valueTransformer`两个transformer对Map的key和value进行处理，调用transform

这里的transformer都是实现Transformer接口（实现transform方法）的类，常见的有

- `ConstantTransformer`：返回transform传入的参数本身
- `InvokerTransformer`：可传入待执行的方法名、参数类型列表、参数列表，transform方法中对传入参数进行反射调用（rce的关键）
- `ChainedTransformer`：组合多个transformer形成链式调用，最前的transform执行结果作为下一个transformer的参数

## CC1的LazyMap

- **Map版CC1**：AnnotationInvocationHandler#readObject-> memberValue.setValue(Map)-> rce
- **LazyMap版CC1**：AnnotationInvocationHandler#readObject-> memberValue.setValue(Map)-> AnnotationInvocationHandler#invoke(proxy)-> LazyMap#get-> rce

这里反驳一下p牛在java安全漫谈11中的这两段话

![image-20230713235555818](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230713235555818.png)

![image-20230713235613928](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230713235613928.png)

1. 被代理的对象是`Map` 并不是`AnnotationInvocationHandler`

最直观的判别方法：被代理的对象是谁，`Proxy.newProxyInstance`就会被强制转型成谁，传入的参数也会是与被代理对象强相关，这里是

```java
Map proxyMap = (Map)Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class},handler)
```

2. LazyMap的触发与setValue有关

Map被代理后需要有调用Map.xxx的地方才会触发`AnnotationInvocationHandler#invoke`，调用Map.xxx的地方就在`AnnotationInvocationHandler#readObject`中，也就是Map版CC1的触发点

## 语法基础

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