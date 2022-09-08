---
title: "Java学习笔记Ⅸ"
slug: "java-study-notes-09-design-patterns"
description: "语法基础补全之关于设计模式的小结"
date: 2022-09-04T14:44:57+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

学了小半年java了，审复杂一点的代码还是会被无情的java语法铁拳暴击，经常会被各种impl，interface给绕晕，一通debug下来属于是似是而非的懂了，但没有完全懂，所以小结一下关于设计模式相关的内容

*纯笔记梳理向，无原创内容，主要来自于廖大的博客[设计模式](https://www.liaoxuefeng.com/wiki/1252599548343744/1264742167474528)，侵删

## 创建型模式

不使用这种设计模式时我们通常使用`new`直接调用类的构造方法来创建实例

```python
# python
class Person{
}
Person p1 = new Person();
```

```go
// go
type Person struct{
}
var p1 Person
p1 = new(Person)
```

如果是简单的项目直接`new`也没什么大问题，但是当出现很多执行流程明确但流程细节有小差异的业务时，我们希望把它们抽象出来，暴露一个接口供外界使用，根据我们传入的参数来动态创建实例或调用方法，做到易读易用

### 工厂方法Factory Method

工厂方法Factory Method是一种对象创建型模式，目的是使得创建对象和使用对象是分离的，并且客户端总是引用抽象工厂和抽象产品

举例：我们希望实现一个解析字符串`Number`的工厂，先抽象接口

```java
public interface NumberFactory {
    Number parse(String s);
}
```

再写实现类

```java
public class NumberFactoryImpl implements NumberFactory{
	public Number parse(String s){
		return new BigDecimal(s);
	}
}
```

接口是`NumberFactory`，实现类是`NumberFactoryImpl`，返回的实际产品是`BigDecimal`

在工厂模式下，我们通常会在`NumberFactory`中再定义一个静态方法来返回真正的子类用于客户端调用来创建实例

```java
public interface NumberFactory {
    Number parse(String s); // 方法
    static NumberFactory getFactory(){  // 获取工厂实例(实现类impl)
        return impl;
    }
    static NumberFactory impl = new NumberFactoryImpl();    // 创建实例 不需要客户手动new NumberFactoryImpl(); 让用户对该实现类无感
}
```

此时客户端再调用时只需要与接口`NumberFactory`和抽象产品`Number`打交道

```java
NumberFactroy factory = NumberFactroy.getFactory;
Number result = factory.parse("123.123");
```

直接忽略了中间的`NumberFactoryImpl`实现类和实际产品`BigDecimal`

#### 静态工厂方法

上面的方法虽然很香，但复杂度还是非常的劝退，如果业务没那么复杂时可以直接简化为静态工厂方法——取消接口，通过静态方法直接返回产品

```java
public class NumberFactory{
	public static Number parse(String s){
		return new BigDecimal(s);
	}
}
```

客户端调用时可以省略`getFactory`这一步

```java
Number result = NumberFactory.parse("123.123");
```

在这种方法中，`NumberFactory`既是产品又是静态工厂，提供了静态方法`parse`来创建`Number`；`Integer`就是这样实现的

```java
Integer n = Integer.valueOf(100);
```

比直接`new Integer(100)`好的地方在于`valueOf`内部可能会使用`new`创建一个新的`Integer`实例，但也可能直接返回一个缓存的`Integer`实例，对于调用方来说，没必要知道`Integer`创建的细节；如果调用方直接使用`Integer n = new Integer(100)`，那么就失去了使用缓存优化的可能性

另一个常用的静态工厂方法是`MessageDigest`计算摘要

```java
MessageDigest md5 = MessageDigest.getInstance("MD5");
MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
```

调用方通过产品名称获得产品实例，不但调用简单，而且获得的引用仍然是`MessageDigest`这个抽象类

### 抽象工厂Abstract Factory

抽象工厂模式Abstract Factory是一个比较复杂的创建型模式，会出现抽象工厂对应多个实际工厂、每个实际工厂创建多个实际产品的情况

![image-20220902110636017](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220902110636017.png)

举例：为用户提供一个md转HTML或Word的服务，定义如下接口

```java
public interface AbstractFactory {
    HtmlDocument createHtml(String md);
    WordDocument createWord(String md);
}
```

此时不着急写实现类，因为格式转换太复杂了，~~小伙伴对不起产能不足~~，所以我们再写两个接口留着供应商实现

```java
public interface HtmlDocument{
	String toHtml();
	void save(Path path) throws IOException;
}

public interface WordDocument{
	void save(Path path) throws IOException;
}
```

此时`AbstractFactory`是抽象工厂，`HtmlDocument`和`WordDocument`是留给供应商实现的两个抽象产品

当前有两个供应商：FastDoc便宜且速度快，GoodDoc贵但效果好，我们计划同时使用两家供应商以便给免费和付费用户提供不同的服务

首先来看FastDoc的具体实现，它必须要有实际的产品，即`FastHtmlDocument`和`FastWordDocument`（对应接口的实现类）

```java
public class FastHtmlDocument implements HtmlDocument{
	public String toHtml(){
	}
	public void save(Path path) throws IOException{
	}
}

public class FastWordDocument implements WordDocument{
	public void save(Path path) throws IOException{
	}
}
```

还需要实际的工厂来生产这两种产品，既`FastFactory`

```java
public class FastFactory implements AbstractFactory{
	public HtmlDocument createHtml(String md){
		return new FastHtmlDocument(md);
	}
	public WordDocument createWord(String md){
		return new FastWordDocument(md);
	}
}
```

当我们作为客户端来使用FastDoc的服务时，需要这样编写代码

```java
AbstractFactory factory = new FastFactory();
HtmlDocument html = factory.createHtml("## hello");
html.save(Paths.get(".", "fast.html"));
WordDocument word = factroy.createWord("## hello");
word.save(Paths.get(".", "fast.doc"));
```

而对于GoodDoc，也是相同的操作（和FastDoc一样）

```java
// 工厂
public class GoodFactory implements AbstractFactory{
	public HtmlDocument createHtml(String md){
		return new GoodHtmlDocument(md);
	}
	public WordDocument createWord(String md){
		return new GoodWordDocument(md);
	}
}

// 产品
public class GoodHtmlDocument implements HtmlDocument{
	public String toHtml(){
	}
	public void save(Path path) throws IOException{
	}
}
public class GoodWordDocument implements WordDocument{
	public void save(Path path) throws IOException{
	}
}
```

客户端想要使用GoodDoc只需要把`new FastFactory`变为`new GoodFactory`即可

```java
AbstractFactory factory = new GoodFactory();
HtmlDocument html = factory.createHtml("## hello");
html.save(Paths.get(".", "fast.html"));
WordDocument word = factroy.createWord("## hello");
word.save(Paths.get(".", "fast.doc"));
```

对于客户端而言，只需要`new`一个工厂（`FastFactory`或`GoodFactory`），剩下的只需要知道产品接口而不需要知道实际产品（比如中间的`FastHtmlDocument`），如果把创建工厂的代码也放到`AbstractFactory`就可以连实际工厂也屏蔽了

```java
public interface AbstractFactory{
	public static AbstractFactory createFactory(String name){
		if(name.equalsIgnoreCase("fast")){
			return new FastFactory();
		}else if(name.equalsIgnoreCase("good")){
			return new GoodFactory();
		}else{
			throw new IllegalArgumentException("Invalid factory name");
		}
	}
}
```

### 生成器Builder

生成器Builder是使用多个小型工厂来最终创建一个完整对象，当我们使用Builder时一般是因为创建这个对象的步骤比较多，每个步骤都需要一个零部件，最终组合成一个完整的对象

举例：md文本转html，直接写一个完整的转换器比较困难，但如果针对单一类型的文本就很好做了

- `#`开头：`HeadingBuilder`转换
- `>`开头：`QuoteBuilder`转换
- `---`开头：`HrBuilder`转换
- 其余使用`ParagraphBuilder`转换

我们先写一个总的`HtmlBuilder`把转换的任务拆分

```java
public class HtmlBuilder{
	private HeadingBuilder headingBuilder = new HeadingBuilder();
	private HrBuilder hrBuilder = new HrBuilder();
	private QuoteBuilder quoteBuilder = new QuoteBuilder();
	private ParagraphBuilder = new ParagraphBuilder();

	public String toHtml(String markdown){
		StringBuilder buffer = new StringBuilder();
		markdown.lines().forEach(line -> {
			if(line.startsWith("#")){
				buffer.append(headingBuilder.buildHeading(line)).append("\n");
			}else if(line.startsWith(">")){
				buffer.append(quoteBuilder.buildQuote(line)).append("\n");
			}else if(line.startsWith("---")){
				buffer.append(hrBuilder.buildHr(line)).append("\n");
			}else{
				buffer.append(paragraphBuilder.buildParagraph(line)).append("\n");
			}
		});
		return buffer.toString();
	}
}
```

之后分别编写对于功能的Builder即可，比如`HeadingBuilder`

```java
public class HeadingBuilder {
    public String buildHeading(String line) {
        int n = 0;
        while (line.charAt(0) == '#') {
            n++;
            line = line.substring(1);
        }
        return String.format("<h%d>%s</h%d>", n, line.strip(), n);
    }
}
```

### 原型Prototype

原型Prototype，是指创建新对象的时候，根据现有的一个原型来创建（javascript中全是原型）

举例：已有一个`String[]`数组，想创建一个一模一样的`String[]`数组，我们会将现有数组的元素复制到新数组，这个过程封装起来就变成了原型模式

对于普通类想要实现原型拷贝，可以实现`Cloneable`接口来标识一个对象是可复制的

```java
public class Student implements Cloneable{
	private int id;
	private String name;
	private int score;

	public Object clone(){
		Student std = new Student();
		std id = this.id;
		std.name = this.name;
		std.score = this.score;
		return std;
	}
}
```

使用的时因为`clone()`的方法签名是定义在`Object`中，返回类型也是`Object`，所以要强制转型，比较麻烦：

```java
Student std1 = new Student();
std1.setId(123);
std1.setName("Bob");
std1.setScore(88);
// 复制新对象:
Student std2 = (Student) std1.clone();
System.out.println(std1);
System.out.println(std2);
System.out.println(std1 == std2); // false
```

使用原型模式更好的方式是定义一个`copy()`方法，返回明确的类型：

```java
public class Student {
    private int id;
    private String name;
    private int score;

    public Student copy() {
        Student std = new Student();
        std.id = this.id;
        std.name = this.name;
        std.score = this.score;
        return std;
    }
}
```

原型模式应用不是很广泛，因为很多实例会持有类似文件、Socket这样的资源，而这些资源是无法复制给另一个对象共享的，只有存储简单类型的“值”对象可以复制

### 单例Singleton

单例Singleton是为了保证在一个进程中，某一个类有且仅有一个实例

*应用场景：很多程序（尤其是web程序），大部分服务类都应该被视作Singleton，如果全部按照Singleton的写法写，会非常麻烦，所以通常是通过约定让框架（如Spring）来实例化这些类，保证只有一个实例，调用方自觉通过框架获取实例而不是`new`操作符；因此除非确有必要，否则Singleton模式一般以约定为主，不会刻意实现它

因为这个类只有一个实例，因此不能让调用方通过`new`来创建实例了，这样无法做到控制仅有1个实例，所以单例的构造方法必须是`private`，这样防止了调用方自己创建实例，但是在类的内部是可以用一个静态字段来引用唯一创建的实例的

```java
public class Singleton{
	// 静态字段引用唯一实例
	private static final Singleton INSTANCE = new Singleton();

	// private构造方法保证外部无法实例化
	private Singleton(){
	}
}
```

外部调用方通过静态方法返回实例

```java
public class Singleton {
    // 静态字段引用唯一实例
    private static final Singleton INSTANCE = new Singleton();

    // 通过静态方法返回实例
    public static Singleton getInstance() {
        return INSTANCE;
    }

    // private构造方法保证外部无法实例化
    private Singleton() {
    }
}
```

或者直接把static变量暴露在外部

```java
public class Singleton {
    // 静态字段引用唯一实例 改为public
    public static final Singleton INSTANCE = new Singleton();

    // private构造方法保证外部无法实例化
    private Singleton() {
    }
}
```

所以单例模式的实现方法很简单：

- 只有`private`构造方法，确保外部无法实例化
- 通过`private static`变量持有唯一实例，保证全局唯一性（构造静态方法返回实例）
- 通过`public static`变量返回此唯一实例，使外部调用方能获取到实例

Java标准库中的`Runtime`就是单例（静态方法返回实例）

```java
Runtime runtime = Runtime.getRuntime();
```

#### 延迟加载

即调用方第一次调用`getInstance`时才初始化全局唯一实例

```java
public class Singleton{
	private static Singleton INSTANCE = null;
	public static Singleton getInstance(){
		if(INSTANCE == null){
			INSTANCE = new Singleton();
		}
		return INSTANCE;
	}

	private Singleton(){
	}
}
```

但是这样的写法在多线程中是错误的，在竞争条件下会创建出多个实例，必须对整个方法进行加锁

```java
public synchronized static Singleton getInstance(){
	if(INSTANCE == null){
		INSTANCE = new Singleton();
	}
	return INSTANCE;
}
```

但是加锁会影响并发性能

还有类似这样的双重检查：

```java
public static Singleton getInstance(){
	if(INSTANCE == null){
		synchronized(Singleton.class){
			if(INSTANCE == null){
				INSTANCE = new Singleton();
			}
		}
	}
	return INSTANCE;
}
```

由于java的内存模型，双重检查在这里也不存在；要实现真正的延迟加载只能通过java的ClassLoader来完成，如果没有特殊的需求，使用Singleton模式的时候，最好不要延迟加载，这样会使代码更简单

#### 枚举

另一种实现Singleton的方法是利用java的`enum`枚举类，因为java保证枚举类的每个枚举都是单例，所以我们只需要编写一个只有一个枚举的类即可

```java
public enum World{
	// 唯一枚举
	INSTANCE;
	private String name = "word";

	public String getName(){
		return this.name;
	}

	public void setName(String name){
		this.name = name;
	}
}
```

枚举类也可以像其它类那样定义自己的字段、方法，比如我们可以这样调用`World`

```java
String name = World.INSTANCE.getName();
```

使用枚举实现Singleton还避免了第一种方式实现Singleton的一个潜在问题，即序列化和反序列化会绕过普通类的`private`构造方法从而创建出多个实例，而枚举类就没有这个问题

## 结构型模式

结构型模式不仅仅简单地使用继承，而更多地通过组合与运行期的动态组合来实现更灵活的功能

### 适配器Adapter/Wrapper

近似理解为充电器转接口，将A接口转换为B接口；或者可以理解为接口的强制类型转换（近似）

举例：已有一个实现了`Callable`接口的`Task`类

```java
public class Task implements Callable<Long> {
    private long num;
    public Task(long num) {
        this.num = num;
    }

    public Long call() throws Exception {
        long r = 0;
        for (long n = 1; n <= this.num; n++) {
            r = r + n;
        }
        System.out.println("Result: " + r);
        return r;
    }
}
```

现在想通过一个线程来执行

```java
Callable<Long> callable = new Task(123450000L);
Thread thread = new Thread(callable);
thread.start();
```

但是无法编译通过，因为`Thread`接收`Runnable`接口 不接收`Callable`接口

一个办法是将`Task`类改为继承`Runnable`，但可能会影响其它代码的正常运行；另一个办法是使用Adapter，把这个`Callable`接口变为`Runnable`接口

```java
public class RunnableAdapter implements Runnable{	// 实现目标接口
	// 引用待转换接口
	private Callable<?> callable;

	public RunnableAdapter(Callable<?> callbale){
		this.callable = callable;
	}
	// 实现指定接口
	public void run(){
		// 实现被转换接口的方法
		try{
			callable.call();
		}catch(Exception e){
			throw new RuntimeExceptin(e);
		}
	}
}
```

```java
Callable<Long> callable = new Task(123450000L);
Thread thread = new Thread(new RunnableAdapter(callable));
thread.start();
```

这样一来`Thread`接收的对象就是实现了`Runnable`接口的`RunnableAdapter`，调用它的`run`方法，在这个`run`内部实际调用了`callable`的`call`方法

在java中广泛使用适配器模式，比如我们持有的数据类型是`String[]`，但需要`List`接口

```java
String[] exist = new String[]{"Good", "morning", "Bob", "and", "Alice"};
Set<String> set = new HashSet<>(Arrays.asList(exist));
```

这里的`Arrays.asList`就是一个转换器，将数组转换为List

例子2：我们持有一个`InputStream`，希望调用`readText(Reader)`方法，但它的参数类型是`Reader`不是`InputStream`，我们可以使用适配器把`InputStream`变为`Reader`

```java
InputStream input = Files.newInputStream(Paths.get("/path/to/file"));
Reader reader = new InputStreamReader(input, "UTF-8");
readText(reader);
```

这里的`InputStreamReader`就是java标准库的Adapter，负责把一个`InputStream`适配为`Reader`，类似的还有`OutputStreamWriter`

### 桥接Bridge

举例：某汽车厂商生产3种品牌的汽车：Big, Tiny, Boss，每个品牌各有纯油、纯电、混动，如果用传统的继承来表示各个车型，需要3个抽象类+9个实现子类

![image-20220902231807074](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220902231807074.png)

如果要新增一个品牌或新的动力类型，那么子类会爆炸增长，桥接模式就是为了解决继承有可能带来的子类爆炸问题

桥接模式中，首先将`Car`按品牌进行子类化，但是每个品牌的动力类型不再使用子类扩充，而是通过一个抽象的“修正”类，以组合的形式引入

首先定义抽象类`Car`，引入`Engine`

```java
public abstract class Car{
	protected Engine engine;

	public Car(Engine engine){
		this.engine = engine;
	}

	public abstract void drive();
}
```

`Engine`的定义如下

```java
public interface Engine{
	void start();
}
```

紧接着，在一个“修正”的抽象类`RefinedCar`中定义一些额外操作

```java
public abstract class RefinedCar extends Car{
	public RefinedCar(Engine engine){
		super(engine);
	}

	public void drive(){
		this.engine.start();
		System.out.println("Drvie " + getBrand() + " car...");
	}

	public abstract String getBrand();
}
```

这样一来，最终的不同品牌的车型继承自`RefinedCar`，比如

```java
public class BossCar extends RefinedCar{
	public BossCar(Engine engine){
		super(engine);
	}
	public String getBrand(){
		return "Boss";
	}
}
```

而动力类型继承自`Engine`，比如

```java
public class HybridEngine implements Engine{
	public void start(){
		System.out.println("Start Hybrid Engine...");
	}
}
```

而最终的“车”，可以组合得到

```java
RefinedCar car = new BossCar(new HybridEngine());
car.drive();
```

显而易见，桥接模式的好处在于能更自由的对想要的子类根据不同的需求进行一个排列组合

![image-20220902233406811](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220902233406811.png)

廖大的这个图还是很清楚的，我们的两个需求分别是车型和动力类型，各有各的需求，通过桥接的模式组装在一起，而不是一味的继承继承继承、

### 组合Composite

Composite模式使得叶子对象和容器对象具有一致性，从而形成统一的树形结构，并用一致的方式去处理它们

举例：XML和HTML就使用了树形结构，根节点下有嵌套的子节点

要以树形结构表示XML，我们可以先抽象出节点类型`Node`

```java
public interface Node{
	// 可以添加一个节点为子节点
	Node add(Node node);
	// 获取子节点
	List<Node> children();
	// 输出XML
	String toXML();
}
```

对于`<abc>`这样的ElementNode，可以作为容器包含多个子节点

```java
public class ElementNode implements Node{
	private String name;
	private List<Node> list = new ArrayList<>();

	public ElementNode(String name){
		this.name = name;
	}

	public Node add(Node node){
		list.add(node);
		return list;
	}

	public List<Node> children(){
		return list;
	}

	public String toXML(){
		String start = "<" + name + ">\n";
		String end = "</" + name + ">\n";
        StringJoiner sj = new StringJoiner("", start, end);
        list.forEach(node -> {
            sj.add(node.toXml() + "\n");
        });
        return sj.toString();
	}
}
```

对于普通文本，我们视作TextNode，没有子节点

```java
public class TextNode implements Node{
	private String text;
	public TextNode(String text){
		this.text = text;
	}

	public Node add(Node node) {
		throw new UnsupportedOperationException();	// 无法增加子节点
	}

	public List<Node> children() {
		return List.of();
	}

	public String toXml() {
		return text;
	}
}
```

还可以有注释节点：

```java
public class CommentNode implements Node {
	private String text;

	public CommentNode(String text) {
		this.text = text;
	}

	public Node add(Node node) {
		throw new UnsupportedOperationException();
	}

	public List<Node> children() {
		return List.of();
	}

	public String toXml() {
		return "<!-- " + text + " -->";
	}
}
```

通过ElementNode, TextNode, CommentNode我们就可以构造出一棵树

```java
Node root = new ElementNode("school");
root.add(new ElementNode("classA")
        .add(new TextNode("Tom"))
        .add(new TextNode("Alice")));
root.add(new ElementNode("classB")
        .add(new TextNode("Bob"))
        .add(new TextNode("Grace"))
        .add(new CommentNode("comment...")));
System.out.println(root.toXml());
```

综上，组合模式需要先统一单个节点以及容器节点的接口

![image-20220903000328506](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903000328506.png)

作为容器节点的ElementNode又可以添加任意个Node，这样就可以构成层级结构

类似的，像文件夹和文件、GUI窗口的各种组件，都符合Composite模式的定义，因为它们的结构天生就是层级结构

### 装饰器Decorator

Decorator是一种在运行期动态给某个对象的实例增加功能的方法，在各个web应用里很常见，Decorator模式的目的就是把一个一个的附加功能，用Decorator的方式给一层一层地累加到原始数据源上，最终，通过组合获得我们想要的功能

举例：给`FileInputStream`增加缓冲和解压缩功能，用装饰器模式是这样的

```java
InputStream fis = new FileInputStream("test.gz");
InputStream bis = new BufferedInputStream(fis);
InputStream gis = new GZIPInputStream(bis);
```

或一次性写

```java
InputStream input = new GZIPInputStream(new BufferedInputStream(new FileInputStream("test".gz)))
```

这里的`BufferedInputStream`和`GZIPInputStream`实际都是从`FilterInputStream`继承而来的，这个`FilterInputStream`就是一个抽象的Decorator

![image-20220903101312952](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903101312952.png)

最顶层的Component是接口，对应到IO就是`InputStream`这个抽象类，下面的AB...是实际的子类，就是`FileInputStream`和`ServletInputStream`这些数据源；Decorator对应到IO就是`FilterInputStream`，下面的AB...就是`BufferedInputStream`和`GZIPInputStream`

与桥接模式的区别在于装饰器模式下把核心功能和附加功能分开了，而不是桥接中的二者并行，虽然二者都是为了避免继承中可能出现的子类爆炸，但装饰器是对已经有的功能做修饰（最终产品可以不被修饰，也可以不断被嵌套），桥接是把并行的多个功能进行排列组合；比如这里的数据源和解压缩功能，如果我们要增加核心功能就增加Component的子类（比如`ByteInputStream`），如果要增加附加功能，就增加Decorator的子类（比如`CipherInputStream`），同时Decorator是继承自Component的；两部分可以独立扩展，具体如何附加功能由调用方自由组合，增强了灵活性

举例：要渲染HTML文本，但是文本可以附加一些比如加粗、斜体之类的效果，我们可以使用装饰器模式

首先还是之前组合模式下的，先定义顶层接口TextNode

```java
public interface TextNode{
	void setText(String text);
	void getText();
}
```

对于核心节点，比如`<span>`，直接继承

```java
public class SpanNode implements TextNode{
	private String text;
	void setText(String text){
		this.text = text;
	}
	void getText(){
		return "<span>" + text + "</span>";
	}
}
```

装饰器的部分，需要先有一个抽象的Decorator类（仍然继承自TextNode）

```java
public abstract class NodeDecorator implements TextNode{
	protected final TextNode target;
	protected NodeDecorator(TextNode target){
		this.target = target;
	}
	public void setText(String text){
		this.target.setText(text);
	}
}
```

这个`NodeDecorator`的核心是持有一个`TextNode`，功能都会附加到这个`TextNode`上；下面我们写一个加粗功能（继承自抽象装饰器）

```java
public class BoldDecorator extends NodeDecorator{
	public BoldDecorator(TextNode target){
		super(target);
	}
	public String getText(){
		return "<b>" + target.getText() + "</b>";
	}
}
```

类似的，可以继续添加`ItalicDecorator`, `UnderlineDecorator`等

```java
TextNode n1 = new SpanNode();
TextNode n2 = new BoldDecorator(new UnderlineDecorator(new SpanNode()));
TextNode n3 = new ItalicDecorator(new BoldDecorator(new SpanNode()));

n1.setText("Hello");
n2.setText("Decorated");
n3.setText("World");
System.out.println(n1.getText());
// <span>Hello</span>

System.out.println(n2.getText());
// <b><u><span>Decorated</span></u></b>

System.out.println(n3.getText());
// <i><b><span>World</span></b></i>
```

### 门面/外观Facade

终于到了经典的门面模式了（tomcat里真的到处都是门面模式），它的设计思想类似房屋中介

举例：假设注册公司有三步：

1. 向工商局申请公司营业执照
2. 在银行开设账户
3. 在税务局开设纳税号

以下是三个系统的接口

```java
// 工商注册:
public class AdminOfIndustry {
    public Company register(String name) {
        ...
    }
}

// 银行开户:
public class Bank {
    public String openAccount(String companyId) {
        ...
    }
}

// 纳税登记:
public class Taxation {
    public String applyTaxCode(String companyId) {
        ...
    }
}
```

如果子系统比较复杂，并且客户对流程也不熟悉，那就把这些复杂流程全部委托给中介Facade

```java
public class Facade {
    public Company openCompany(String name) {
        Company c = this.admin.register(name);
        String bankAccount = this.bank.openAccount(c.getId());
        c.setBankAccount(bankAccount);
        String taxCode = this.taxation.applyTaxCode(c.getId());
        c.setTaxCode(taxCode);
        return c;
    }
}
```

这样客户端只跟Facade打交道即可

```java
Company c = facade.openCompany("Facade Software Ltd.");
```

在Web程序中随处可见Facade，内部有多个子系统提供服务，经常使用一个统一的Facade入口，比如`RestApiController`，使得外部用户调用的时候，只关心Facade提供的接口，不管内部到底是哪个子系统处理的；复杂的web应用中经常有一个Gateway网关入口来转发不同的web服务或附加用户认证功能，本质也是一个Facade

### 享元Flyweight

Flyweight的核心思想很简单：如果一个对象实例已经创建就不可变，那么反复创建一样的实例就没必要，直接返回一个共享的实例即可，减少创建对象过程的同时可以减少节省内存，提高运行速度，实现了不可变实例的复用

与单例的区别在于不可变

在java标准库中有很多应用，比如包装类型`Byte`和`Integer`都是不变类，因此反复创建同一个值相同的包装类型就没必要，以`Integer`为例，用`valueOf`这个静态工厂方法创建的实例（当传入int在(-128~+127)之间时）会直接返回缓存的`Integer`实例

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Integer n1 = Integer.valueOf(100);
        Integer n2 = Integer.valueOf(100);
        System.out.println(n1 == n2); // true
    }
}
```

对于`Byte`来说，因为一共只有256个状态，所以通过`Byte.valueOf()`创建的`Byte`实例都是缓存对象

在实际应用中，享元模式主要应用于缓存，即客户端如果重复请求某些对象，不必每次查询数据库或者读取文件，而是直接返回内存中缓存的数据（不用每次都new new new）

举例：

```java
public class Student{
	// 持有缓存
	private static final Map<String, Student> cache = new HashMap<>();

