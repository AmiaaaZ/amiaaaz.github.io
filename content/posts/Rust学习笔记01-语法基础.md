---
title: "Rust学习笔记Ⅰ"
slug: "rust-stduy-notes-01"
description: "锐意学习中:)"
date: 2023-06-16T23:00:26+08:00
categories: ["NOTES&SUMMARY"]
series: ["Rust学习笔记"]
tags: ["Rust"]
draft: false
toc: true
---

Q: 为啥突然学Rust？

A: ~~没有为啥，闲的~~ 一方面是入夏以来输入/输出都很少，不是好现象，另一方面是也在听南大的操作系统课，rust正好就是对标c/c++的系统级编程语言，不如拿来学学 跟上最新的语言特性和发展进程x

------

## 环境配置

*为了上手方便，暂时下文涉及到的都是windows环境（我懒）

- 前提：Visual Studio及相关组件 https://visualstudio.microsoft.com/zh-hans/downloads/

如果不安装这些组件会直接影响Rust的正常运行

如果直接用rustup-init.exe安装，会默认安装Windows 11 SDK，我是手动在Visual Studio Installer中选择了Windows 10 SDK

![image-20230614154924534](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230614154924534.png)

*也可以不用官方的installer、不安装Visual Studio，用独立出来的cpp build tools~

![image-20230616164819078](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616164819078.png)

- 本体：https://www.rust-lang.org/tools/install

- IDE：Intellij Rust插件

曾经我也是重视的VS code信徒，但其实看似简单之下隐藏了很多繁琐的配置，而且内存占用其实并不小……还是入股Intellij了

- Hello, world!

![image-20230614155344723](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230614155344723.png)

Intellij Rust默认配置下会直接生成Cargo项目 方便管理

- 卸载Rust

如果一个不小心安装有问题...

```
rustup self uninstall
```

## 语言特色

这几年断断续续接触了很多语言（C, Python, PHP, Java, Javascript, Go....），逐渐感受到各个编程语言在“和而不同”之余 存在各自的“舒适区”，这个舒适区由语言特性等很多因素共同决定；如果硬把需求安排在一个不适合的语言，那真的是如鲠在喉，语言是工具 不是目的

- 优点：速度快，内存利用率高，性能好，生成的可执行文件为静态编译，跨平台
- 缺点：编译器严格、编译耗时长，学习曲线陡峭
- 与其他语言的对比：属于强类型，代码风格类似C/C++，会用项目的概念来组织代码文件（类似Java的Maven）
- 敏感肌也很喜欢：可以编写shellcode loader
- 杀手级特性：让可能出现的安全问题、undefined behavior消失在编译前

## 语法基础

