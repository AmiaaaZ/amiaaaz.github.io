---
title: "Java学习笔记ⅩⅢ"
slug: "java-study-notes-13"
description: "基础拾遗，查漏补缺"
date: 2023-07-13T23:34:58+08:00
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