	// 静态工厂方法
	public static Student create(int id, String name){
		String key = id + "\n" + name;
		// 查找缓存
		Student std = cache.get(key);
		if(std == null){
			// 未找到创建新对象
			System.out.println(String.formaet("create new Student (%s, %s)", id, name));
			std = new Student(id, name);
			// 放入缓存
			cache.put(key, std);
		}else{
			// 缓存中存在
			System.out.println(String.format("return cached Student (%s, %s)", std.id, std.name));
		}
		return std;
	}

	private final int id;
	private final String name;

	public Student(int id, String name){
		this.id = id;
		this.name = name;
	}
}
```

### 代理Proxy

与适配器Adpater很相似，但不是将A接口转为B接口，它还是到A接口，只不过在包装的时候可以多加一些额外的代码

举例：有A接口 其中实现了a方法，而AProxy类继承自A，其中的a方法可以增加鉴权，如果想要其它类型的代理只需要继续继承A即可

#### 远程代理Remote Proxy

比如我们经典的RMI(Remote Method Invocation)就是远程代理，本地调用者持有的接口实际是一个代理，这个代理负责把对接口的方法转换为远程调用

#### 虚代理Virtual Proxy

让调用者先持有一个代理对象，但真正的对象尚未创建，如果没有必要这个真正的对象是不会被创建的，直到真正调用时才会创建，比如JDBC的连接池返回的JDBC连接（Connection对象）就是一个虚代理，即获取连接时根本没连上，直到第一次查询或更新操作时才会创建真正的JDBC连接

#### 保护代理Protection Proxy

用代理对象控制对原始对象的访问，常用于鉴权

#### 智能引用Smart Reference

也是一种代理对象，如果有很多客户端对它进行访问，通过内部的计数器可以在外部调用者都不使用后自动释放它

举例：用代理模式编写一个JDBC连接池（DataSource）

首先写一个虚代理，即如果调用者获取到`Connection`后，并没有执行任何SQL操作，那么这个Connection Proxy实际上并不会真正打开JDBC连接，调用者代码：

```java
DataSource LazyDataSource = new LazyDataSource(jdbcUrl, jdbcUsername, jdbcPassword);
System.out.println("get lazy connection...");
try(Connection conn1 = lazyDataSource.getConnection()){
	// 没有打开连接
}
System.out.println("get lazy connection...");
try(Connection conn2 = lazyDataSource.getConnection()){
	try (PreparedStatement ps = conn2.prepareStatement("SELECT * FROM students")) { // 打开了真正的Connection
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                System.out.println(rs.getString("name"));
            }
        }
    }
}
```

对于这个LazyConnectionProxy来说，为了简化代码，我们首先针对Connection接口做一个抽象的代理类

```java
public abstract class AbstractConnectionProxy implements Connection{
	// 抽象方法获取实际的Connection
    protected abstract Connection getRealConnection();
    // 实现Connection接口的每一个方法
    public Statement createStatement() throws SQLException{
        return getRealConnection().createStatement();
    }