use this book~ [The Rust Programming Language](https://rust-book.cs.brown.edu/)

### 变量&可变性

`const`修饰的都是完全不可变的，必须指明数据类型，作用域为全局

`let`修饰的不可变，`let mut`修饰的可变，作用域为当前函数

shadowing的存在使我们可以使用同名变量，但只有可变的变量才可以被shadowing

```rust
fn main() {
    const SECRET_NUM: u32 = 12;
    let a = "a";
    // a = "b";                     // will error, not mutable
    let mut x: u32 = 1;
    {
        let mut x = x;
        x += 2;                     // shadowing
        println!("{x}");            // 3
        println!("{SECRET_NUM}")    // 12
    }
    println!("{x}");                // 1
    println!("{SECRET_NUM}")        // 12
}
```

### 数据类型

rust是statically typed language静态类型语言，写代码时没有显式声明的类型都会由编译器进行推断，如果发生错误会无法成功build

#### 标量

- 整型：默认i32，isize/usize主要作为某些集合的索引，编译器会检测interger overflow

![image-20230615035627186](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615035627186.png)

*debug模式编译下会检查integer overflow，release构建中则不会检查，如果出现此类错误 rust会进行two's complement wrapping的操作（也就是u8下 256->0, 257->1），不会因此panic 但可能出现非预期情况，不鼓励这样做；如果确实需要这样的“优化”，可选用如下函数

![image-20230615040141460](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615040141460.png)

*对这一特性我表示存疑，本机环境cargo 1.70.0 (ec8a8a0ca 2023-04-25)，并不会因为加上`--release`就不检查溢出了，还是会在compile阶段报错退出

![image-20230615040800001](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615040800001.png)

- 浮点型：默认f64（与f32速度几乎一样 但精度更高）

- 布尔型：true, false(fool)
- 字符型：单引号char代表一个unicode标量值

#### 复合类型

- 元组tuple 可包含不同类型数据, 数组array 必须每个元素类型相同

```rust
fn main() {
    let tup = (500, 3.14, 42);	// tup: (i32, f64, u8)
    let (x, y, z) = tup;		// destructuring
    println!("{z}");			// 42
    println!("{}", tup.1)		// 3.14
}
```

```rust
fn main(){
    let t = ([1; 2], [3; 4]);		// t: ([i32; 3], [i32; 4])
    let (a, _) = t;					// a = [1, 1], _ = [3, 3, 3]
    println!("{}", a[0] + t.1[0]);	// 4
}
```

*当出现数组元素越界访问时不会出现编译错误，但会出现运行时错误而panic退出

```rust
use std::io;

fn main() {
    let a = [1, 2, 3, 4, 5];

    println!("Please enter an array index.");

    let mut index = String::new();

    io::stdin()
        .read_line(&mut index)
        .expect("Failed to read line");

    let index: usize = index
        .trim()
        .parse()
        .expect("Index entered was not a number");

    let element = a[index];

    println!("The value of the element at index {index} is: {element}");
}
```

![image-20230615044316550](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615044316550.png)

### 函数

函数和变量名均使用snake case规范（下划线分隔小写单词），被使用的函数可位于任何位置，`main`为入口

函数声明必须带参数类型

可以用`{}`创建新作用域（代码块），`{}`是会计算并返回值的表达式，表达式的结尾没有分号，语句有分号、不返回值

函数的返回值无需命名，但需要声明类型

```rust
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {}", x);
}

fn plus_one(x: i32) -> i32 {
    x + 1	// `x + 1;` will error
}
```

### 控制流

可以直接对变量使用if/else，但要小心由此可能产生的类型问题：变量必须只有一个类型

循环有loop/while/for，loop需要手动停止循环，可以通过设置label来停止嵌套loop（适用于多次重试）

```rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {count}");
        let mut remaining = 10;

        loop {
            println!("remaining = {remaining}");
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }

        count += 1;
    }
    println!("End count = {count}");
}
```

for可以方便的遍历数组，方便程度不亚于python

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {element}");
    }
}
```

```rust
fn main() {
    for number in (1..4).rev() {
        println!("{number}!");
    }
    println!("LIFTOFF!!!");
}
```

### 所有权

所有权ownership是rust的特性，正是它让rust做到内存安全和无需垃圾回收

这里的“安全”，在rust中意味着完全不容许[undefined behavior](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)的存在（即使unsafe也不行），为了实现这一目标 rust将大量的功夫用在了compile-time而不是run-time，这样减少了潜在的运行时bug 也侧面提升了性能（减少运行的check），所有权则是这一理念下 内存安全方面的具体实践

#### 堆&Box

rust中，和函数有关的参数等数据存于栈上，堆上的数据则可以不依赖函数独立分配，可以使用`Box::new`来手动分配数据到堆，所有权会自动在“合适的时机”释放这块内存

**Box deallocation principle(fully correct):** if a variable owns a box, when Rust deallocates the variable's frame, the Rust deallocates the box's heap memory.
原则：当使用Box的变量在它的作用域内结束、被释放，那Box所在的堆内存也会被释放（也就是上面说的“合适的时机”，这一释放过程都被rust自动管理，无需手动控制）

很多数据结构内部都是用Box实现的，比如`Vec`, `String`, `HashMap`

下面的栗子里，但执行到第四行时会报错，因为first已经不指向`Ferris`了

```rust
fn main() {
    let first = String::from("Ferris");	// L1
    let full = add_suffix(first);		// L4
    println!("{full}, originally {first}"); // L5 will error!!!
}

fn add_suffix(mut name: String) -> String {
    // L2
    name.push_str(" Jr."); // L3
    name
}
```

![image-20230615140242885](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615140242885.png)

实际过程则是这样的：

![image-20230615135453255](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615135453255.png)

在L2阶段时发生指针复制，原有指针未消失，但我们不能再直接用原来的指针，也就是下面的规则：

**Moved heap data principle:** if a variable x moves ownership of heap data to another variable y, then x cannot be used after move.

`clone`方法可以在指针复制的同时不丢失原指针的所有权，“绕过”上面的规则

~~在同一作用域内、只在栈上拷贝的数据也不受上述规则的限制（废话了，毕竟是在栈上）~~

![image-20230615141118932](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615141118932.png)

*答错的一道quiz：选出下面会出现undefined behavior的选项

![image-20230615141806059](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615141806059.png)

关键在于：在b被传入`move_a_box`后、b才会被释放

#### 引用&借用

*这部分内容有C的基础会好理解一些

非常显然，如果严格按照上面所有权的规则进行 变量被moved之后再想找到原来的值就需要在函数上添加一个返回值，然而我们经常需要使用那个值！

引用reference就是解决这个问题的，它作为Non-owning pointer存在，允许使用堆上的值 但不直接指向堆

```rust
fn main() {
    let m1 = String::from("Hello");
    let m2 = String::from("world");	// L1
    greet(&m1, &m2); // L3
    let s = format!("{} {}", m1, m2);
}

