---
title: "Javascript逆向学习笔记"
slug: "javascript-reverse-study-notes"
description: "学就完事了. | 更新ing"
date: 2022-12-15T23:45:40+08:00
categories: ["NOTES&SUMMARY", "LTS"]
series: ["前端安全"]
tags: ["js"]
draft: false
toc: true
---

## 绕过反调试

- 每xx ms执行一次debugger

```js
setInterval(function () {
    debugger
}, 500)
```

在debugger处右键 永不在此处暂停

- 每xx ms打印一个值 干扰控制台输出

*单一js文件形态：

```js
setInterval(function () {
    if (eval.toString() !== 'function eval() { [native code] }'){
    w();
    dd();
    while (1){
        console.error('苦蟲');
        console.error('闆蘭')
    }
}
    if (setInterval.toString() !== 'function setInterval() { [native code] }'){
        w();
        dd();
        console.error('總難煉為');
        debugger
    }
    console.error('永不言棄,jy');
}, 500);
```

控制台处定位到代码，源代码处右键 在网络面板中显示，找到对应条目右键 阻止请求URL

*嵌入js中作为函数的形态：

```js
function oo0O0(){
    window.a = 'xxxxxxxxxxxxxxxxxxxxxxxxxx' // 一堆乱码
    for (var i = 0, len = window.a.length; i < len; i++) {
        console.log(window.a[i]);
    }
}
```

在该函数处下断点，让它在被调用之前就修改掉内部的赋值

## 猿人学1 - js混淆源码乱码

> https://match.yuanrenxue.com/match/1
>
> 题目要求：抓取所有（5页）机票的价格，并计算所有机票价格的平均值，填入答案

![image-20221215175204997](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221215175204997.png)

首先是一个弹窗提示

观察请求，第一页内容中机票价格的部分来自于

```http
GET /api/match/1?m=a5884f0516dd56ce8282ddcc1ede907c%E4%B8%A81671196895 HTTP/2
Host: match.yuanrenxue.com
```

也就是`/api/match/<page>?m=<length 32>丨<length 10 timestamp(s)>`，末尾存在时间戳限制，前一个一看就像md5

如何定位m的生成？直接搜`m = `，`丨` 甚至是`/api/match`都没找到；我们在网络部分来找到它的发起程序

![image-20221215213810415](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221215213810415.png)

看着代码很复杂，全是混淆，但是在开头点了断点没几步就出结果了……

![image-20221215214022138](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221215214022138.png)

这里`_0x5d83a3`正是我们要的对象，包含m，也反向说明了加密部分在1到4行之内已经完成了，简化如下

```js
var _0x2268f9 = Date['parse'](new Date()) + 100000000  // 时间戳
    , _0x57feae = oo0O0(_0x2268f9['toString']()) + window['f']; // md5部分
const _0x5d83a3 = {};
_0x5d83a3['m'] = _0x57feae + '\u4e28' + _0x2268f9 / 1000;
```

时间戳部分很好说，md5部分有个oo0O0，打断点发现这是在html里的script中定义的

![image-20221215220246973](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221215220246973.png)

拖出来一看好家伙，一顿操作猛如虎 最后`return ''`？？？

```js
function oo0O0(mw) {
    window.b = '';
    for (var i = 0, len = window.a.length; i < len; i++) {
        console.log(window.a[i]);
        window.b += String[e + g](window.a[i][f + h]() - i - window.c)
    }
    var U = ['W5r5W6VdIHZcT8kU', 'WQ8CWRaxWQirAW=='];
    var J = function (o, E) {
        o = o - 0x0;
        var N = U[o];
        if (J['bSSGte'] === undefined) {
            var Y = function (w) {
                var m = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/=',
                    T = String(w)['replace'](/=+$/, '');
                var A = '';
                for (var C = 0x0, b, W, l = 0x0; W = T['charAt'](l++); ~W && (b = C % 0x4 ? b * 0x40 + W : W, C++ % 0x4) ? A += String['fromCharCode'](0xff & b >> (-0x2 * C & 0x6)) : 0x0) {
                    W = m['indexOf'](W)
                }
                return A
            };
            var t = function (w, m) {
                var T = [], A = 0x0, C, b = '', W = '';
                w = Y(w);
                for (var R = 0x0, v = w['length']; R < v; R++) {
                    W += '%' + ('00' + w['charCodeAt'](R)['toString'](0x10))['slice'](-0x2)
                }
                w = decodeURIComponent(W);
                var l;
                for (l = 0x0; l < 0x100; l++) {
                    T[l] = l
                }
                for (l = 0x0; l < 0x100; l++) {
                    A = (A + T[l] + m['charCodeAt'](l % m['length'])) % 0x100, C = T[l], T[l] = T[A], T[A] = C
                }
                l = 0x0, A = 0x0;
                for (var L = 0x0; L < w['length']; L++) {
                    l = (l + 0x1) % 0x100, A = (A + T[l]) % 0x100, C = T[l], T[l] = T[A], T[A] = C, b += String['fromCharCode'](w['charCodeAt'](L) ^ T[(T[l] + T[A]) % 0x100])
                }
                return b
            };
            J['luAabU'] = t, J['qlVPZg'] = {}, J['bSSGte'] = !![]
        }
        var H = J['qlVPZg'][o];
        return H === undefined ? (J['TUDBIJ'] === undefined && (J['TUDBIJ'] = !![]), N = J['luAabU'](N, E), J['qlVPZg'][o] = N) : N = H, N
    };
    eval(atob(window['b'])[J('0x0', ']dQW')](J('0x1', 'GTu!'), '\x27' + mw + '\x27'));
    return ''
}
```

不过别急，`_0x57feae`由两部分组成，前面注定为空，后面还有`window.f`的操作空间，恰好return前面还有个万恶的eval，分段看一下内容：先看`atob(window['b'])`到底是个啥，控制台`copy(window.b)`

![image-20221215222925604](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221215222925604.png)

拖出来一看有一堆加解密算法，而最后跟着一行

```js
window.f = hex_md5(mwqqppz)
```

后面的部分我们如法炮制，最后简化得到

```js
eval(atob(window['b'])['replace']('mwqqppz', '\''+ mw + '\''));
```

这下看懂了，相当于执行

```js
var mw = Date['parse'](new Date()) + 100000000
window.f = hex_md5(mw)
```

前面一堆圈的函数就可以直接抛弃了；over，写脚本开始嗦~

```js
// 2.js
// 省略atob(window['b'])的一堆函数 在末尾加上下面的getResult()

function getResult() {
    var timestamp = Date['parse'](new Date()) + 100000000  // 时间戳
    var md5 = hex_md5(timestamp['toString']()); // md5部分
    const res = {};
    res['m'] = md5 + '\u4e28' + timestamp / 1000;
    return res['m']
    // { m: 'fac30e8b1f385e2eecd11154f33aa469丨1671216252' }
}
```

```python
import subprocess
from functools import partial

subprocess.Popen = partial(subprocess.Popen, encoding='utf-8')

with open('2.js', 'r', encoding='utf-8') as f:
    import execjs
    import requests

    _url = 'https://match.yuanrenxue.com/api/match/'
    headers = {'User-Agent': 'yuanrenxue.project'}
    ctx = execjs.compile(f.read())
    sum = 0
    num = 0

    for i in range(1, 6):
        res = ctx.call('getResult')
        value = requests.get(f'{_url}1?page={i}&m={res}', headers=headers).json()
        for j in value['data']:
            sum += j['value']
            num += 1

    print(sum / num)
	# 4700.0
```