    public PreparedStatement prepareStatement(String sql) throws SQLException{
        return getRealConnection().createStatement(sql);
    }
}
```

这个`AbstractConnectionProxy`代理类的作用是把`Connection`接口定义的方法全部实现一遍，因为`Connection`接口定义的方法太多了，后面我们要编写的`LazyConnectionProxy`只需要继承`AbstractConnectionProxy`，就不必再把`Connection`接口方法挨个实现一遍

```java
public class LazyConnectionProxy extends AbstractConnectionProxy{
    private Supplier<Connection> supplier;
    private Connection target = null

    public LazyConnectionProxy(Supplier<Connection> supplier){
        this.supplier = supplire;
    }

    // 覆写close方法：只有target不为null时才需要关闭
    public void clase() throws SQLException{
        if (target != null) {
            System.out.println("Close connection: " + target);
            super.close();
    }

    @Override
    protected Connection getRealConnection(){
        if (target == null) {
            target = supplier.get();
        }
        return target;
    }
}
```

如果调用者没有执行任何SQL语句，那么`target`字段始终为`null`。只有第一次执行SQL语句时（即调用任何类似`prepareStatement()`方法时，触发`getRealConnection()`调用），才会真正打开实际的JDBC Connection

最后，我们还需要编写一个`LazyDataSource`来支持这个`LazyConnectionProxy`：

```java
public class LazyDataSource implements DataSource {
    private String url;
    private String username;
    private String password;

