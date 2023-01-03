---
title: "Javascript逆向学习笔记"
slug: "javascript-reverse-study-notes"
description: "学就完事了. | 更新ing"
date: 2023-01-03T22:53:09+08:00
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

在debugger处右键 永不在此处暂停（或：添加条件断点 false）

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

## obfuscator特征&解混淆

> ob混淆：[JavaScript Obfuscator Tool](https://obfuscator.io/)
>
> 解混淆：[ob混淆还原工具](https://github.com/DingZaiHub/ob-decrypt)  |  [ob混淆专解 测试版 V0.6](http://tool.yuanrenxue.com/decode_obfuscator)

以基础代码举例

```js
function hi() {
  console.log("Hello World!");
}
hi();
```

旧版obfuscator加密后的结果是这样的：

```js
var _0x30bb = ['log', 'Hello\x20World!'];	// 定义数组 [1]

(function (_0x38d89d, _0x30bbb2) {
    var _0xae0a32 = function (_0x2e4e9d) {
        while (--_0x2e4e9d) {
            _0x38d89d['push'](_0x38d89d['shift']());	// 对数组内变量做移位操作 [2]
        }
    };
    _0xae0a32(++_0x30bbb2);
}(_0x30bb, 0x153));

var _0xae0a = function (_0x38d89d, _0x30bbb2) {	// 解密函数 [3]
    _0x38d89d = _0x38d89d - 0x0;
    var _0xae0a32 = _0x30bb[_0x38d89d];
    return _0xae0a32;
};

function hi() {
    console[_0xae0a('0x1')](_0xae0a('0x0'));	// 加密后的函数 [4]
}

hi();	// 调用
```

对于[2]可以直接执行，获得正确顺序下的数组内容

![image-20221216102737795](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221216102737795.png)

正确数组有了，[3]中还提供了现成的解密方法，我们只需要用它就可以还原数据了

```python
import re

import execjs

decrypt = '''var _0x30bb = ['Hello World!', 'log']  // 处理过移位的数组
var _0xae0a = function (_0x38d89d, _0x30bbb2) {	// 解密函数
    _0x38d89d = _0x38d89d - 0x0;
    var _0xae0a32 = _0x30bb[_0x38d89d];
    return _0xae0a32;
};'''
decrypt_func = '_0xae0a'
ctx = execjs.compile(decrypt)

with open('a.js', 'r', encoding='utf-8') as f:
    code = f.read()
    res = code

for i in set(re.findall(decrypt_func + '\([\s\S]+?\)', code)):
    args = re.findall('\(([\s\S]+?)\)', i)[0]
    arg1 = eval(args.split(',')[0])
    _res = ctx.call(decrypt_func, arg1)
    res = res.replace(i, "'" + _res + "'")

print(res)
```

脚本其实是完成了批量匹配和解密的过程

而目前4.0.0版obfuscator的默认加密已经没有上面可读性那么好了（看github上说不会再更新了）

```js
function _0x3b94() {    // 定义数组 [1]
    var _0x47a143 = ['log', '1914804MUXMLC', 'Hello\x20World!', '13668568PxihTU', '3072916cTlxVO', '1319280YhFVKJ', '20xdoBic', '374500AdEIEJ', '565449UDQpof', '1069183kuRgHD'];
    _0x3b94 = function () {
        return _0x47a143;
    };
    return _0x3b94();	// 从原先的赋值变量改为函数加载 并进行嵌套 同时增加了数组内的干扰项 排列顺序与之后解密有关系
}

(function (_0x41ac74, _0x1a5714) {
    var _0x59b7d7 = _0x1e16, _0x75200a = _0x41ac74();
    while (!![]) {	// 对原先流程复杂化
        try {
            var _0x28ad51 = -parseInt(_0x59b7d7(0x122)) / 0x1 + -parseInt(_0x59b7d7(0x120)) / 0x2 + parseInt(_0x59b7d7(0x121)) / 0x3 + -parseInt(_0x59b7d7(0x124)) / 0x4 + parseInt(_0x59b7d7(0x11f)) / 0x5 * (parseInt(_0x59b7d7(0x11e)) / 0x6) + -parseInt(_0x59b7d7(0x127)) / 0x7 + parseInt(_0x59b7d7(0x126)) / 0x8;
            if (_0x28ad51 === _0x1a5714) break; else _0x75200a['push'](_0x75200a['shift']());   // 对数组内变量做移位操作 [2]
        } catch (_0x4accb5) {
            _0x75200a['push'](_0x75200a['shift']());
        }
    }
}(_0x3b94, 0x93154));

function _0x1e16(_0xf3f526, _0x3900a2) {    // 解密函数 [3]
    var _0x3b9493 = _0x3b94();
    return _0x1e16 = function (_0x1e16b4, _0x52b1ed) {	// 仍旧是嵌套 核心执行部分在这里
        _0x1e16b4 = _0x1e16b4 - 0x11e;
        var _0x17070a = _0x3b9493[_0x1e16b4];
        return _0x17070a;
    }, _0x1e16(_0xf3f526, _0x3900a2);
}

function hi() {
    var _0x2ffbfb = _0x1e16;	// 再次替换变量名
    console[_0x2ffbfb(0x123)](_0x2ffbfb(0x125));    // 加密后的函数 [4]
}

hi();
```

但仔细看每一个存在差异的地方就能发现改动是有限的，核心逻辑上并没有太大区别，修改后的脚本（也仅有两处需要改）：

```python
import re

import execjs

decrypt = '''[1] + [2] + [3]'''
decrypt_func = '*见下注'
ctx = execjs.compile(decrypt)

with open('a.js', 'r', encoding='utf-8') as f:
    code = f.read()
    res = code

for i in set(re.findall(decrypt_func + '\([\s\S]+?\)', code)):
    args = re.findall('\(([\s\S]+?)\)', i)[0]
    arg1 = eval(args.split(',')[0])
    _res = ctx.call(decrypt_func, arg1)
    res = res.replace(i, "'" + _res + "'")

print(res)
```

注：由于最后解密的部分又重新替换了个名字，所以要么在js处进行替换 要么在我们解密脚本中进行替换，结果是一样的

当然，这里展示的只是一个非常简单的单一函数Ob混淆后再解混淆的实例，实际情况要更复杂更麻烦，建议直接用工具嗦~

## 添加hook

### 油猴

一个简单的在cookie生成处下断点的例子

```js
(function () {
  'use strict';
  Object.defineProperty(document,'cookie',{
    set:function(val){
        debugger;
        return val;
    }
});})();
```

油猴前面的注释有其特殊的含义，主要注意`@match`，它确定了这段代码的作用域 支持正则

### 浏览器插件



## 对js进行ast/json转换

- js2ast: https://astexplorer.net/
- js2json

```js
// js2json.js
// cmd: node js2json <jsfile> <outjsonfile>
const fs = require('fs')
const esprima = require('esprima')

const input_text = process.argv[2]
const output_text = process.argv[3]

const data = fs.readFileSync(input_text)
const ast = esprima.parseScript(data.toString())
const ast_to_json = JSON.stringify(ast)

fs.writeFileSync(output_text, ast_to_json)
```

- json2js

```js
// json2js
// cmd: node json2js <jsonfile> <outjsfile>
const fs = require('fs')
const escodegen = require('escodegen')

const input_text = process.argv[2]
const output_text = process.argv[3]

const data = fs.readFileSync(input_text)
const ast = JSON.parse(data.toString())
const code = escodegen.generate(ast, {
    format: {
        compact: true,
        escapeless: true
    }
})

fs.writeFileSync(output_text, code)
```

## 切分js

- 转AST进行操作

- 可以先对js转json，再操作json

举例：

```python
import json
import os

os.system('node js2json 1.js 1.json')

with open('1.json', 'r', encoding='utf-8') as f:
    node = json.loads(f.read())

left_node = {
    'type': 'Program',
    'body': node['body'][:3],
    'sourceType': 'script'
}

right_node = {
    'type': 'Program',
    'body': node['body'][3:],
    'sourceType': 'script'
}

with open('1_left.json', 'w', encoding='utf-8') as f1, open('1_right.json', 'w', encoding='utf-8') as f2:
    f1.write(json.dumps(left_node))
    f2.write(json.dumps(right_node))

os.system('node json2js 1_left.json 1_left.js')
os.system('node json2js 1_right.json 1_right.js')
```

## execjs库相关问题

如果遇到编码相关问题，请修改subprocess.py中`__init__`方法中的encoding参数为'utf-8'

！！！但之后记得改回来！以免影响到其他库的正常运行

## requests库tricks

### 确保headers顺序不变

用`requests.Session()`对象，对这个session对象设置headers，还能减少后续继续赋值cookie的过程

### cookie转字典

```python
cookie_dict = requests.utils.dict_from_cookiejar(response.cookies)
```

## 猿人学1 - js混淆 源码乱码

> https://match.yuanrenxue.com/match/1
>
> 题目要求：抓取所有（5页）机票的价格，并计算所有机票价格的平均值，填入答案

![image-20221215175204997](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221215175204997.png)

~~首先把人力计算的路堵死了（~~

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
    console.log(res)
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

## 猿人学2 - js 混淆 动态cookie1

> https://match.yuanrenxue.com/match/2
>
> 题目要求：提取全部5页发布日热度的值，计算所有值的**加和**,并提交答案 (感谢蔡老板为本题提供混淆方案)

惯例抓包，这次信息在`/api/match/2?page=<page_number>`获取，并且请求的cookie有时效，在f12网络部分可以看出这是个ajax请求；还有个包是`/match/2`，诶 正好有巨量的混淆过的js代码，但我在网络部分并没有找到

![image-20221216101556644](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221216101556644.png)

*这里也可以借助hook的方式来打断点

![image-20221216155159401](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221216155159401.png)

或者直接不要在以往会never pause的地方过掉，也能在堆栈处停下

![image-20221216155631382](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221216155631382.png)

断下来之后找到cookie生成的地方（搜document）

![image-20221216160528185](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221216160528185.png)

把等于号后面的部分执行一下，可以确定这里就是cookie中m的生成处了

但是好像并没有什么卵用……因为我们还是搞不懂document['cookie']具体是怎么生成的，不过一眼Ob鉴定为烂 我们拖出来用解混淆的工具解一下，结果嘿！您猜怎么着！还是看不懂！

```js
// 原
document[$dbsm_0x42c3(QO0qoO, Qolo0q) + $dbsm_0x42c3(oqQoqO, QIolqQ)] = _0x426597[$dbsm_0x42c3(QOoqQQ, OILiQO) + '\x4c\x4b'](_0x426597[$dbsm_0x42c3(LOILqL, qIIo1Q) + '\x4c\x4b'](_0x426597[$dbsm_0x42c3(iLOo0O, qqqOoQ) + '\x54\x78'](_0x426597[$dbsm_0x42c3(oqOqLO, qoO0i0) + '\x59\x45'](_0x426597[$dbsm_0x42c3(IoOqoO, qiQO1q) + '\x51\x65'](_0x426597[$dbsm_0x42c3(qOqoL1, LloilO) + '\x67\x4b'](QOO0Qo, _0x426597['\x4c\x41\x74' + '\x54\x65'](_0x3c9ca8)), iQoOQo), _0x426597[$dbsm_0x42c3(QiqQqQ, lqO110) + '\x55\x4a'](_0x313b78, _0x46f74d)), OQiQQo), _0x46f74d), _0x426597[$dbsm_0x42c3(oqlQq0, IIo0I0) + '\x43\x73']),

document[$dbsm_0x42c3(qqLQOq, iOiqII) + $dbsm_0x42c3(q1IoqQ, QQlLlq)] = _0x5500bb['\x4e\x74\x44' + '\x72\x43'](_0x5500bb[$dbsm_0x42c3(qqqQoq, oqQiiO) + '\x6d\x65'](_0x5500bb[$dbsm_0x42c3(Ioo0ql, olq0Oq) + '\x6d\x65'](_0x5500bb[$dbsm_0x42c3(qOIqQi, OOqIQi) + '\x72\x44'](_0x5500bb[$dbsm_0x42c3(Q1qoqQ, lILOOq) + '\x72\x44'](_0x5500bb[$dbsm_0x42c3(qOO1Q0, oiqlQQ) + '\x72\x44'](Ql1OO0, _0x5500bb['\x7a\x76\x67' + '\x6c\x77'](_0x3c9ca8)), Qoqq0I), _0x5500bb[$dbsm_0x42c3(iqOiQ0, QOiq0Q) + '\x47\x6b'](_0x313b78, _0x160e3a)), lOo0QQ), _0x160e3a), _0x5500bb[$dbsm_0x42c3(qiOOiO, liQIoQ) + '\x4e\x5a']),
```

```js
// 解混淆后
document[$dbsm_0x42c3(QO0qoO, Qolo0q) + $dbsm_0x42c3(oqQoqO, QIolqQ)] = _0x426597[$dbsm_0x42c3(QOoqQQ, OILiQO) + "LK"](_0x426597[$dbsm_0x42c3(LOILqL, qIIo1Q) + "LK"](_0x426597[$dbsm_0x42c3(iLOo0O, qqqOoQ) + "Tx"](_0x426597[$dbsm_0x42c3(oqOqLO, qoO0i0) + "YE"](_0x426597[$dbsm_0x42c3(IoOqoO, qiQO1q) + "Qe"](_0x426597[$dbsm_0x42c3(qOqoL1, LloilO) + "gK"](QOO0Qo, _0x426597["LAtTe"](_0x3c9ca8)), iQoOQo), _0x426597[$dbsm_0x42c3(QiqQqQ, lqO110) + "UJ"](_0x313b78, _0x46f74d)), OQiQQo), _0x46f74d), _0x426597[$dbsm_0x42c3(oqlQq0, IIo0I0) + "Cs"]);

document[$dbsm_0x42c3(qqLQOq, iOiqII) + $dbsm_0x42c3(q1IoqQ, QQlLlq)] = _0x5500bb["NtDrC"](_0x5500bb[$dbsm_0x42c3(qqqQoq, oqQiiO) + "me"](_0x5500bb[$dbsm_0x42c3(Ioo0ql, olq0Oq) + "me"](_0x5500bb[$dbsm_0x42c3(qOIqQi, OOqIQi) + "rD"](_0x5500bb[$dbsm_0x42c3(Q1qoqQ, lILOOq) + "rD"](_0x5500bb[$dbsm_0x42c3(qOO1Q0, oiqlQQ) + "rD"](Ql1OO0, _0x5500bb["zvglw"](_0x3c9ca8)), Qoqq0I), _0x5500bb[$dbsm_0x42c3(iqOiQ0, QOiq0Q) + "Gk"](_0x313b78, _0x160e3a)), lOo0QQ), _0x160e3a), _0x5500bb[$dbsm_0x42c3(qiOOiO, liQIoQ) + "NZ"]);
```

这下呃呃了，只能手搓了😋

### 简单处理

观察js，全部折叠以后会发现有6个部分，我们把js切分成前后两个部分并写入right.js（前三）和left.js（后三）中，便于后续处理

![image-20221217225948100](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221217225948100.png)

```js
var $dbsm_0x123c = [....]	// 乱序数组 [1]
(function(....){})($dbsm_0x123c, 0x70)	// 还原数组 [2]
var $dbsm_0x42c3 = function(....){}	// 解密函数 [3]
(function $dbsm_0x37d29a(....))()	// 有用的函数？ [4]
function $dbsm_0x1a0b2e(....){}	// 有用的函数？ [5]
setInterval(function(....){}, 0xfa0)	// ? [6]
```

### 变量回复

一般情况下调用解密函数`$dbsm_0x42c31`时传入的参数直接就是字符串，但我们可以看到[4]中调用[3]时会提前把变量赋值，再传入

```js
// 通常
res = func('abcd')
// 本题
content = 'abcd'
res2 = func(content)
```

这样做一两处还好，太多的话我们肯定需要批量还原；那如何还原呢？我们借助AST，这里赋值的部分都是一个个AssignmentExpression节点

替换变量分为这几步

- 定位这些节点所在的列表容器
- 提取变量名与字符串的映射关系 存入字典
- 替换变量为字符串
- 删除原变量

我们要恢复的是[4]到[6]中的变量赋值，首先要从AST中找到它们：

- 4：`node['expression']['callee']['body']['body'][0]['expression']['expressions']`

![image-20230103104153723](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103104153723.png)

- 5: `node['body']['body'][0]['expression']['expressions']`

![image-20230103104616646](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103104616646.png)

- 6: `node['expression']['arguments'][0]['body']['body'][0]['expression']['expressions']`

![image-20230103105001086](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103105001086.png)

通过这样的方式我们就可以找到4~6中所有的含`=`的赋值表达式，根据等号右边的类型还可以细分这样两种：

- `expr['left']['type'] == 'Identifier'`

即`a=1`，我们直接把它加入全局字典即可

- `expr['right']['type'] == 'BinaryExpression'`

即`a=1+1`，这种类型的需要我们再考虑right的二元表达式是什么样的，甚至需要继续迭代处理

代码如下：

```python
import copy
import json
import os
import sys

val_dict = {}


def getExpr(node):
    val_dict.clear()
    try:
        # 处理[4]
        expr_list = node['expression']['callee']['body']['body'][0]['expression']['expressions']
    except TypeError:
        # 处理[5]
        expr_list = node['body']['body'][0]['expression']['expressions']
    except KeyError:
        # 处理[6]
        expr_list = node['expression']['arguments'][0]['body']['body'][0]['expression']['expressions']
    return expr_list


def getMapping(node):
    # [4]到[6]变量名称有重复 分开处理
    expr_list = getExpr(node)

    for expr in expr_list:
        if expr['type'] != 'AssignmentExpression':
            continue
        if expr['left']['type'] == 'Identifier':
            if expr['right']['type'] == 'Literal':
                val_dict[expr['left']['name']] = expr['right']
            elif expr['right']['type'] == 'BinaryExpression':
                left = val_dict[expr['right']['left']['name']]
                right = expr['right']['right']
                try:
                    new_node = copy.deepcopy(right)
                    new_node['value'] = eval(f'{left["value"]} {expr["right"]["operator"]} {right["value"]}')
                    val_dict[expr['left']['name']] = new_node
                except TypeError:
                    print(expr['right'])
                    sys.exit()


def varReload(node):
    # 一次只还原一个部分 递归处理
    if type(node) == list:
        for i in node:  # 虽然arguments本身是列表 但会在下面再被捕捉 不在此处进入
            varReload(i)
        return
    elif type(node) != dict:
        return

    try:    # 捕获一个包含arguments属性的节点
        for i in range(len(node['arguments'])): # 遍历参数
            if node['arguments'][i]['type'] == 'Identifier':
                try:  # 捕获前面参数不匹配 而后面参数匹配的情况 确保每个参数都会被处理
                    node['arguments'][i] = val_dict[node['arguments'][i]['name']]
                except KeyError:
                    pass
        else:
            raise KeyError
    except KeyError:    # 无论结果如何 将该节点的所有属性再次递归 防止遗漏
        for key in node.keys():
            varReload(node[key])
        return


def delNode(node):
    expr_list = getExpr(node)
    for i in range(len(expr_list) - 1, -1, -1):
        if expr_list[i]['type'] != 'AssignmentExpression':
            continue
        if expr_list[i]['left']['type'] == 'Identifier' and expr_list[i]['right']['type'] in (
                'Literal', 'BinaryExpression'):
            del expr_list[i]


if __name__ == '__main__':
    with open('1_right.json', 'r', encoding='utf8') as f:
        data = json.loads(f.read())
    for item in data['body']:
        getMapping(item)
        varReload(item)
        delNode(item)
    with open('1_left_var_reload.json', 'w', encoding='utf8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_var_reload.json 1_left_var_reload.js')

```

最终实现这样的效果：

![image-20221218011540192](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218011540192.png)

### 调用解密函数

我们已经成功把[4]到[6]中调用解密函数的部分都把变量换成了字符串（参数还原），接下来我们把用到解密函数的部分进行还原，还原的方式是找到解密函数`$dbsm_0x42c3`被调用的位置，获取参数，调用函数，替换返回值

在ast中观察，函数调用节点是CallExpression，内部callee的name是`$dbsm_0x42c3`

![image-20230103140231617](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103140231617.png)

编写对应代码

```python
import json
import os

import execjs


def funcReload(node, ctx):
    if type(node) == list:
        for i in node:
            funcReload(i, ctx)
        return
    elif type(node) != dict:
        return
    if node['type'] == 'CallExpression':    # 捕获一个CallExpression节点
        try:
            if node['callee']['name'] == "$dbsm_0x42c3":    # 确认函数名
                arg_list = []
                for i in node['arguments']: # 提取所有参数
                    if i['type'] != 'Literal':
                        break
                    arg_list.append(i['value'])
                else:
                    value = ctx.call('$dbsm_0x42c3', *arg_list)
                    print(value)
                    new_node = {'type': 'Literal', 'value': value}
                    node.clear()
                    node.update(new_node)
                    return
        except KeyError:
            pass
    for key in node.keys():
        funcReload(node[key], ctx)


if __name__ == '__main__':
    with open('1_left_var_reload.json', 'r', encoding='utf-8') as f:
        data = json.loads(f.read())
    with open('1_left.js', 'r', encoding='utf-8') as f:
        ctx = execjs.compile(f.read())
    funcReload(data, ctx)
    with open('1_left_call_reload.json', 'w', encoding='utf-8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_call_reload.json 1_left_call_reload.js')

```

有个蜜汁彩蛋？由于这里乱码部分心机的加了中文字符，导致即使open处用utf-8还是会导致execjs解析失败，所以要修改subprocess.py中的encoding参数也为utf-8（切记 用完之后改回来 不然会影响其它库的正常工作的）

执行完的效果则是这样

![image-20221218021235668](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218021235668.png)

在新生成的js中已经找不到`$dbsm_0x42c3`的身影了

### 对象调用还原

#### 字符串拼接

字符串拼接部分的节点类型为BinaryExpression，且左右子节点都是Literal

```python
import json
import os


def concatString(node):
    if type(node) == list:
        for i in node:
            concatString(i)
        return
    elif type(node) != dict:
        return
    if node['type'] == 'BinaryExpression':
        if not (node['left']['type'] == 'Literal' and node['right']['type'] == 'Literal'):  # 当`+`左右任意一方为非Literal类型 则递归处理
            concatString(node['left'])
            concatString(node['right'])
        if node['left']['type'] == 'Literal' and node['right']['type'] == 'Literal':
            new_node = {'type': 'Literal', 'value': node['left']['value'] + node['right']['value']}
            node.clear()
            node.update(new_node)
            return
    for key in node.keys():
        concatString(node[key])


if __name__ == '__main__':
    with open('1_left_call_reload.json', 'r', encoding='utf-8') as f:
        data = json.loads(f.read())
    concatString(data)
    with open('1_left_concat_string.json', 'w', encoding='utf-8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_concat_string.json 1_left_concat_string.js')
```

![image-20221218022613304](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218022613304.png)

#### 获取对象的属性字典

观察恢复过字符串的js，我们可以发现函数中通常是这样操作的：

![image-20221218151522895](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218151522895.png)

![image-20221218152556874](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218152556874.png)

先定义一个对象，再不断对其添加属性，难搞的是这些属性有的是字符串 有的是函数定义 有的又交叉其它对象的属性，之后再把这个对象赋值给另一个对象，再通过属性名进行引用，中间可能会再间接赋值给一个对象

我们接下来要做的是将这些属性被调用的地方，替换为对应的值/内容（类似之前的变量回复和解密函数回复），我们希望构造一个字典存储这样的映射关系

```python
property_dict = {
	'obj1_name': {
		'property1_name': 'value1',
		'property2_name': 'value2',
		....
	},
	....
}
```

接下来惯例ast中分析结构，最开始属性赋值的长长的部分都属于SequenceExpression，具体赋值的语句属于AssignmentExpression，且左边的类型是Identifier

![image-20221218153009563](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218153009563.png)

编写对应代码：

```python
import json
import os
import copy

property_dict = {}
object_list = []


def getPropertyMapping(node):
    if type(node) == list:
        for i in node:
            getPropertyMapping(i)
        return
    elif type(node) != dict:
        return

    if node['type'] == 'SequenceExpression':  # 捕获一个表达式序列
        for i in range(- len(node['expressions']), 0, 1):  # 一边正向遍历列表 一边删除 不产生KeyError
            expr = node['expressions'][i]
            if expr['type'] == 'AssignmentExpression':  # 捕获一个变量声明
                if expr['left']['type'] == 'Identifier':
                    if expr['right']['type'] == 'ObjectExpression' and expr['right']['properties'] == []:  # 声明的是一个空对象
                        property_dict[expr['left']['name']] = {}
                        object_list.append(expr['left']['name'])
                        del node['expressions'][i]
                        continue
                    elif expr['right']['type'] == 'Identifier':  # 一个对象 = 另一个对象
                        if expr['right']['name'] in property_dict:
                            property_dict[expr['left']['name']] = property_dict[expr['right']['name']]
                            object_list.append(expr['left']['name'])
                            del node['expressions'][i]
                            continue
                elif expr['left']['type'] == 'MemberExpression':  # 属性声明
                    _object = expr['left']['object']['name']
                    try:
                        _property_name = expr['left']['property']['value']
                        property_dict[_object][_property_name] = expr['right']
                        del node['expressions'][i]
                        continue
                    except KeyError:
                        pass
            for key in expr.keys():  # 将处理完的子节点的所有属性加入递归 避免遗漏
                getPropertyMapping(expr[key])
    for key in node.keys():
        getPropertyMapping(node[key])


if __name__ == '__main__':
    with open('1_left_concat_string.json', 'r', encoding='utf-8') as f:
        data = json.loads(f.read())
    getPropertyMapping(data)
    with open('1_left_property_delete.json', 'w', encoding='utf-8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_property_delete.json 1_left_property_delete.js')
```

这一步我们生成了1_left_property_delete.json和1_left_property_delete.js中间文件，其中删除了前面提到的对象赋值的部分

#### 对象调用还原

我们获得了对象->属性->值的映射字典后，需要查找调用这些属性的地方进行还原（上一步我们都做了删除处理）

对象调用有两种方式，一种是函数式调用CallExpression类型的节点，一种是MemberExpression类型 直接返回字符串，而函数式调用的返回值有多种类型，需要进行不同的处理：

1. 返回字符串Literal

```js
return 'a'
```

2. 返回对象调用MemberExpression

```js
return obj['a'](c, d)
```

3. 返回二元表达式BinaryExpression

```js
return a*b
```

4. 返回逻辑计算LogicalExpression

```js
return a||b
```

5. 返回函数调用CallExpression

```js
return a(b)
```

对于1和2 直接替换内容即可，属于MemberExpression节点，但如果包裹在函数中需要另外考虑；对于3和4，都是由中间的符号和左右的标识组成，只要将调用时的两个参数替换 再将整个节点替换；对于5，可以同时提取形参名和实参节点，以调用对象时传参的顺序为标准 填入临时字典

编写对应代码：

```python
import copy
import json
import os

# 此处省略上一步获取属性字典的函数 执行时要加上 因为需要使用property_dict这个全局变量

def propertyReload(node):
    if type(node) == list:
        for i in node:
            propertyReload(i)
        return
    elif type(node) != dict:
        return

    if node['type'] == 'MemberExpression':  # 捕获一个属性调用节点
        try:
            _obj = node['object']['name']
            _property = node['property']['value']
            new_node = property_dict[_obj][_property]
            if new_node['type'] != 'FunctionExpression':  # 不是函数类型节点直接替换
                node.clear()
                node.update(new_node)
                propertyReload(node)
        except KeyError:
            pass
    try:
        if node['type'] != 'CallExpression' or node['callee']['type'] != 'MemberExpression':  # 函数式调用 且子节点callee类型为MemberExpression
            raise KeyError
        _obj = node['callee']['object']['name']
        _property = node['callee']['property']['value']
        func_node = property_dict[_obj][_property]
    except KeyError:
        for key in node.keys():
            propertyReload(node[key])
        return

    param_list = [i['name'] for i in func_node['params']]  # 形参
    arg_list = node['arguments']  # 实参
    param_arg_dict = dict(zip(param_list, arg_list))  # 形参与实参对应字典

    ret_node = copy.deepcopy(func_node['body']['body'][0])  # 拷贝一份函数节点的返回值节点
    if ret_node['type'] != 'ReturnStatement':
        print(f'有超过一行的函数体：{func_node}')
        exit()

    if ret_node['argument']['type'] == 'Literal' or ret_node['argument']['type'] == 'MemberExpression':  # 实参直接替换形参 对应1, 2种情况被包裹在一个函数中再返回时
        node.clear()
        node.update(ret_node['argument'])
    elif ret_node['argument']['type'] == 'BinaryExpression' or ret_node['argument']['type'] == 'LogicalExpression':  # 对应3, 4种情况
        if ret_node['argument']['left']['type'] == 'Identifier':
            ret_node['argument']['left'] = param_arg_dict[ret_node['argument']['left']['name']]
        if ret_node['argument']['right']['type'] == 'Identifier':
            ret_node['argument']['right'] = param_arg_dict[ret_node['argument']['right']['name']]
        node.clear()
        node.update(ret_node['argument'])
    elif ret_node['argument']['type'] == 'CallExpression':  # 对应第五种情况
        if ret_node['argument']['callee']['type'] != 'MemberExpression':
            func_name = ret_node['argument']['callee']['name']
            if func_name in param_arg_dict:
                ret_node['argument']['callee'] = param_arg_dict[func_name]
        for i in range(len(ret_node['argument']['arguments'])):
            if ret_node['argument']['arguments'][i]['type'] == 'Identifier':
                arg_name = ret_node['argument']['arguments'][i]['name']
                ret_node['argument']['arguments'][i] = param_arg_dict[arg_name]
        node.clear()
        node.update(ret_node['argument'])
    else:
        print(f'意料之外的函数返回值类型：{ret_node}')
        exit()

    # 递归处理防止遗漏
    propertyReload(node)
    return


if __name__ == '__main__':
    with open('1_left_property_delete.json', 'r', encoding='utf-8') as f:
        data = json.loads(f.read())

    propertyReload(data)
    with open('1_left_property_reload.json', 'w', encoding='utf-8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_property_reload.json 1_left_property_reload.js')
```

最终效果

![image-20221219021808232](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221219021808232.png)

### 分支流程判断

上一步生成的js文件中有很多多余的分支判断，比如上图中的`if('QcUHF' !== 'QcUHF')`，这样的bool判断是可以提前处理掉的，我们只需要递归找到这样的节点，判断其表达式中的bool值并决定是否保留此分支

代码如下：

```python
import json
import os
import sys

operator_dict = {'===': '==', '!==': '!='}


def ifReload(node):
    if type(node) == list:
        for i in node:
            ifReload(i)
        return
    elif type(node) != dict:
        return

    if node['type'] == 'IfStatement':  # 捕获一个分支语句
        if node['test']['type'] == 'BinaryExpression':  # 当判断内容为二元表达式时
            if node['test']['left']['type'] == 'Literal' and node['test']['right']['type'] == 'Literal':  # 当表达式两侧均为Literal类型时
                consequent = node['consequent']  # 当前分支下if的block
                alternate = node['alternate']  # 当前分支下else的block
                try:  # 执行表达式 根据执行结果保留block
                    if eval(f"'{node['test']['left']['value']}' {operator_dict[node['test']['operator']]} '{node['test']['right']['value']}'"):
                        node.clear()
                        node.update(consequent)
                    else:
                        node.clear()
                        node.update(alternate)
                except KeyError:
                    print(f'Unexcepted operator: {node}')
                    sys.exit()
                else:
                    ifReload(node)

    for key in node.keys():
        ifReload(node[key])


if __name__ == '__main__':
    with open('1_left_property_reload.json', 'r', encoding='utf8') as f:
        data = json.loads(f.read())
    ifReload(data)

    with open('1_left_if_reload.json', 'w', encoding='utf8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_if_reload.json 1_left_if_reload.js')

```

执行结果：

![image-20230103164701409](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103164701409.png)

可以看到我们已经将不需要的分支去除掉了

### 控制流平坦化

虽然if被我们砍去了大半，但while+switch的组合还是留下了很多障碍

![image-20230103165127573](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103165127573.png)

如图，本来从上至下的执行顺序被人为控制，我们需要进行控制流平坦化处理来还原程序执行的真实顺序

![image-20230103165544743](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103165544743.png)

执行顺序在ExpressionStatement中，我们按执行顺序提取出cases对应的block，再替换整个node

代码如下：

```python
import json
import os
import sys



def flowFlatten(node):
    if type(node) == list:
        flag = False
        try:
            if len(node) == 2:
                if node[1]['type'] == 'WhileStatement':
                    var1 = node[0]['expression']['expressions'][0]
                    var2 = node[0]['expression']['expressions'][1]
                    if var1['right']['type'] == 'CallExpression' and var2['right']['type'] == 'Literal':
                        flag = True
            if not flag:
                raise KeyError
        except (KeyError, TypeError):
            for i in node:
                flowFlatten(i)
            return
    elif type(node) == dict:
        for key in node.keys():
            flowFlatten(node[key])
        return
    else:
        return

    flow = var1['right']['callee']['object']['value'].split('|')
    cases = node[1]['body']['body'][0]['cases']
    results = [cases[int(i)]['consequent'][0] for i in flow]

    node.clear()
    node.extend(results)
    for i in node:
        flowFlatten(i)


if __name__ == '__main__':
    with open('1_left_if_reload.json', 'r', encoding='utf8') as f:
        data = json.loads(f.read())
    flowFlatten(data)

    with open('1_left_flow_flattening.json', 'w', encoding='utf8') as f:
        f.write(json.dumps(data))
    os.system('node json2js 1_left_flow_flattening.json 1_left_flow_flattening.js')

```

实现效果：

![image-20230103170512666](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103170512666.png)

### 去除干扰项

此时程序已经比较方便阅读，能直接找到设置cookie的接口

![image-20230103170714330](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103170714330.png)

但还是无法直接运行这个js文件，因为有最后那个setInterval函数的干扰，同时还有混杂的干扰项

![image-20230103170857905](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103170857905.png)

以及第四部分`dbsm_0x37d29a`的若干参数，都是无用的，我们手动去除一下

另外`_0x3c9ca8`函数（以及众多函数）也都属于多余的干扰项，我们手动删去调用它的地方

最终得到的js:

```js
(function $dbsm_0x37d29a() {

    function _0x112208(_0x5b69d8, _0x3de4a1) {
        {
            _0x448c2f = (65535 & _0x5b69d8) + (65535 & _0x3de4a1);
            return (_0x5b69d8 >> 16) + (_0x3de4a1 >> 16) + (_0x448c2f >> 16) << 16 | 65535 & _0x448c2f;
        }
    }

    function _0x101700(_0x19c5f2, _0x40c04f) {
        {
            return _0x19c5f2 << _0x40c04f | _0x19c5f2 >>> 32 - _0x40c04f;
        }
    }

    function _0x4d9052(_0x2ad611, _0x12667c, _0x4e5444, _0x21c32c, _0x2ca7da, _0x44626f) {

        {
            return _0x112208(_0x101700(_0x112208(_0x112208(_0x12667c, _0x2ad611), _0x112208(_0x21c32c, _0x44626f)), _0x2ca7da), _0x4e5444);
        }
    }

    function _0x5624ba(_0x173d50, _0x1eb601, _0x3e80e6, _0x27ae79, _0x196272, _0x352dd6, _0x315a43) {
        {
            return _0x4d9052(_0x1eb601 & _0x3e80e6 | ~_0x1eb601 & _0x27ae79, _0x173d50, _0x1eb601, _0x196272, _0x352dd6, _0x315a43);
        }
    }

    function _0x2d8b1d(_0x32a9d0, _0x585bb5, _0x19b9f2, _0x53bbfb, _0x1cbfed, _0x34200c, _0x5135ca) {

        {
            return _0x4d9052(_0x585bb5 & _0x53bbfb | _0x19b9f2 & ~_0x53bbfb, _0x32a9d0, _0x585bb5, _0x1cbfed, _0x34200c, _0x5135ca);
        }
    }

    function _0x21cf21(_0x5f0db4, _0x560b61) {

        {
            _0x45ae5c = [99, 111, 110, 115, 111, 108, 101], _0x7cdad8 = '';
            for (_0x5d58e6 = 0; _0x5d58e6 < _0x45ae5c['length']; _0x5d58e6++) {
                {
                    _0x7cdad8 += String['fromCharCode'](_0x45ae5c[_0x5d58e6]);
                }
            }
            return _0x7cdad8;
        }
    }

    function _0x3316ae(_0x5c1f3b, _0xdee360, _0x251700, _0x2a047e, _0x4ea0af, _0x62d9e8, _0x1edd4c) {

        {
            return _0x4d9052(_0xdee360 ^ _0x251700 ^ _0x2a047e, _0x5c1f3b, _0xdee360, _0x4ea0af, _0x62d9e8, _0x1edd4c);
        }
    }

    function _0x160619(_0x2afda5, _0x4cf1da, _0x354d4e, _0x2c2702, _0x4b938d, _0x58d9fb, _0x5b82c0) {
        {
            return _0x4d9052(_0x354d4e ^ (_0x4cf1da | ~_0x2c2702), _0x2afda5, _0x4cf1da, _0x4b938d, _0x58d9fb, _0x5b82c0);
        }
    }

    function _0x1a8c0e(_0x4b49f3, _0x31923d, _0xbd3204, _0x693550, _0x540797, _0x5dacc8, _0x22f03d, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4) {

        {
            _0x4b49f3[_0x31923d >> 5] |= 128 << _0x31923d % 32, _0x4b49f3[14 + (_0x31923d + 64 >>> 9 << 4)] = _0x31923d;
            _0x5b3e7f = 1732584193, _0x2ee10b = -271733879, _0x30b068 = -1732584194, _0x3a35a4 = _0x5b3e7f - 1460850315;

            for (_0xbd3204 = 0; _0xbd3204 < _0x4b49f3['length']; _0xbd3204 += 16) _0x693550 = _0x5b3e7f, _0x540797 = _0x2ee10b, _0x5dacc8 = _0x30b068, _0x22f03d = _0x3a35a4, _0x5b3e7f = _0x5624ba(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204], 7, -680876936), _0x3a35a4 = _0x5624ba(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 1], 12, -389564586), _0x30b068 = _0x5624ba(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 2], 17, 606105819), _0x2ee10b = _0x5624ba(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 3], 22, -1044525330), _0x5b3e7f = _0x5624ba(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 4], 7, -176418897), _0x3a35a4 = _0x5624ba(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 5], 12, 1200080426), _0x30b068 = _0x5624ba(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 6], 17, -1473231341), _0x2ee10b = _0x5624ba(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 7], 22, -45705983), _0x5b3e7f = _0x5624ba(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 8], 7, 1770010416), _0x3a35a4 = _0x5624ba(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 9], 12, -1958414417), _0x30b068 = _0x5624ba(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 10], 17, -42063), _0x2ee10b = _0x5624ba(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 11], 22, -1990404162), _0x5b3e7f = _0x5624ba(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 12], 7, 1804603682), _0x3a35a4 = _0x5624ba(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 13], 12, -40341101), _0x30b068 = _0x5624ba(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 14], 17, -1502882290), _0x2ee10b = _0x5624ba(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 15], 22, 1236535329), _0x5b3e7f = _0x2d8b1d(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 1], 5, -165796510), _0x3a35a4 = _0x2d8b1d(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 6], 9, -1069501632), _0x30b068 = _0x2d8b1d(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 11], 14, 643717713), _0x2ee10b = _0x2d8b1d(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204], 20, -373897302), _0x5b3e7f = _0x2d8b1d(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 5], 5, -701558691), _0x3a35a4 = _0x2d8b1d(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 10], 9, 38016083), _0x30b068 = _0x2d8b1d(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 15], 14, -660478335), _0x2ee10b = _0x2d8b1d(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 4], 20, -405537848), _0x5b3e7f = _0x2d8b1d(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 9], 5, 568446438), _0x3a35a4 = _0x2d8b1d(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 14], 9, -1019803690), _0x30b068 = _0x2d8b1d(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 3], 14, -187363961), _0x2ee10b = _0x2d8b1d(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 8], 20, 1163531501), _0x5b3e7f = _0x2d8b1d(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 13], 5, -1444681467), _0x3a35a4 = _0x2d8b1d(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 2], 9, -51403784), _0x30b068 = _0x2d8b1d(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 7], 14, 1735328473), _0x2ee10b = _0x2d8b1d(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 12], 20, -1926607734), _0x5b3e7f = _0x3316ae(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 5], 4, -378558), _0x3a35a4 = _0x3316ae(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 8], 11, -2022574463), _0x30b068 = _0x3316ae(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 11], 16, 1839030562), _0x2ee10b = _0x3316ae(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 14], 23, -35309556), _0x5b3e7f = _0x3316ae(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 1], 4, -1530992060), _0x3a35a4 = _0x3316ae(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 4], 11, 1272893353), _0x30b068 = _0x3316ae(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 7], 16, -155497632), _0x2ee10b = _0x3316ae(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 10], 23, -1094730640), _0x5b3e7f = _0x3316ae(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 13], 4, 681279174), _0x3a35a4 = _0x3316ae(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204], 11, -358537222), _0x30b068 = _0x3316ae(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 3], 16, -722521979), _0x2ee10b = _0x3316ae(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 6], 23, 76029189), _0x5b3e7f = _0x3316ae(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 9], 4, -640364487), _0x3a35a4 = _0x3316ae(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 12], 11, -421815835), _0x30b068 = _0x3316ae(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 15], 16, 530742520), _0x2ee10b = _0x3316ae(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 2], 23, -995338651), _0x5b3e7f = _0x160619(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204], 6, -198630844), _0x3a35a4 = _0x160619(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 7], 10, 1126891415), _0x30b068 = _0x160619(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 14], 15, -1416354905), _0x2ee10b = _0x160619(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 5], 21, -57434055), _0x5b3e7f = _0x160619(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 12], 6, 1700485571), _0x3a35a4 = _0x160619(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 3], 10, -1894986606), _0x30b068 = _0x160619(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 10], 15, -1051523), _0x2ee10b = _0x160619(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 1], 21, -2054922799), _0x5b3e7f = _0x160619(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 8], 6, 1873313359), _0x3a35a4 = _0x160619(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 15], 10, -30611744), _0x30b068 = _0x160619(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 6], 15, -1560198380), _0x2ee10b = _0x160619(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 13], 21, 1309151649), _0x5b3e7f = _0x160619(_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4, _0x4b49f3[_0xbd3204 + 4], 6, -145523070), _0x3a35a4 = _0x160619(_0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x30b068, _0x4b49f3[_0xbd3204 + 11], 10, -1120210379), _0x30b068 = _0x160619(_0x30b068, _0x3a35a4, _0x5b3e7f, _0x2ee10b, _0x4b49f3[_0xbd3204 + 2], 15, 718787259), _0x2ee10b = _0x160619(_0x2ee10b, _0x30b068, _0x3a35a4, _0x5b3e7f, _0x4b49f3[_0xbd3204 + 9], 21, -343485441), _0x5b3e7f = _0x112208(_0x5b3e7f, _0x693550), _0x2ee10b = _0x112208(_0x2ee10b, _0x540797), _0x30b068 = _0x112208(_0x30b068, _0x5dacc8), _0x3a35a4 = _0x112208(_0x3a35a4, _0x22f03d);
            return [_0x5b3e7f, _0x2ee10b, _0x30b068, _0x3a35a4];
        }
    }

    function _0xb8fd83(_0x28b0d4) {

        {
            _0x18d4aa = '', _0x630f0 = 32 * _0x28b0d4['length'];
            for (_0x24362f = 0; _0x24362f < _0x630f0; _0x24362f += 8) _0x18d4aa += String['fromCharCode'](_0x28b0d4[_0x24362f >> 5] >>> _0x24362f % 32 & 255);
            return _0x18d4aa;
        }
    }

    function _0x44ecf2(_0x12f7d8) {

        {
            var _0x4a27a3 = [];
            for (_0x4a27a3[(_0x12f7d8['length'] >> 2) - 1] = void 0, _0x4d24a7 = 0; _0x4d24a7 < _0x4a27a3['length']; _0x4d24a7 += 1) _0x4a27a3[_0x4d24a7] = 0;
            var _0x4fa8f0 = 8 * _0x12f7d8['length'];
            for (_0x4d24a7 = 0; _0x4d24a7 < _0x4fa8f0; _0x4d24a7 += 8) _0x4a27a3[_0x4d24a7 >> 5] |= (255 & _0x12f7d8['charCodeAt'](_0x4d24a7 / 8)) << _0x4d24a7 % 32;
            return _0x4a27a3;
        }
    }

    function _0x57fdd5(_0x2ace3b) {
        {
            return _0xb8fd83(_0x1a8c0e(_0x44ecf2(_0x2ace3b), 8 * _0x2ace3b['length']));
        }
    }

    function _0x3781b2(_0x5802aa, _0x324521, _0x33c9ff, _0x5d4f74, _0x344078, _0x385415, _0x160dd3, _0x61f2ad, _0x5a2d55, _0x47bfff) {
        {
            _0x1548fd = '0123456789abcdef', _0x54f778 = '';
            for (_0x5da9b5 = 0; _0x5da9b5 < _0x5802aa['length']; _0x5da9b5 += 1) _0xd26743 = _0x5802aa['charCodeAt'](_0x5da9b5), _0x54f778 += _0x1548fd['charAt'](_0xd26743 >>> 4 & 15) + _0x1548fd['charAt'](15 & _0xd26743);
            return _0x54f778;
        }
    }

    function _0x45dccd(_0x5b4c95) {
        {
            return unescape(encodeURIComponent(_0x5b4c95));
        }
    }

    function _0x443ca7(_0x48561e) {

        {
            return _0x57fdd5(_0x45dccd(_0x48561e));
        }
    }

    function _0x184fb0(_0x49a1f3) {

        {
            return _0x3781b2(_0x443ca7(_0x49a1f3));
        }
    }

    function _0x313b78(_0x575158, _0x1fa91a, _0x1cf5de) {
        {
            return _0x1fa91a ? _0x1cf5de ? _0x21cf21(_0x1fa91a, _0x575158) : y(_0x1fa91a, _0x575158) : _0x1cf5de ? _0x443ca7(_0x575158) : _0x184fb0(_0x575158);
        }
    }

    function _0xdad69f(_0x160e3a, _0x3818c5) {
        {
            return _0x313b78(_0x160e3a) + '|' + _0x160e3a;
        }
    }

    function _0x3e5ed0(_0x133a8b, _0x27a18b) {
        {
            return Date['parse'](new Date());
        }
    }

    return _0xdad69f(_0x3e5ed0());
}())

```

### 解题

辛苦得到的美化后的js可以直接被调用

```python
import execjs
import requests

with open('end.js', 'r', encoding='utf8') as f:
    data = f.read()

_url = 'https://match.yuanrenxue.com/api/match/2'
headers = {'User-Agent': 'yuanrenxue.project'}
res = 0

for i in range(1, 6):
    cookie = {'m': execjs.eval(data)}
    value = requests.get(f'{_url}?page={i}', cookies=cookie, headers=headers).json()
    for j in value['data']:
        res += j['value']

print(res)
# 248974
```

## 猿人学3 - 访问逻辑 推心置腹

> https://match.yuanrenxue.com/match/3
>
> 题目要求：抓取下列5页商标的数据，并将**出现频率最高**的申请号填入答案中

抓包看了一手，没什么特殊的

![image-20230103204430002](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103204430002.png)

f12看网络部分有点东西

![image-20230103204449404](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103204449404.png)

每次?page=x请求时都会请求一下/jssm，那就很简单了~

简单测试一下发现cookie中m不是必须的，sessionid是必须的，而对于其它请求头来说，必须具备的有以下几个

```http
POST /jssm HTTP/2
Host: match.yuanrenxue.com
Cookie: qpfccr=true
Content-Length: 0
Accept: */*
Referer: https://match.yuanrenxue.com/match/3
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9


```

但是吧，用python的requests直接发包是无法指定好这几个请求头的顺序，导致无法返回含有Set-Cookie的sessinid

尝试很久，无果，遂查找wp（）发现这里有两种处理方式：

### 元组

需要用自定义的items方法，借用元组的格式来固定顺序；详见[此处](https://zhuanlan.zhihu.com/p/360335607)

### requests.Session()

上一种方式还是有点麻烦，我这里用Session()对象也可以确保headers顺序不变

```python
import requests

headers = {
    'Host': 'match.yuanrenxue.com',
    'Connection': 'keep-alive',
    'Content-Length': '0',
    'Pragma': 'no-cache',
    'Cache-Control': 'no-cache',
    'sec-ch-ua': '"Google Chrome";v="95", "Chromium";v="95", ";Not A Brand";v="99"',
    'sec-ch-ua-mobile': '?0',
    'User-Agent': 'yuanrenxue.project',
    'sec-ch-ua-platform': '"Windows"',
    'Accept': '*/*',
    'Origin': 'https://match.yuanrenxue.com',
    'Sec-Fetch-Site': 'same-origin',
    'Sec-Fetch-Mode': 'cors',
    'Sec-Fetch-Dest': 'empty',
    'Referer': 'https://match.yuanrenxue.com/match/3',
    'Accept-Encoding': 'gzip, deflate, br',
    'Accept-Language': 'zh-CN,zh;q=0.9',
}
session = requests.Session()
session.headers = headers

_url = 'https://match.yuanrenxue.com/api/match/3'
value = []

for i in range(1, 6):
    session.post('https://match.yuanrenxue.com/jssm', headers=headers, cookies={'Cookie': 'sessionid=;'})
    res = session.get(f'{_url}?page={i}').json()
    for j in res['data']:
        value.append(j['value'])

print(max(value, key=value.count))
# 8717
```