fn greet(g1: &String, g2: &String) {
    // L2
    println!("{} {}!", g1, g2);
}
```

![image-20230615143339298](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615143339298.png)

`&m1`作为m1的引用、再作为greet的参数，这一行为被称为借用borrowing

`*`作为前缀表示解引用，可以读出具体数据，修改`*x`的值就是修改`x`指向堆内存的值

![image-20230615151017338](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230615151017338.png)

只有第一次的解引用`*x`可以被修改值并同步堆内存的修改，二次解引用的`**&x`只可以读值不可以修改，而`&*x`是直接指向值的引用指针、和`*x`的数据类型并不一样

`*x`, `**&x`和`*&*x`三者是完全等价的（数据类型、值），都可以直接读出堆内存指向的值，但只有`*x`可被改值

**Pointer Safety Principle:** data should never be aliased and mutated at the same time.
数据不可以<u>同时</u> 既有别名（存在引用）、又本身可变

![image-20230616143232706](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616143232706.png)

在上面的例子中，当`vec.push(4)`被执行时，原`vec`的cap和len都需要增大，为了满足新的cap和len的需求 很可能会重新分配在堆上的位置，所以`num`是否还指向`vec[2]`就是undefined behavior，很可能出现潜在的安全问题，所以这一行为会被编译器阻止

borrow checker的核心原理是检查它们的权限

- read: data can be copied to another location
- write: data can be mutated in-place
- own: data can be moved or dropped

这些权限只存在于编译期用于编译器来进行检查，而引用/别名的出现会暂时对原有变量的权限做出改变；让我们深入了解在上面例子中 read, write, own权限都发生了什么样的变化：

![image-20230616145353540](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616145353540.png)

我们可以看到，在`vec`被`num`借用期间，`vec`会丢失write和owner权限，而对应的`num`如果可变则获取write和owner权限 不可变则只获得owner权限，`*num`的权限和被借用的`vec`保持一致；一旦存在引用就会使数据的权限发生变化（暂时变成read-only），当引用消失 权限就会立刻回收，把write+owner权限还给原数据

而上上张图（出现undefined behavior的反例）中则有这样的权限关系

![image-20230616151045240](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616151045240.png)

编译时会在`vec.push(4)`就报错 而不是第四行

![image-20230616151300402](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616151300402.png)

浅层原因（蛋）是borrow checker已经检测到 第三行的`vec`只有read，而write权限被`num`借走了，所以此时`vec`不可变 导致报错，深层原因（鸡）则是rust为了避免可能存在的安全问题 而设计了这样的borrow checker（先鸡后蛋）

上面所有提到的`&`引用都是不可变引用immutable reference（也叫做shared references），它作为non-owning pointer 只作为不可变的别名；然而我们也可以让它暂时拥有write权限（仍然非owner），也就是`&mut`，可变引用mutable reference（也叫做unique reference）

![image-20230619144108851](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619144108851.png)

对比前面的不可变引用`num`有这些区别：

- 不可变引用：原数据仍有read权限，`num`只读+owner，`*num`只读
- 可变引用：原数据失去所有权限，`num`只读+owner，`*num`可读可写

在可变引用存在时，实际是允许了可变而避免了别名——原数据失去read权限，而`*可变引用`可读可写，可以直接通过`*可变引用`操作原数据内容

可变引用也可以降权为read-only的引用

![image-20230619150722466](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619150722466.png)

这里的`num2`是对`*num`的引用，因此把`*num`的write和owner权限借走了，又因为是不可变引用，`*num2`只读 不可写

上面讨论的都是在顺序执行流里 权限在引用存在时的流动关系，当if-else等控制流存在时 基本相同，不再赘述

*两道比较复杂的题，有亿点让人头大

![image-20230616161802457](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616161802457.png)

当rust判断 数据存在引用时，就会失去原有的write权限，当引用结束了它的生命周期，write会被归还

![image-20230616162437517](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230616162437517.png)

![image-20230619153804139](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619153804139.png)

这个题 其它几个选项的描述都是对的，但只有use after being free的解释是最根本的

作为Pointer Safety Principle的一部分，borrow checker也会强制要求数据的生存时间必须大于它的任何引用，然鹅 当引用作为函数的输入或输出时，rust是无法确定这个引用的生存周期的，rust在这种情况下引入了F权限

![image-20230619154721628](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619154721628.png)

![image-20230619155103418](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619155103418.png)

这个例子就不会被成功编译，因为只从函数声明来看无法确定输出的`&String`类型的对象是`strings`的引用还是`default`的引用；如果`default`进入了`first_or`的程序流、最后返回`default`，在经过drop之后就无法正确打印`s`了，因为`s`指向的`default`已经无了

![image-20230619155304536](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619155304536.png)

这样的例子也是不安全的，因为当函数执行完毕后`&s`会消失

#### 如何修复报错

##### **Returning a reference to the stack**

*核心：返回引用需要关心 引用指向的原数据能不能活到函数提供返回值，如果坚持返回引用 需要加生命周期的前缀，或者放弃返回引用 直接返回数据本身

```rust
fn return_a_string()-> &String{
	let s = String::from("Hello, world");
	&s
}
```

这个例子中当需要返回`&s`时它已经寄了，不能保证`s`获得足够长

有4种方式可以延长字符串的寿命

1. 将返回类型由`&String`改为`String`

```rust
fn return_a_string() -> String {
    let s = String::from("Hello world");
    s
}
```

2. 返回静态字符串（当我们无需涉及堆内存分配时

```rust
fn return_a_string() -> &'static str {
    "Hello world"
}
```

3. 显式调用gc，将borrow-checking挪到运行时

```rust
use std::rc::Rc;
fn return_a_string() -> Rc<String> {
    let s = Rc::new(String::from("Hello world"));
    Rc::clone(&s)
}
```

`Rc::clone`仅克隆指向`s`的指针 而不克隆数据，在运行时 Rc会检查最后一个指向数据的Rc何时删除 并在那之后释放数据

4. 添加可变引用作为函数参数

```rust
fn return_a_string(output: &mut String) {
    output.replace_range(.., "Hello world");
}
```

由函数调用放负责为返回的字符串创建空间，但是如果运用得当这种做法时memory-efficient的

##### Not enough permissions

*当试图修改只读数据，或在存在引用时删除数据

![image-20230619164157328](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619164157328.png)

这个例子的第二行就会报错，因为`name`本身是不可变引用 不存在write权限

我们希望这样调用

![image-20230619164821878](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230619164821878.png)

`first`指向`name[0]`，`name.push()`重新分配了`name` 导致打印`first`会报错；也就是在存在`&name`时还试图修改`name`

如何修改？如果只是简单粗暴的修改接收参数的类型（从不可变引用改为可变引用）是很不负责任的做法，因为传入的参数可被修改 这不是调用方所预期的行为

另一个选择是将引用改为数据，确保传入的`name`一定拥有`name`的所有权——但这也不是好的解决方案，rust并不鼓励直接传入像Vec或String这样的数据类型

那么传入的`&Vec<String>`类型和传出的`String`都不改变，只有修改函数体了

1. 克隆传入参数

```rust
fn stringify_name_with_title(name: &Vec<String>) -> String {
    let mut name_clone = name.clone();
    name_clone.push(String::from("Esq."));
    let full = name_clone.join(" ");
    full
}
```

通过克隆`name`，我们可以直接修改这个vector的副本 且不影响原参数

2. 使用已存在的函数

```rust
fn stringify_name_with_title(name: &Vec<String>) -> String {
    let mut full = name.join(" ");
    full.push_str(" Esq.");
    full
}
```

3. 看情况会用到的方式

```rust
// before
fn round_in_place(v: &Vec<f32>) {
    for n in v {
        *n = n.round();
    }
}
// error[E0594]: cannot assign to `*n`, which is behind a `&` reference
// --> test.rs:4:9
```

```rust
// after
fn round_in_place(v: &mut Vec<f32>) {
    for n in v {
        *n = n.round();
    }
}
```

在这个对小数取整的函数中，需要对传入的Vec一一修改，最好的方式是将参数改为可变引用 并直接用`*n`修改原数据