    public LazyDataSource(String url, String username, String password) {
        this.url = url;
        this.username = username;
        this.password = password;
    }

    public Connection getConnection(String username, String password) throws SQLException {
        return new LazyConnectionProxy(() -> {
            try {
                Connection conn = DriverManager.getConnection(url, username, password);
                System.out.println("Open connection: " + conn);
                return conn;
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        });
    }
    ...
}
```

我们执行代码，输出如下：

```
get lazy connection...
get lazy connection...
Open connection: com.mysql.jdbc.JDBC4Connection@7a36aefa
小明
小红
小军
小白
...
Close connection: com.mysql.jdbc.JDBC4Connection@7a36aefa
```

可见第一个`getConnection()`调用获取到的`LazyConnectionProxy`并没有实际打开真正的JDBC Connection。

使用连接池的时候，我们更希望能重复使用连接。如果调用方编写这样的代码：

```java
DataSource pooledDataSource = new PooledDataSource(jdbcUrl, jdbcUsername, jdbcPassword);
try (Connection conn = pooledDataSource.getConnection()) {
}
try (Connection conn = pooledDataSource.getConnection()) {
    // 获取到的是同一个Connection
}
try (Connection conn = pooledDataSource.getConnection()) {
    // 获取到的是同一个Connection
}
```

调用方并不关心是否复用了`Connection`，但从`PooledDataSource`获取的`Connection`确实自带这个优化功能。如何实现可复用`Connection`的连接池？答案仍然是使用代理模式。

```java
public class PooledConnectionProxy extends AbstractConnectionProxy {
    // 实际的Connection:
    Connection target;
    // 空闲队列:
    Queue<PooledConnectionProxy> idleQueue;

    public PooledConnectionProxy(Queue<PooledConnectionProxy> idleQueue, Connection target) {
        this.idleQueue = idleQueue;
        this.target = target;
    }

    public void close() throws SQLException {
        System.out.println("Fake close and released to idle queue for future reuse: " + target);
        // 并没有调用实际Connection的close()方法,
        // 而是把自己放入空闲队列:
        idleQueue.offer(this);
    }

    protected Connection getRealConnection() {
        return target;
    }
}
```

复用连接的关键在于覆写`close()`方法，它并没有真正关闭底层JDBC连接，而是把自己放回一个空闲队列，以便下次使用。

空闲队列由`PooledDataSource`负责维护：

```java
public class PooledDataSource implements DataSource {
    private String url;
    private String username;
    private String password;

    // 维护一个空闲队列:
    private Queue<PooledConnectionProxy> idleQueue = new ArrayBlockingQueue<>(100);

    public PooledDataSource(String url, String username, String password) {
        this.url = url;
        this.username = username;
        this.password = password;
    }

    public Connection getConnection(String username, String password) throws SQLException {
        // 首先试图获取一个空闲连接:
        PooledConnectionProxy conn = idleQueue.poll();
        if (conn == null) {
            // 没有空闲连接时，打开一个新连接:
            conn = openNewConnection();
        } else {
            System.out.println("Return pooled connection: " + conn.target);
        }
        return conn;
    }

    private PooledConnectionProxy openNewConnection() throws SQLException {
        Connection conn = DriverManager.getConnection(url, username, password);
        System.out.println("Open new connection: " + conn);
        return new PooledConnectionProxy(idleQueue, conn);
    }
    ...
}
```

我们执行调用方代码，输出如下：

```
Open new connection: com.mysql.jdbc.JDBC4Connection@61ca2dfa
Fake close and released to idle queue for future reuse: com.mysql.jdbc.JDBC4Connection@61ca2dfa
Return pooled connection: com.mysql.jdbc.JDBC4Connection@61ca2dfa
Fake close and released to idle queue for future reuse: com.mysql.jdbc.JDBC4Connection@61ca2dfa
Return pooled connection: com.mysql.jdbc.JDBC4Connection@61ca2dfa
Fake close and released to idle queue for future reuse: com.mysql.jdbc.JDBC4Connection@61ca2dfa
```

除了第一次打开了一个真正的JDBC Connection，后续获取的`Connection`实际上是同一个JDBC Connection。但是，对于调用方来说，完全不需要知道底层做了哪些优化。

我们实际使用的DataSource，例如HikariCP，都是基于代理模式实现的，原理同上，但增加了更多的如动态伸缩的功能（一个连接空闲一段时间后自动关闭）

*与装饰器的区别：

Decorator模式让调用者自己创建核心类，然后组合各种功能，而Proxy模式决不能让调用者自己创建再组合，否则就失去了代理的功能。Proxy模式让调用者认为获取到的是核心类接口，但实际上是代理类。

## 行为型模式

行为型模式主要涉及算法和对象间的职责分配。通过使用对象组合，行为型模式可以描述一组对象应该如何协作来完成一个整体任务。

### 责任链Chain of Responsibility

是一种处理请求的模式，它让多个处理器都有机会处理该请求，直到其中某个处理成功为止。责任链模式把多个处理器串成链，然后让请求在链上传递（比如web中的Filter就是典型的责任链）

![image-20220903135616160](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903135616160.png)

举例：财务审批，假设某个员工需要报销一笔费用，审核者可以分为：

- Manager：只能审核1000元以下的报销；
- Director：只能审核10000元以下的报销；
- CEO：可以审核任意额度。

用责任链模式设计此报销流程时，每个审核者只关心自己责任范围内的请求，并且处理它。对于超出自己责任范围的，扔给下一个审核者处理，这样，将来继续添加审核者的时候，不用改动现有逻辑

首先，我们要抽象出请求对象，它将在责任链上传递：

```java
public class Request {
    private String name;
    private BigDecimal amount;

    public Request(String name, BigDecimal amount) {
        this.name = name;
        this.amount = amount;
    }

    public String getName() {
        return name;
    }

    public BigDecimal getAmount() {
        return amount;
    }
}
```

其次，我们要抽象出处理器：

```java
public interface Handler {
    // 返回Boolean.TRUE = 成功
    // 返回Boolean.FALSE = 拒绝
    // 返回null = 交下一个处理
	Boolean process(Request request);
}
```

并且做好约定：如果返回`Boolean.TRUE`，表示处理成功，如果返回`Boolean.FALSE`，表示处理失败（请求被拒绝），如果返回`null`，则交由下一个`Handler`处理。

然后，依次编写ManagerHandler、DirectorHandler和CEOHandler。以ManagerHandler为例：

```java
public class ManagerHandler implements Handler {
    public Boolean process(Request request) {
        // 如果超过1000元，处理不了，交下一个处理:
        if (request.getAmount().compareTo(BigDecimal.valueOf(1000)) > 0) {
            return null;
        }
        // 对Bob有偏见:
        return !request.getName().equalsIgnoreCase("bob");
    }
}
```

有了不同的`Handler`后，我们还要把这些`Handler`组合起来，变成一个链，并通过一个统一入口处理：

```java
public class HandlerChain {
    // 持有所有Handler:
    private List<Handler> handlers = new ArrayList<>();

    public void addHandler(Handler handler) {
        this.handlers.add(handler);
    }

    public boolean process(Request request) {
        // 依次调用每个Handler:
        for (Handler handler : handlers) {
            Boolean r = handler.process(request);
            if (r != null) {
                // 如果返回TRUE或FALSE，处理结束:
                System.out.println(request + " " + (r ? "Approved by " : "Denied by ") + handler.getClass().getSimpleName());
                return r;
            }
        }
        throw new RuntimeException("Could not handle request: " + request);
    }
}
```

现在，我们就可以在客户端组装出责任链，然后用责任链来处理请求：

```java
// 构造责任链:
HandlerChain chain = new HandlerChain();
chain.addHandler(new ManagerHandler());
chain.addHandler(new DirectorHandler());
chain.addHandler(new CEOHandler());
// 处理请求:
chain.process(new Request("Bob", new BigDecimal("123.45")));
chain.process(new Request("Alice", new BigDecimal("1234.56")));
chain.process(new Request("Bill", new BigDecimal("12345.67")));
chain.process(new Request("John", new BigDecimal("123456.78")));
```

责任链模式本身很容易理解，需要注意的是，`Handler`添加的顺序很重要，如果顺序不对，处理的结果可能就不是符合要求的。

此外，责任链模式有很多变种。有些责任链的实现方式是通过某个`Handler`手动调用下一个`Handler`来传递`Request`，例如：

```
public class AHandler implements Handler {
    private Handler next;
    public void process(Request request) {
        if (!canProcess(request)) {
            // 手动交给下一个Handler处理:
            next.process(request);
        } else {
            ...
        }
    }
}
```

还有一些责任链模式，每个`Handler`都有机会处理`Request`，通常这种责任链被称为拦截器（Interceptor）或者过滤器（Filter），它的目的不是找到某个`Handler`处理掉`Request`，而是每个`Handler`都做一些工作，比如：

- 记录日志；
- 检查权限；
- 准备相关资源；
- ...

例如，JavaEE的Servlet规范定义的`Filter`就是一种责任链模式，它不但允许每个`Filter`都有机会处理请求，还允许每个`Filter`决定是否将请求“放行”给下一个`Filter`：

```
public class AuditFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws IOException, ServletException {
        log(req);
        if (check(req)) {
            // 放行:
            chain.doFilter(req, resp);
        } else {
            // 拒绝:
            sendError(resp);
        }
    }
}
```

这种模式不但允许一个`Filter`自行决定处理`ServletRequest`和`ServletResponse`，还可以“伪造”`ServletRequest`和`ServletResponse`以便让下一个`Filter`处理，能实现非常复杂的功能。

### 命令Command

把请求封装成一个命令，然后执行该命令

在使用命令模式前，我们先以一个编辑器为例子，看看如何实现简单的编辑操作（用一个`StringBuilder`模拟一个文本编辑器，它支持`copy()`、`paste()`、`add()`、`delete()`等方法）

```java
public class TextEditor {
    private StringBuilder buffer = new StringBuilder();

    public void copy() {
        ...
    }

    public void paste() {
        String text = getFromClipBoard();
        add(text);
    }

    public void add(String s) {
        buffer.append(s);
    }

    public void delete() {
        if (buffer.length() > 0) {
            buffer.deleteCharAt(buffer.length() - 1);
        }
    }

    public String getState() {
        return buffer.toString();
    }
}
```

正常情况，我们像这样调用`TextEditor`：

```java
TextEditor editor = new TextEditor();
editor.add("Command pattern in text editor.\n");
editor.copy();
editor.paste();
System.out.println(editor.getState());
```

这是直接调用方法，调用方需要了解`TextEditor`的所有接口信息。

如果改用命令模式，我们就要把调用方发送命令和执行方执行命令分开。怎么分？

解决方案是引入一个`Command`接口：

```
public interface Command {
    void execute();
}
```

调用方创建一个对应的`Command`，然后执行，并不关心内部是如何具体执行的。

为了支持`CopyCommand`和`PasteCommand`这两个命令，我们从`Command`接口派生：

```java
public class CopyCommand implements Command {
    // 持有执行者对象:
    private TextEditor receiver;

    public CopyCommand(TextEditor receiver) {
        this.receiver = receiver;
    }

    public void execute() {
        receiver.copy();
    }
}

public class PasteCommand implements Command {
    private TextEditor receiver;

    public PasteCommand(TextEditor receiver) {
        this.receiver = receiver;
    }

    public void execute() {
        receiver.paste();
    }
}
```

最后我们把`Command`和`TextEditor`组装一下，客户端这么写：

```java
TextEditor editor = new TextEditor();
editor.add("Command pattern in text editor.\n");
// 执行一个CopyCommand:
Command copy = new CopyCommand(editor);
copy.execute();
editor.add("----\n");
// 执行一个PasteCommand:
Command paste = new PasteCommand(editor);
paste.execute();
System.out.println(editor.getState());
```

这就是命令模式的结构：

![image-20220903142302481](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903142302481.png)

虽然看起来比直接调用复杂的多，但如果TextEditor复杂到一定程度，并且需要支持Undo、Redo的功能时，就需要使用命令模式，因为我们可以给每个命令增加`undo()`：

```java
public interface Command {
    void execute();
    void undo();
}
```

然后把执行的一系列命令用`List`保存起来，就既能支持Undo，又能支持Redo。这个时候，我们又需要一个`Invoker`对象，负责执行命令并保存历史命令：

![image-20220903142424888](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903142424888.png)

### 解释器Interpreter

是一种针对特定问题设计的一种解决方案

举例：匹配字符串的时候，由于匹配条件非常灵活，使得通过代码来实现非常不灵活，针对以下的匹配条件：

- 以`+`开头的数字表示的区号和电话号码，如`+861012345678`；
- 以英文开头，后接英文和数字，并以.分隔的域名，如`www.liaoxuefeng.com`；
- 以`/`开头的文件路径，如`/path/to/file.txt`；
- ...

因此，需要一种通用的表示方法——正则表达式来进行匹配。正则表达式就是一个字符串，但要把正则表达式解析为语法树，然后再匹配指定的字符串，就需要一个解释器。

实现一个完整的正则表达式的解释器非常复杂，但是使用解释器模式却很简单：

```
String s = "+861012345678";
System.out.println(s.matches("^\\+\\d+$"));
```

类似的，当我们使用JDBC时，执行的SQL语句虽然是字符串，但最终需要数据库服务器的SQL解释器来把SQL“翻译”成数据库服务器能执行的代码，这个执行引擎也非常复杂，但对于使用者来说，仅仅需要写出SQL字符串即可。

### 迭代器Iterator

提供一种方法顺序访问一个聚合对象中的各个元素，而又不需要暴露该对象的内部表示。

我们以`List`为例，要遍历`ArrayList`，即使我们知道它的内部存储了一个`Object[]`数组，也不应该直接使用数组索引去遍历，因为这样需要了解集合内部的存储结构。如果使用`Iterator`遍历，那么，`ArrayList`和`LinkedList`都可以以一种统一的接口来遍历：

```java
List<String> list = ...
for (Iterator<String> it = list.iterator(); it.hasNext(); ) {
    String s = it.next();
}
```

实际上，因为Iterator模式十分有用，因此，Java允许我们直接把任何支持`Iterator`的集合对象用`foreach`循环写出来：

```java
List<String> list = ...
for (String s : list) {

}
```

然后由Java编译器完成Iterator模式的所有循环代码。

如何实现一个Iterator模式呢？我们以一个自定义的集合为例，通过Iterator模式实现倒序遍历：

```java
public class ReverseArrayCollection<T> implements Iterable<T> {
    private T[] array;

    public ReverseArrayCollection(T... objs) {
        this.array = Arrays.copyOfRange(objs, 0, objs.length);
    }

    public Iterator<T> iterator() {
        return new ReverseIterator();
    }

    class ReverseIterator implements Iterator<T> {
        // 索引位置:
        int index;

        public ReverseIterator() {
            // 创建Iterator时,索引在数组末尾:
            this.index = ReverseArrayCollection.this.array.length;
        }

        public boolean hasNext() {
            // 如果索引大于0,那么可以移动到下一个元素(倒序往前移动):
            return index > 0;
        }

        public T next() {
            // 将索引移动到下一个元素并返回(倒序往前移动):
            index--;
            return array[index];
        }
    }
}
```

使用内部类的好处是内部类隐含地持有一个它所在对象的`this`引用，可以通过`ReverseArrayCollection.this`引用到它所在的集合。上述代码实现的逻辑非常简单，但是实际应用时，如果考虑到多线程访问，当一个线程正在迭代某个集合，而另一个线程修改了集合的内容时，是否能继续安全地迭代，还是抛出`ConcurrentModificationException`，就需要更仔细地设计。

### 中介/调停者Mediator

用一个中介对象来封装一系列的对象交互。中介者使各个对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

![image-20220903144825329](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903144825329.png)

这个小系统有4个参与对象：

- 多选框；
- “选择全部”按钮；
- “取消所有”按钮；
- “反选”按钮。

它的复杂性在于，当多选框变化时，它会影响“选择全部”和“取消所有”按钮的状态（是否可点击），当用户点击某个按钮时，例如“反选”，除了会影响多选框的状态，它又可能影响“选择全部”和“取消所有”按钮的状态。

![image-20220903144849479](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903144849479.png)

如果我们引入一个中介，把多方会谈变成多个双方会谈，虽然多了一个对象，但对象之间的关系就变简单了

![image-20220903144925149](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903144925149.png)

下面我们用中介模式来实现各个UI组件的交互。首先把UI组件给画出来：

```java
public class Main {
    public static void main(String[] args) {
        new OrderFrame("Hanburger", "Nugget", "Chip", "Coffee");
    }
}

class OrderFrame extends JFrame {
    public OrderFrame(String... names) {
        setTitle("Order");
        setSize(460, 200);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        Container c = getContentPane();
        c.setLayout(new FlowLayout(FlowLayout.LEADING, 20, 20));
        c.add(new JLabel("Use Mediator Pattern"));
        List<JCheckBox> checkboxList = addCheckBox(names);
        JButton selectAll = addButton("Select All");
        JButton selectNone = addButton("Select None");
        selectNone.setEnabled(false);
        JButton selectInverse = addButton("Inverse Select");
        new Mediator(checkBoxList, selectAll, selectNone, selectInverse);
        setVisible(true);
    }

    private List<JCheckBox> addCheckBox(String... names) {
        JPanel panel = new JPanel();
        panel.add(new JLabel("Menu:"));
        List<JCheckBox> list = new ArrayList<>();
        for (String name : names) {
            JCheckBox checkbox = new JCheckBox(name);
            list.add(checkbox);
            panel.add(checkbox);
        }
        getContentPane().add(panel);
        return list;
    }

    private JButton addButton(String label) {
        JButton button = new JButton(label);
        getContentPane().add(button);
        return button;
    }
}
```

然后，我们设计一个Mediator类，它引用4个UI组件，并负责跟它们交互：

```java
public class Mediator {
    // 引用UI组件:
    private List<JCheckBox> checkBoxList;
    private JButton selectAll;
    private JButton selectNone;
    private JButton selectInverse;

    public Mediator(List<JCheckBox> checkBoxList, JButton selectAll, JButton selectNone, JButton selectInverse) {
        this.checkBoxList = checkBoxList;
        this.selectAll = selectAll;
        this.selectNone = selectNone;
        this.selectInverse = selectInverse;
        // 绑定事件:
        this.checkBoxList.forEach(checkBox -> {
            checkBox.addChangeListener(this::onCheckBoxChanged);
        });
        this.selectAll.addActionListener(this::onSelectAllClicked);
        this.selectNone.addActionListener(this::onSelectNoneClicked);
        this.selectInverse.addActionListener(this::onSelectInverseClicked);
    }

    // 当checkbox有变化时:
    public void onCheckBoxChanged(ChangeEvent event) {
        boolean allChecked = true;
        boolean allUnchecked = true;
        for (var checkBox : checkBoxList) {
            if (checkBox.isSelected()) {
                allUnchecked = false;
            } else {
                allChecked = false;
            }
        }
        selectAll.setEnabled(!allChecked);
        selectNone.setEnabled(!allUnchecked);
    }

    // 当点击select all:
    public void onSelectAllClicked(ActionEvent event) {
        checkBoxList.forEach(checkBox -> checkBox.setSelected(true));
        selectAll.setEnabled(false);
        selectNone.setEnabled(true);
    }

    // 当点击select none:
    public void onSelectNoneClicked(ActionEvent event) {
        checkBoxList.forEach(checkBox -> checkBox.setSelected(false));
        selectAll.setEnabled(true);
        selectNone.setEnabled(false);
    }

    // 当点击select inverse:
    public void onSelectInverseClicked(ActionEvent event) {
        checkBoxList.forEach(checkBox -> checkBox.setSelected(!checkBox.isSelected()));
        onCheckBoxChanged(null);
    }
}
```

Mediator模式经常用在有众多交互组件的UI上。为了简化UI程序，MVC模式以及MVVM模式都可以看作是Mediator模式的扩展。

### 备忘录Memento

主要用于捕获一个对象的内部状态，以便在将来的某个时候恢复此状态；最简单的备忘录模式就是保存到文件，打开文件。对于文本编辑器来说，保存就是把`TextEditor`类的字符串存储到文件，打开就是恢复`TextEditor`类的状态。对于图像编辑器来说，原理是一样的，只是保存和恢复的数据格式比较复杂而已。Java的序列化也可以看作是备忘录模式。

在使用文本编辑器的时候，我们还经常使用Undo、Redo这些功能。这些其实也可以用备忘录模式实现，即不定期地把`TextEditor`类的字符串复制一份存起来，这样就可以Undo或Redo。

标准的备忘录模式有这么几种角色：

- Memonto：存储的内部状态；
- Originator：创建一个备忘录并设置其状态；
- Caretaker：负责保存备忘录。

实际上我们在使用备忘录模式的时候，不必设计得这么复杂，只需要对类似`TextEditor`的类，增加`getState()`和`setState()`就可以了。

我们以一个文本编辑器`TextEditor`为例，它内部使用`StringBuilder`允许用户增删字符，为了支持这个`TextEditor`能保存和恢复状态，我们增加`getState()`和`setState()`两个方法：

```java
public class TextEditor {
    private StringBuilder buffer = new StringBuilder();

    public void add(char ch) {
        buffer.append(ch);
    }

    public void add(String s) {
        buffer.append(s);
    }

    public void delete() {
        if (buffer.length() > 0) {
            buffer.deleteCharAt(buffer.length() - 1);
        }
    }

    // 获取状态:
    public String getState() {
        return buffer.toString();
    }

    // 恢复状态:
    public void setState(String state) {
        this.buffer.delete(0, this.buffer.length());
        this.buffer.append(state);
    }
}
```

对这个简单的文本编辑器，用一个`String`就可以表示其状态，对于复杂的对象模型，通常我们会使用JSON、XML等复杂格式。

### 观察者Observer/发布-订阅Publish-Subscribe

它是一种通知机制，让发送通知的一方（被观察方）和接收通知的一方（观察者）能彼此分离，互不影响，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新

举例：假设一个电商网站，有多种`Product`（商品），同时，`Customer`（消费者）和`Admin`（管理员）对商品上架、价格改变都感兴趣，希望能第一时间获得通知。于是，`Store`（商场）可以这么写：

```java
public class Store {
    Customer customer;
    Admin admin;

    private Map<String, Product> products = new HashMap<>();

    public void addNewProduct(String name, double price) {
        Product p = new Product(name, price);
        products.put(p.getName(), p);
        // 通知用户:
        customer.onPublished(p);
        // 通知管理员:
        admin.onPublished(p);
    }

    public void setProductPrice(String name, double price) {
        Product p = products.get(name);
        p.setPrice(price);
        // 通知用户:
        customer.onPriceChanged(p);
        // 通知管理员:
        admin.onPriceChanged(p);
    }
}
```

有了分析上面那么多设计模式的经验，再看这里就会发现很明显的“通知”这个功能被多次重用，而且如果添加用户角色会更复杂，我们就可以用观察者模式，分离被观察者和观察者之间的耦合关系，让`Store`不能直接引用`Customer`和`Admin`，相反，它引用一个`ProductObserver`接口，任何人想要观察`Store`，只要实现该接口，并且把自己注册到`Store`即可

```java
public class Store {
    private List<ProductObserver> observers = new ArrayList<>();
    private Map<String, Product> products = new HashMap<>();

    // 注册观察者:
    public void addObserver(ProductObserver observer) {
        this.observers.add(observer);
    }

    // 取消注册:
    public void removeObserver(ProductObserver observer) {
        this.observers.remove(observer);
    }

    public void addNewProduct(String name, double price) {
        Product p = new Product(name, price);
        products.put(p.getName(), p);
        // 通知观察者:
        observers.forEach(o -> o.onPublished(p));
    }

    public void setProductPrice(String name, double price) {
        Product p = products.get(name);
        p.setPrice(price);
        // 通知观察者:
        observers.forEach(o -> o.onPriceChanged(p));
    }
}
```

就是这么一个小小的改动，使得观察者类型就可以无限扩充，而且，观察者的定义可以放到客户端：

```java
// observer:
Admin a = new Admin();
Customer c = new Customer();
// store:
Store store = new Store();
// 注册观察者:
store.addObserver(a);
store.addObserver(c);
```

甚至可以注册匿名观察者：

```java
store.addObserver(new ProductObserver() {
    public void onPublished(Product product) {
        System.out.println("[Log] on product published: " + product);
    }

    public void onPriceChanged(Product product) {
        System.out.println("[Log] on product price changed: " + product);
    }
});
```

![image-20220903151100802](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903151100802.png)

观察者模式也有很多变体形式。有的观察者模式把被观察者也抽象出接口（表示可以被观察 我觉得这个比较好理解）

```java
public interface ProductObservable { // 注意此处拼写是Observable不是Observer!
    void addObserver(ProductObserver observer);
    void removeObserver(ProductObserver observer);
}
```

对应的实体被观察者就要实现该接口：

```java
public class Store implements ProductObservable {
    ...
}
```

有些观察者模式把通知变成一个Event对象，从而不再有多种方法通知，而是统一成一种（类似回调一样 onEvent）

```java
public interface ProductObserver {
    void onEvent(ProductEvent event);
}
```

让观察者自己从Event对象中读取通知类型和通知数据。

广义的观察者模式包括所有消息系统。所谓消息系统，就是把观察者和被观察者完全分离，通过消息系统本身来通知：

![image-20220903151459127](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903151459127.png)

消息发送方称为Producer，消息接收方称为Consumer，Producer发送消息的时候，必须选择发送到哪个Topic。Consumer可以订阅自己感兴趣的Topic，从而只获得特定类型的消息。

使用消息系统实现观察者模式时，Producer和Consumer甚至经常不在同一台机器上，并且双方对对方完全一无所知，因为注册观察者这个动作本身都在消息系统中完成，而不是在Producer内部完成。

此外，注意到我们在编写观察者模式的时候，通知Observer是依靠语句：

```java
observers.forEach(o -> o.onPublished(p));
```

这说明各个观察者是依次获得的同步通知，如果上一个观察者处理太慢，会导致下一个观察者不能及时获得通知。此外，如果观察者在处理通知的时候，发生了异常，还需要被观察者处理异常，才能保证继续通知下一个观察者。

### 状态State

允许一个对象在其内部状态改变时改变它的行为。对象看起来似乎修改了它的类。

如何表示状态？我们定义一个`enum`就可以表示不同的状态。但不同的状态需要对应不同的行为，比如收到消息时：

```java
if (state == ONLINE) {
    // 闪烁图标
} else if (state == BUSY) {
    reply("现在忙，稍后回复");
} else if ...
```

状态模式的目的是为了把上述一大串`if...else...`的逻辑给分拆到不同的状态类中，使得将来增加状态比较容易。

例如，我们设计一个聊天机器人，它有两个状态：

- 未连线；
- 已连线。

对于未连线状态，我们收到消息也不回复：

```java
public class DisconnectedState implements State {
    public String init() {
        return "Bye!";
    }

    public String reply(String input) {
        return "";
    }
}
```

对于已连线状态，我们回应收到的消息：

```java
public class ConnectedState implements State {
    public String init() {
        return "Hello, I'm Bob.";
    }

    public String reply(String input) {
        if (input.endsWith("?")) {
            return "Yes. " + input.substring(0, input.length() - 1) + "!";
        }
        if (input.endsWith(".")) {
            return input.substring(0, input.length() - 1) + "!";
        }
        return input.substring(0, input.length() - 1) + "?";
    }
}
```

状态模式的关键设计思想在于状态切换，我们引入一个`BotContext`完成状态切换：

```java
public class BotContext {
	private State state = new DisconnectedState();

	public String chat(String input) {
		if ("hello".equalsIgnoreCase(input)) {
            // 收到hello切换到在线状态:
			state = new ConnectedState();
			return state.init();
		} else if ("bye".equalsIgnoreCase(input)) {
            // 收到bye切换到离线状态:
			state = new DisconnectedState();
			return state.init();
		}
		return state.reply(input);
	}
}
```

聊天机器人

```java
Scanner scanner = new Scanner(System.in);
BotContext bot = new BotContext();
for(;;){
	System.put.print("> ");
	String input = scanner.nextLine();
	String output = bot.chat(input);
	System.out.println(output.isEmpty() ? "(no reply)" : "< " + output);
}
```

### 策略Strategy

是指，定义一组算法并把其封装到一个对象中。然后在运行时，可以灵活的使用其中的一个算法。就像把函数作为函数的参数，整体流程是确定的，但具体的细节还可以自定义

举例：Arrays

```java
import java.util.Arrays;
public class Main {
    public static void main(String[] args) throws InterruptedException {
        String[] array = { "apple", "Pear", "Banana", "orange" };
        Arrays.sort(array, String::compareToIgnoreCase);
        System.out.println(Arrays.toString(array));
    }
}
```

如果我们想忽略大小写排序，就传入`String::compareToIgnoreCase`，如果我们想倒序排序，就传入`(s1, s2) -> -s1.compareTo(s2)`，这个比较两个元素大小的算法就是策略。

举例：购物车结算

```java
public interface DiscountStrategy{
	// 计算折扣额度
	BigDecimal getDiscount(BigDecimal total);
}
```

实现各种策略，普通用户

```java
public class UserDiscountStrategy implements DiscountStrategy{
	public BigDecimal getDiscount(BigDecimal total){
		// 普通会员9折
		return total.multiply(new BigDecimal("0.1")).setScale(2, RoundingMode.DOWN);
	}
}
```

满减策略如下：

```java
public class OverDiscountStrategy implements DiscountStrategy {
    public BigDecimal getDiscount(BigDecimal total) {
        // 满100减20优惠:
        return total.compareTo(BigDecimal.valueOf(100)) >= 0 ? BigDecimal.valueOf(20) : BigDecimal.ZERO;
    }
}
```

最后应用策略需要一个`DiscountContext`

```java
public class DiscountContext{
	// 持有某个策略
	private DiscountStrategy strategy = new UserDiscountStrategy();
	// 允许客户端设置新策略
	public void setStrategy(DiscountStrategy strategy){
		this.strategy = strategy;
	}

	public BigDecimal calculatePrice(BigDecimal total){
		return total.subtract(this.strategy.getDiscount(total)).setScale(2);
	}
}
```

调用方必须首先创建一个DiscountContext，并指定一个策略（或者使用默认策略），即可获得折扣后的价格：

```java
DiscountContext ctx = new DiscountContext();

// 默认使用普通会员折扣:
BigDecimal pay1 = ctx.calculatePrice(BigDecimal.valueOf(105));
System.out.println(pay1);

// 使用满减折扣:
ctx.setStrategy(new OverDiscountStrategy());
BigDecimal pay2 = ctx.calculatePrice(BigDecimal.valueOf(105));
System.out.println(pay2);

// 使用Prime会员折扣:
ctx.setStrategy(new PrimeDiscountStrategy());
BigDecimal pay3 = ctx.calculatePrice(BigDecimal.valueOf(105));
System.out.println(pay3);
```

![image-20220903162006786](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903162006786.png)

策略模式的核心思想是在一个计算方法中把容易变化的算法抽出来作为“策略”参数传进去，从而使得新增策略不必修改原有逻辑。

### 模板Template

emmmm 我的感觉是就像普通的面对对象的思想，充分利用接口，继承和抽象

举例：假设我们开发了一个从数据库读取设置的类：

```java
public class Setting {
    public final String getSetting(String key) {
        String value = readFromDatabase(key);
        return value;
    }

	private String readFromDatabase(String key) {
        // TODO: 从数据库读取
    }
}
```

由于从数据库读取数据较慢，我们可以考虑把读取的设置缓存起来，这样下一次读取同样的key就不必再访问数据库了。但是怎么实现缓存，暂时没想好，但不妨碍我们先写出使用缓存的代码：

```java
public class Setting {
    public final String getSetting(String key) {
        // 先从缓存读取:
        String value = lookupCache(key);
        if (value == null) {
            // 在缓存中未找到,从数据库读取:
            value = readFromDatabase(key);
            System.out.println("[DEBUG] load from db: " + key + " = " + value);
            // 放入缓存:
            putIntoCache(key, value);
        } else {
            System.out.println("[DEBUG] load from cache: " + key + " = " + value);
        }
        return value;
    }
}
```

整个流程没有问题，但是，`lookupCache(key)`和`putIntoCache(key, value)`这两个方法还根本没实现，怎么编译通过？这个不要紧，我们声明抽象方法就可以（把上面的Setting整个换成一手抽象），如何实现`lookupCache(key)`和`putIntoCache(key, value)`这两个方法就交给子类了

```java
public abstract class AbstractSetting {
    public final String getSetting(String key) {
        String value = lookupCache(key);
        if (value == null) {
            value = readFromDatabase(key);
            putIntoCache(key, value);
        }
        return value;
    }

    protected abstract String lookupCache(String key);

    protected abstract void putIntoCache(String key, String value);
}
```

子类其实并不关心核心代码中`getSetting(key)`的逻辑，它只需要关心如何完成两个小小的子任务就可以了。

假设我们希望用一个`Map`做缓存，那么可以写一个`LocalSetting`：

```java
public class LocalSetting extends AbstractSetting {
    private Map<String, String> cache = new HashMap<>();

    protected String lookupCache(String key) {
        return cache.get(key);
    }

    protected void putIntoCache(String key, String value) {
        cache.put(key, value);
    }
}
```

如果我们要使用Redis做缓存，那么可以再写一个`RedisSetting`：

```java
public class RedisSetting extends AbstractSetting {
    private RedisClient client = RedisClient.create("redis://localhost:6379");

    protected String lookupCache(String key) {
        try (StatefulRedisConnection<String, String> connection = client.connect()) {
            RedisCommands<String, String> commands = connection.sync();
            return commands.get(key);
        }
    }

    protected void putIntoCache(String key, String value) {
        try (StatefulRedisConnection<String, String> connection = client.connect()) {
            RedisCommands<String, String> commands = connection.sync();
            commands.set(key, value);
        }
    }
}
```

客户端代码使用本地缓存的代码这么写：

```java
AbstractSetting setting1 = new LocalSetting();
System.out.println("test = " + setting1.getSetting("test"));
System.out.println("test = " + setting1.getSetting("test"));
```

要改成Redis缓存，只需要把`LocalSetting`替换为`RedisSetting`：

```java
AbstractSetting setting2 = new RedisSetting();
System.out.println("autosave = " + setting2.getSetting("autosave"));
System.out.println("autosave = " + setting2.getSetting("autosave"));
```

可见，模板方法的核心思想是：父类定义骨架，子类实现某些细节。

为了防止子类重写父类的骨架方法，可以在父类中对骨架方法使用`final`。对于需要子类实现的抽象方法，一般声明为`protected`，使得这些方法对外部客户端不可见。

Java标准库也有很多模板方法的应用。在集合类中，`AbstractList`和`AbstractQueuedSynchronizer`都定义了很多通用操作，子类只需要实现某些必要方法。

### 访问者Vistor

是一种操作一组对象的操作，它的目的是不改变对象的定义，但允许新增不同的访问者，来定义新的操作

举例：我们要递归遍历某个文件夹的所有子文件夹和文件，然后找出`.java`文件，正常的做法是写个递归

```java
void scan(File dir, List<File> collector) {
    for (File file : dir.listFiles()) {
        if (file.isFile() && file.getName().endsWith(".java")) {
            collector.add(file);
        } else if (file.isDir()) {
            // 递归调用:
            scan(file, collector);
        }
    }
}
```

问题在于扫描目录的逻辑和处理文件的逻辑是一起的，如果下次需要增加一个清理`.class`文件的功能，就必须再重复写扫描逻辑

因此，访问者模式先把数据结构（这里是文件夹和文件构成的树型结构）和对其的操作（查找文件）分离开，以后如果要新增操作（例如清理`.class`文件），只需要新增访问者，不需要改变现有逻辑。

用访问者模式改写：

首先需要定义访问者接口（能干的事）

```java
public interface Vistor{
	// 访问文件夹
	void visitDir(File dir);
	// 访问文件
	void visitFile(File file);
}
```

还要定义能持有文件夹和文件的数据结构FileStructure，我们给它传入一个访问者

```java
public class FileStructure{
	// 根目录
	private File path;
	public FileStructure(File path){
		this.path = path;
	}
    public void handle(Visitor visitor) {
		scan(this.path, visitor);
	}

	private void scan(File file, Visitor visitor) {
		if (file.isDirectory()) {
            // 让访问者处理文件夹:
			visitor.visitDir(file);
			for (File sub : file.listFiles()) {
                // 递归处理子文件夹:
				scan(sub, visitor);
			}
		} else if (file.isFile()) {
            // 让访问者处理文件:
			visitor.visitFile(file);
		}
	}
}
```

这样，我们就把访问者的行为抽象出来了。如果我们要实现一种操作，例如，查找`.java`文件，就传入`JavaFileVisitor`：

```java
FileStructure fs = new FileStructure(new File("."));
fs.handle(new JavaFileVisitor());
```

这个`JavaFileVisitor`实现如下：

```java
public class JavaFileVisitor implements Visitor {
    public void visitDir(File dir) {
        System.out.println("Visit dir: " + dir);
    }

    public void visitFile(File file) {
        if (file.getName().endsWith(".java")) {
            System.out.println("Found java file: " + file);
        }
    }
}
```

类似的，如果要清理`.class`文件，可以再写一个`ClassFileClearnerVisitor`：

```java
public class ClassFileCleanerVisitor implements Visitor {
	public void visitDir(File dir) {
	}

	public void visitFile(File file) {
		if (file.getName().endsWith(".class")) {
			System.out.println("Will clean class file: " + file);
		}
	}
}
```

可见，访问者模式的核心思想是为了访问比较复杂的数据结构，不去改变数据结构，而是把对数据的操作抽象出来，在“访问”的过程中以回调形式在访问者中处理操作逻辑。如果要新增一组操作，那么只需要增加一个新的访问者。

## 小结

- **工厂方法 Factory Method**

不通过new来创建实例，而是抽象出接口xxxFactory（工厂，其中会定义一个静态方法用来在内部直接创建实例，留给外部调用getFactory），再写实现类xxxFactoryImpl（继承自接口，其中定义了具体的代码逻辑 返回抽象产品），客户端只需要和接口&抽象产品打交道即可

- **静态工厂方法 Static Factory Method**

上面那个还是有点复杂，取消了接口，直接xxxFactory类通过静态方法返回抽象产品，这里的xxxFactory既是产品又是静态工厂

- **抽象工厂 Abstract Factory**

有多个实际工厂 且分别对应多个实际产品，用抽象工厂模式会将这些实际工厂继续向上抽象，对客户端来说只需要知道抽象工厂和产品接口即可（中间的实际产品和实际工厂都会被屏蔽 不需要管）

![image-20220902110636017](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220902110636017.png)

- **生成器 Builder**

将大的任务拆分，用多个小型工厂来创建一个完整对象

*举例：md to html转换器，对不同类型文本的转换

- **原型 Protype**

创建新对象的时候，根据现有的一个原型来创建

*举例：已有一个`String[]`数组，想创建一个一模一样的`String[]`数组，我们会将现有数组的元素复制到新数组，这个过程封装起来就变成了原型模式

- **单例 Singleton**

确保在一个进程中某个类有且仅有一个实例；在这种模式下构造方法为private防止调用方创建实例，类内部通过`private static`来引用唯一创建的实例，或者通过`public static`变量返回此唯一实例

*举例：`Runtime runtime = Runtime.getRuntime();`

还可以通过enum枚举类来实现单例（因为java保证枚举类的每个枚举都是单例），同时避免了第一种方式实现Singleton的一个潜在问题，即序列化和反序列化会绕过普通类的`private`构造方法从而创建出多个实例，而枚举类就没有这个问题

- **适配器 Adapter/Wrapper**

近似理解为充电器转接口，将A接口转换为B接口；或者可以理解为接口的强制类型转换（近似）

- **桥接 Bridge**

解决继承可能出现的子类爆炸情况，适用于多个需求同时排列组合

*举例：汽车品牌×动力类型

![image-20220902233406811](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220902233406811.png)

- **组合 Composite**

抽象出最基本的节点类型，使得叶子对象和容器对象具有一致性，从而形成统一的树形结构，在具体的实现类再细分实现的内容

*举例：html渲染器

![image-20220903000328506](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903000328506.png)

- **装饰器Decorator**

在运行期动态给某个对象的实例增加功能的方法，与桥接模式的区别在于装饰器模式下把核心功能和附加功能分开了，而不是桥接中的二者并行，虽然二者都是为了避免继承中可能出现的子类爆炸，但装饰器是对已经有的功能做修饰（最终产品可以不被修饰，也可以不断被嵌套），桥接是把并行的多个功能进行排列组合

![image-20220903101312952](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903101312952.png)

*举例：html渲染器，但文本可以附加一些比如加粗、斜体之类的效果

- **门面/外观 Facade**

类似房屋中介，内部有多个子系统提供服务，但对外只提供一个统一的Facade入口

*举例；委托中介帮忙办理开公司的业务，其中包含工商注册、银行开户、纳税登记，但我们只需要给他信息就行，不需要知道核心是怎么跑的

- **享元 Flyweight**

如果一个对象实例已经创建就不可变，那直接返回一个共享的实例即可；与单例的区别在于完全不可变（毕竟是缓存对象）

- **代理 Proxy**

与适配器Adpater很相似，但不是将A接口转为B接口，它还是到A接口，只不过在包装的时候可以多加一些额外的代码

*举例：有A接口 其中实现了a方法，而AProxy类继承自A，其中的a方法可以增加鉴权，如果想要其它类型的代理只需要继续继承A即可

- **远程代理 Remote Proxy**

*举例：Java RMI

- **虚代理 Virtual Proxy**

让调用者先持有一个代理对象，但真正的对象尚未创建，直到用的时候才创建

*举例：JDBC的连接池返回的JDBC连接（Connection对象）就是一个虚代理，即获取连接时根本没连上，直到第一次查询或更新操作时才会创建真正的JDBC连接

- **保护代理 Protection Proxy**

用代理对象控制对原始对象的访问，常用于鉴权

- **智能引用 Smart Reference**

也是一种代理对象，如果有很多客户端对它进行访问，通过内部的计数器可以在外部调用者都不使用后自动释放

- **责任链 Chain of Responsibility**

*举例：web中的filter, interceptor

- **命令 Command**

把请求封装成一个命令，然后执行该命令

*举例：文本编辑器

![image-20220903142302481](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903142302481.png)

把调用方发送命令和执行方执行命令分开了，客户端需要操心的是命令内容，而不是具体的操作（函数）；当TextEditor本身复杂到一定程度、需要持Undo、Redo的功能时，就需要使用命令模式，因为我们可以给每个命令增加`undo()`：然后把执行的一系列命令用`List`保存起来，就既能支持Undo，又能支持Redo。这个时候，我们又需要一个`Invoker`对象，负责执行命令并保存历史命令：

![image-20220903142424888](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903142424888.png)

- **解释器Interpreter**

是一种针对特定问题设计的一种解决方案

*举例：正则表达式；JDBC连接语句

- **迭代器Iterator**

提供一种方法顺序访问一个聚合对象中的各个元素，而又不需要暴露该对象的内部表示。

- **中介/调停者 Mediator**

提供一种方法顺序访问一个聚合对象中的各个元素，而又不需要暴露该对象的内部表示；经常用在有众多交互组件的UI上

*举例：点餐输入框，点击会影响选择状态（选择全部/取消所有/反选）

![image-20220903144849479](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903144849479.png)

设计中介（引入UI组件，并用类似回调的函数与之交互）

![image-20220903144925149](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903144925149.png)

- **备忘录 Memento**

主要用于捕获一个对象的内部状态，以便在将来的某个时候恢复此状态

*举例：反序列化

- **观察者 Observer/发布-订阅 Publish-Subscribe**

一种通知机制，让发送通知的一方（被观察方）和接收通知的一方（观察者）能彼此分离，互不影响，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新

*举例：电商平台，消费者和管理员同时对商品的上架、价格改变都感兴趣，需要收到通知

- **状态 State**

允许一个对象在其内部状态改变时改变它的行为。对象看起来似乎修改了它的类。

*举例：聊天机器人（已连线/未连线状态的切换）

- **策略 Strategy**

就像把函数作为函数的参数，整体流程是确定的，但具体的细节还可以自定义

*举例：商场中不同规格的会员折扣

![image-20220903162006786](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220903162006786.png)

- **模板 Template**

充分利用接口，继承和抽象；父类定义骨架，子类实现某些细节

- **访问者 Visitor**

是一种操作一组对象的操作，它的目的是不改变对象的定义，但允许新增不同的访问者，来定义新的操作；核心思想是为了访问比较复杂的数据结构，不去改变数据结构，而是把对数据的操作抽象出来，在“访问”的过程中以回调形式在访问者中处理操作逻辑。如果要新增一组操作，那么只需要增加一个新的访问者。

*举例：我们要递归遍历某个文件夹的所有子文件夹和文件，然后找出`.java`文件，正常的做法是写个递归，但问题在于扫描目录的逻辑和处理文件的逻辑是一起的，如果下次需要增加一个清理`.class`文件的功能，就必须再重复写扫描逻辑；因此访问者模式先把数据结构（这里是文件夹和文件构成的树型结构）和对其的操作（查找文件）分离开，以后如果要新增操作（例如清理`.class`文件），只需要新增访问者，不需要改变现有逻辑。

------

{{% spoiler "一些代码方面不算思考的思考" %}}

最近把一个自用的信息收集向工具（python编写 streamlit强力驱动）[200scan](https://github.com/AmiaaaZ/200scan)转为公开仓库了，使用体验上来说目前觉得还挺不错（大概）

单论代码来说真没多少含金量 核心逻辑极其简单，但在写的时候还是稍微悟了一小点东西，注意到了平时写的时候不会注意到的点。首先是代码风格，比如变量名是下划线/驼峰？是否完全按照PEP8的编码规范？目前我的变量名是下划线，函数名是驼峰，格式化字符串全部用f-string，基本遵守PEP8让代码更具有可读性；代码风格其实都是花架子，真正想要编写一个好用的工具时必然会被迫去注意性能问题，比如我要解析requests返回的页面，是用re还是bs4？这个循环换个方式写会不会更好？这个字典操作是不是太复杂了？要不要多加一个try catch？等等等等这类问题是我以前从不会注意的，感觉代码能跑就行，快不快无所谓；最后还会被迫注意是否具有可扩展性，比如这个功能重复使用是否可以抽象为函数或者类？然后就会不自觉地去搜索对应的最佳实践，还有其它语言在对应方面的表现是否更好

最开始200scan只是一个简单的命令行工具，只有读urls.txt然后输出对应的results.txt的功能，后来用的多了就希望能有个高交互的GUI界面，尝试了一下javafx就放弃了，实在是不喜欢java的臃肿，之后就转向了宇宙之神streamlit，但是目前写过200Scan, fofa WebViewer, check AV这三个功能以后也逐渐感受到了它的局限，不过streamlit的版本迭代非常快，估计也是问题不大

说来真的是无比惭愧，接触python这个语言也两年多了，但还是不能写起代码来得心应手，java也磕磕绊绊学了小半年了，进行代码审计的时候还是会头大，go一直说要学，到现在也只会个基本语法……太菜了太菜了啊啊啊啊啊，都大三了还这么沸物，还是个铁沸物

这学期我必好好学习，拼搏半年 我要转安研！

{{% /spoiler %}}

