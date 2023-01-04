---
title: "Javascript逆向学习笔记"
slug: "javascript-reverse-study-notes"
description: "逆向真是太开心了呢（） | 更新ing"
date: 2023-01-05T00:44:51+08:00
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

- version2.0

```js
// ==UserScript==
// @name         New Userscript
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       You
// @include        *
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function () {
  'use strict';
  var cookie_cache = document.cookie;
  Object.defineProperty(document, 'cookie',{
      get: function(){
          return cookie_cache;
      },
      set: function(val){
          console.log('Setting cookie', val);
          // 填写cookie名
          if(val.indexOf('m') !== -1){
       // if(val.indexOf('RM4hZBv0dDon443M') !== -1){
              debugger;
          }
          var cookie = val.split(",")[0];
          var ncookie = cookie.split("=");
          var flag = false;
          var cache = cookie_cache.split("; ");
          cache = cache.map(function(a){
              if(a.split("=")[0] === ncookie[0]){
                  flag = true;
                  return cookie;
              }
              return a;
          })
          cookie_cache = cache.join("; ");
          if(!flag){
              cookie_cache += cookie + "; ";
          }
          return cookie_cache;
      }
  });
})();
```



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

## 猿人学4 - 雪碧图、样式干扰

> https://match.yuanrenxue.com/match/4
>
> 题目要求：采集这5页的全部数字，计算**加和**并提交结果

![image-20230104101145806](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104101145806.png)

看着不难，但是数字的部分其实都是图片

![image-20230104101758029](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104101758029.png)

每一个数字出现的顺序在dom中也是乱的，通过style控制，正常的`left:0px`是无偏移，正常按顺序显示，正数是额外向右偏，负数是额外向左偏；还会有多余的数字（通过设置display:none来不显示）

抓包看请求，这些图片是通过请求/api/match/4?page=2来得到的

![image-20230104111129311](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104111129311.png)

但是意外的是并没有任何与display有关的字段，还有几个看不太明白的字段（key, value, iv），说明前端有额外的参与

![image-20230104110910504](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104110910504.png)

![image-20230104111011421](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104111011421.png)

这里根据`hex_md5(bta(key+value))`来设置哪些是display:none的部分

需要的基本都解决了，还有个关键的：具体字母的数字怎么识别？可以用ocr，但因为这里还算简单 都是单个数字拼起来的，也就0~9 共10个数字，所以我们存个字典直接映射即可

愉快的代码时间！！！

```python
import re
import execjs
import requests

img_dict = {
    'iVBORw0KGgoAAAANSUhEUgAAABQAAAAdCAYAAACqhkzFAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAMTSURBVEhLrZY/TBNRHMe/bY82LeGA0Dp4NTGYqO1gdKEMwmQcGh38M5CQlGDSwWiMAwwYg2iCJsZJQ9CJ0ETsQMJgwuagLIUFXIAFXNoYcqDloLRXrq3v7v3aXktbSPCTXO77fZRvf/feu9+r5dz5iwX8R+oESigMPka6/wb2vSJUBx+1QoVD3kTz7DScb+f4YBVHA6U+ZKLPsON1IE9DtbDLP9AxEIawRgOEle4cKYSD+ZeQq8IcKquMXTbyOllPL7aiEWgSDRCmQAnahyHsiGQZruUvkIKXcObyFeM62zcOd1ylvwJ5MYDkuxA5Tjlw8A2S12iyGM7YODrujsFqfqTFCJw9L+CWyTPS3QPIBMgwKFBC9n4AaW4AdQVtQxEy1czBObkIJznAi9TDO6SLgVIYqt9QBq5YFEKCTC2momg2VXngD5bmnAc+uIqUIXQUOL/X3hJl5mHfUEgzPJ04pMUxAnN+CYeG1UlAmCLZANtqAk2kARG5m1zxQI9paWUZAsmGsArLnxOh0ZSxwCBypjwo2zQPxzCTgJ2kTtbbZ9zZ/7pRKO8WOOWqrX9iePyJiqnNGgTTuhRhgT5o5kc+JaeosDYssHbpx+OtXEziSIWqx0fqOFpg7ns2Zd24s8B1WE0V5h2mJW/ELTc0kjo2Zcm4s8ClikCIlR+si59tZpL66yosc2U8clM8bhgDkfVFUzuqR569rqXOqG7CPsOlEWiJJSraUaa/i3Q9upC94CHNCtr4WXoN+aJMzcFlakd73eGG5wnuhZHykmZ1tnx9TboYqDfNWLxoWDvqhfK+h0w1Pcg87cUBOavMmu1HMoxShuXJNNpNi7N3ewLJTyEUzIdQIIT0wgQ7EcmzxWifZMcEOZ3KY3Qwgp3RQOnbOfqJx5XGtlSOS4aK1tlHEIcXyHNsrW0dY6SBFX0ufShc70S21OwE5AR+lb+ZVfb5OVpGviGTTkHeSmBfScJqtdX55SAFoY0OsMXxISM6SvvNrshwrS7A9WoENupyv+O/oGm831sslno/RU6OsvsHu3+3yQH/AOyW6SvqnweCAAAAAElFTkSuQmCC': '0',
    'iVBORw0KGgoAAAANSUhEUgAAABUAAAAcCAYAAACOGPReAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAD0SURBVEhLY5RVUPvPQGXABKUJAA2GvzP3Mry6f5PhzfI4qBhuQNjQqD6Gzxc3Mjxzk2H4CRUiBHAYKs3wP7Gd4evhSwzPWr0ZPvBBhYkEqIaaBzL8nrmS4f3FfQzP6oIY3smwM/yFSpECkAyNY/g2q4PhhZsBwxegy/5BRZlfP2HgIdbfUIAnTD8xCGyuZ5AyW8jATqmhzD9fMwjumsUgbWPKwJu3AipKGkAydC8DZ24sg5SGDQNPei8D01OoMBkAydCnDIyHTkHZlAE8YUo+GDWU+mAEGfq46RCUhQAUGypbZwdlIcBoRFEfjBpKfUADQxkYAKYHOb9g+7HMAAAAAElFTkSuQmCC': '1',
    'iVBORw0KGgoAAAANSUhEUgAAABQAAAAdCAYAAACqhkzFAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAKCSURBVEhLrZZLaBNRFIb/PDp5SFMxjqVOAjYLNUGUbhI3cdEGwaAgbmopWFIQBEUwFhQVldIqCuqilLoymEVVlCqWuNdVXRk31Y1uTKAQMRI0ycQk453JaWaSJjYPPxjmnJPwcefOuSfROXftlvAfaSAUIIXOITfuR9bBQzQBZfqEEzPgVuPY8uIeuMXPVK2lVnjoCrKzJ5F2mKqSZpi/xGCfCEOfpAKhCkPz+HEpgN9sRVpMoqjcJZMJBSVS0Wfeoz94CkaNVE93wOOqygyZr9i+cBnOwT3YsXe/cvUPDsMxHcPWTOU7MmWbD+m5i5RVUIUKGfQt38DOA0dgufuSauskoYuE0RuchV0jzQ8FIAqUMFShmIB9+jhs559SoQnJKKwLcZgpBVwoTFLIUIXXTsMaqdvhZjz8qBECRYeXog2P3CoJts8UMko2F0UdC90o2ihkcAl1mzoTnnGh0kwyGRhXKWR0IBRQOOZGnjL57RsjFDLaF87MI+1Ru9+ysgSOYpn2hKEovo+71RMjfoJtKkpJhRaFAsozr5C67kOOKvLe2e+cBVfXaZsLhSD+LD3DGluZum8p5RA06tt/C4/exK83D7A2xKNEJT07UfzV0aaHoImQHnFuDGlNv8kja2BkBObF5ieqgdDf8BG3PboAPrBx/tVTN7H9KMTuI+WxVQesIRUHPzWKnndU2ATNCgUUn9TKrB8eY8DbukxGFYZu4+dBVWZZYXPvxC3oKG8VEnqRn1R7rCfBfi/Gahu2VSpC3wRyDiViiOhdDre9snUqwsOCZnqkYHhLYSfIb9n5+psESerq6nvOPMylrLDEa7q3C7L7hrVt0z1Fu9Dor0g3AH8BJlTqZkAngxQAAAAASUVORK5CYII=': '2',
    'iVBORw0KGgoAAAANSUhEUgAAABQAAAAdCAYAAACqhkzFAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAALaSURBVEhLrZVPSBRRHMe/O+rOuuaUyRoyBlGQfw5lh9xTekg8qIdMKSsQFKRAD0WUROhBCIlSBD1YQsJKQSB1iRa8VRf1Up4UQiNcRVw1nXVdx3V3euP+ZnZmR9EVP/CY3/vx5sPMe+/3nu3suYsKjhGrUCxB5GErNssKERIEyDzlIYOX/MgY+4r0zm5w85ROwCRU2j1Yu+fGhi7ZG072Iau3Bc6BacrE4ejJ6EKgySyzy+yrdptpIKJ8HlbahrHZKFImjnEcIeHU6FuIlfk4U3AJObstH2L9C7hmJKTQKEBg0h7sUE/DJLT5x5FbeRWZ99kcTVFSY9wDR/kN5Pxkn6vBFyPUTjFhEPYjs6YBqYkiE/NIfTeODOqpyEUNFMUwCNmy7bNyJr5M787pfuwxhwfhh+1YhWIxtgWKGXafh6IYSQuVjssIUgxMwdFLIZGc8IEHKxV5iFL3xGg37AnzfnAtu68jUlYFueIa1i4IiFDaOfYa2XcGqRfHKnzlxVzdeepYSZVmkfXyERwfrGWnktQvc7If6T62yrxhVRKwCvX6jTet3KK8C4EiN5Y6hrEw6YV811rLhzwPRSi19dhqqmFCFzvINPzI7rwN51B8ZQ4pNNA4iNW2UgTpVOL835Fb0qz/atL7EEPNyPrm01+MutwIPaUOI3khw9Y5aTggeISu1FN8RCHm5YQX7fQ8qlDk9WpR4eRlinQhW37rDtgXpc1Yz2xrsYtLg4StCHz+iHD1IazVPQgY6hnSLzgGKGbov6y4irHY58W/T13YKS2grIHCGoTfeLHcV4V1w9V6+v0zwz2j78MuSH9uYp2SKhwbnKbvYB5hJjHOmyo7OdIC4ckP6segL5wAb7rR1Jd5dslrzSxLYQeE6/ktXbYVCmJp0YfghpRQKbWPsVVXjs0iEWGBxzalVXhZQtrMFJwj/eCHJigbY2FuFpFI7EJNvvT2wPf3NxRF1QD/AbAv8WdRHzjKAAAAAElFTkSuQmCC': '3',
    'iVBORw0KGgoAAAANSUhEUgAAABUAAAAbCAYAAACTHcTmAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAHHSURBVEhLrZW9LwNhHMe/WrT10mrKVE2EROIfUAuTTQxMLISBQSxeIjQSTZRIsYgYhYVRIrWzaCxiMjFVoqjWaatPaR9Pe09bbfT6XHKf5Mn9vne9T+55fs/1qhxtnRRqWD1DcLILSR5Nfg+aR495ktHxoyBDiA8WhOVQJaXeKUgtPCggLrXPs6dsxzePSghLU9sjiBh4qICYtG8fUo8Z6UxNXlEvZc+WRUDaC+LpR5SnhssLGHldjopS6l1BpFWuda9XsEwH5KCAstS5iU/WHHkLSbAerAmtl8JvupH0DOCDN8d4e4q6wyc5VKCslHrdCHdwI7mHZXZHrgX4X9r3d9oElvMN1Io9ZJZ/pL1IeIbz06598KFx8UYOgpRI7fg52UWIdzszbev4Mqp4FKVYOrGJSG6Ts27btmZUTTtHQcrWUVpy4otHk39PuNulyFL7GOJ7hXWsCfhgK/mPVAOTstfwaAEhMz9DHmF1zalexyIcbj81UUqh0TBdr9OS7muDDoRAz4ZBxajmN2dggqJrepKE+g8fWFPvXPkeaPDhE0MzaeIrhpfnAGJRSTvp+1sQJBFHOBTUTppOp7LHzM7STGpusvEK+AUL4d3X/AgqvQAAAABJRU5ErkJggg==': '4',
    'iVBORw0KGgoAAAANSUhEUgAAABUAAAAcCAYAAACOGPReAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAIxSURBVEhLrZU/aBNRHMe/bWKexnoiaSg0FSSCaAfFwT+L7dKloUs7OqRVcHBwcZOCiCAOhk4dxAwdHFRwEEoRSjvoZNqhdDE4tC7aIYmWvOiV1941vnf51dy7S0ou6Qfu+P5+B9/7/e7e+72us+cuVHHE+EzF0jcUzlPQCjyH/itphChUdKub+bcCXv4tVQr7hsp0hmNqWXsob5ek6kWVqUxneNpPw1yfxi+nWo7Y02uIzjkPAuFUWucSrKNqvzEcoUWSAdFNbyewSxJK/SQZEN1U+0kCXaSCopvGGfZJgsv2SQbFY2rI+jqn+Y/ipbYr1dap/WYVWzfra0q98f/nkDDBcWwjj+j7WbC5Fcr6aV6pxG2oEMzAn8EbKDx+ja31jxBytTRCM7WNepVhIWRl6qKEB9tIovDsA8w7fuPWR9/QPezenQAfTmKHUg5iE32To4jkKJYc2r7G5ywiU6PoTWUQK7rKZ0lU7o9TUKN10wPyWUQfLOA0hQrz6jhs0orgporcI0TXXNUacVgkFe2ZSsI/iqT8tG162DBv23RvYICURG4K3xkVmKEZmIOkJWqXhUkrgpsm0tjJjKBCoRqRp+afkK7hMr2I6tgt0g1IXIf94h22l6dRkiPygMjXtzj5kgLCtaOeg3+fQNnR/u1pMaatRUVkYwHxyYfo9pwQTdpncnjol24o0PMpg74Rv6HCZbqC41/y6OG1QeKdpSGZO1HcRGz+FRKpyzgzlaUnXoB/3J2gmVZucHAAAAAASUVORK5CYII=': '5',
    'iVBORw0KGgoAAAANSUhEUgAAABUAAAAdCAYAAABFRCf7AAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAANLSURBVEhLrZZNSFRRFMf/48e8UZtRktFkRsiEdCIKVwaikrlJbZG0MEGDwEWUIPaBQoiIZIJGG4mikoyyIBJKW4gbs4W66WOhtlAXzixq/HzGjG/8eN0378yb+5w3qdAPLnPOmXf/nHfvOfc+U+bR4zL+M/sTrW6F71Ih/CfskAQB2xQ2SyLMU5OwPr+HuEEPRfcSddXB96QeK04BOxQyZg7pWedhJi+GfiOp7sHqwC0s7RKMkyQIwaGfzGdmLFrUAbGlFOsC+ZBwaLwfjqoSZOSeQlpw5MBR1oQjw3NIoKdCGLz+Rfgn72PRTq7khr2tFpbX4TWLwJULTM+QY5Cp/PgGlkOCEJHauYegAieosEuUbUyxU1tDy9c3SOzdQ9AAvWh7ObeOblg7u8k+GJyoA4E8FzbJi5mdhGWCnAPCiVZhM5tMRuLsAFkHJyxakcu6hWy2QcL4pGpW3IR/6AsWZ35iYV4dnvkfWBp7C/+dMvWZ3SglFRzPpuR4WZYRHAtyWsFZ2fFhVha0mPEQFkbljALSoBHO1G7V1hMIQG54Cu+FY6zsVcyhTiI/hOQswq9PfdhyUIChiW7bbWQpOLDOBANMMknppLIcpIc6KasEzrYhJIv0KGPHlo/VrlrydBvFI2CDNXfyu+s4fLkVMdMUDuKBqbcRtvr3SA69BsN/phIByjaKKPvDPQLr7THyDPjcDOuwmxNwYaNBtaKKWse7YSI7GqbO70giWyHgVJdAE431covESipuah/t6fkGMzdNsruCv+FMveuIJzMavpMlEItryIskdGaERYc9XLnYsO0kk2Mr1YG1c3XkRZLgVXc0LDoxAYF7lY28yMm20ZfIbCkij5GfD4mrxFhRPQK5jepjR52XbEW0XCuRaOxcO40/ZLP1g+Wj2tqcKHMesZuRbKVEVl506O4eHRU9WCvWTnPEz47BMqjaOlFMNLJSErVgILsSiyMPsJ3Pp5yLnfY+LHeVclm6kdLWrJWgwR1VCGmkB7+z9V2u9H5wErv3uUZiiEh5xZK5G24UfaZBxiBcaUL6lIhYiigEFLFdgjHsUkztrNEJKvz7Y6K6A76rhfA5lS+TUB1KrHQ8SBztR8LDPpgMemR/nz0HAvgL80YzEyuMQpQAAAAASUVORK5CYII=': '6',
    'iVBORw0KGgoAAAANSUhEUgAAABQAAAAeCAYAAAAsEj5rAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAHhSURBVEhLrZY9LANhGMf/PprDVQkxyFXSVCKIwYStiZoqhoqEhTCQWCwiBBEDiUTMBpOJhKQDkw1L1YDFxoSlBi2tlurrrn2uH9xX2/sll/6f4X557n3e903LWhxtDCZSTr/mIXXY4thgjSHGwEp76o7amOkdmih8BHcMlDAULz4DW3htSlf8+RIaJn3Fd8i2ZxAiGeK3sK34UrE4oTCP6JAT31TWneyg8jmdixKyNQ/eOCqCF7AuBKgoRigs48NlR5LK+rP1PEnBwuRmfnf8Kn0rUZhQWEfEJU/if3cSBQmTmwN4o6zUnYRxobh2kb5sd7X+PcWXDQvZohthee3EfWedy042F4PCCXzmTLbGf4hKyn8xJtwYRthGGU/gd9OnQgkDwh7EXB2ZU2G5v0TVFRUK6At7xxCzUxbhb/YoKaMrTM724J2yNIxqha2Si47Qi1hndqtw9wHFYUS7+hF2jaeytnDKi2jGFwfv36GcT6JRQMg9ncqaF+zPwTVe+mi84uc2t4+qbhcZjQ49+GrN7BVYHu50ZRLqwt5BxLLLh6qHfUraqAtHnIhSFG8CcGfa05VRFSZahcxRQ/wZllPKOqgIPfi2yzeBSPAVFRT1UBF2I5GzflzwEWWU9TD5zxLwC1sVsHrJiVs0AAAAAElFTkSuQmCC': '7',
    'iVBORw0KGgoAAAANSUhEUgAAABQAAAAdCAYAAACqhkzFAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAANtSURBVEhLrZXfS1NhGMe/rumZbR4nOcO2IA1CvQi8UW8y8EJqZCR1IQiKwvRCuiqTfomtkkrqIpN+EFSGNaFECL0IbzQvnH/A6EavNiinJVubO/5a73nPs51zNqf04wPjfJ8X9uV5n/d93ifr8JFjcfxHMhueuYS1dieiFTZIgoAtvihBCAVhnptErvshDAG+qGMHwzJsvh7Cj5MO9vfMGCQ/Ckb6sP/2F1pRMNCXsGPz/Vt8TzEzSiwz/qMFxrbgwEr7I0Tb7LSioDe8OYSVGhHbFOYEZ1DcVIfisuMo4r86ONxTyEsai1jpGcK6xlNj6MRaQznWKTL4p1BU5YLRqy1UAFmvumDtGEN+wlQoR7SnigKd4Qls2EiyDVs/dSGLojRmrsIyF6QAiFQ0ktIaslokspMzMT4gmQFDMEyK1ZO+MqqhL4x9JFn1EK8m+Yeohl4vu2Ok4UCsWa1LOmw3FaWkAbNfvTqaGg4jd86fXAjX34JUS0EK8YGX+FlBARZhfjpJWmfIbnlnHwoX6PiEUiy9mEVkwIV4ubKEWheksVksXSilekvI/9APwcsDTnqn2JsQ81zDikPQFTsdCXmf+2Ht9FCsoMuQE/DAdHkQBeqt2BHz9CDy3XozmZQM7dh+Pozlen3rGViUzRa22COxSWsycj8fcLfA9E69/JoM5T4exTeNWe7CBA6x1rOXKK1XXCK33gSsdBvkfg7eHdX1s5phG8ustxprPJCL3QWxW/+SJLE7Ib25h6WjghKHvCh2tsDIEqUMqxBrT5ixa+3zZDaTCUxCaPXAmtiKWI1f1M+KYXUr1hxcMSRYpvtJ70KgH7k+tdLhGhe/FYphvV1zCHv3cQLjguYlsjn4gSmGooANLv4N+TAUw2AIVF5GIbbaSO7BlqOQFEMK8cdFMRxZhIkLGRGRBhfp3XAhVimSZkbBgMYw8AQmH1ecWGVH2qzQEo3kYfVUGKvqtpA35+EPsmLIDiLn/hQsSsBgs6J3HKHHLYin+p5n47XtLMLP3MleN7DZY+me51rXevE741huLkeM4gQ5bOIp40BgM5qLJAZpEYUdpyHMUKx8FLJunIPtutpaCdZZD8vDPtXM5J/BwUbVTGaHQS9Thu0rF9kUrELEJmKDGSW2J7DTzPbNwzIyiOyPX2lVJYPh3wL8BvLZG6cpuRANAAAAAElFTkSuQmCC': '8',
    'iVBORw0KGgoAAAANSUhEUgAAABQAAAAcCAYAAABh2p9gAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAMzSURBVEhLpZZPSBRhGMYfV9dpdV3UHDVXIw1KDUwJVy/VLTAVWugQRJqUpxAkFyuMIBMPWRGYiCGGC7J0KCRKL13Si9pBvKiH9NIYoavgqKvjujt9M/OuM+uustUPvpn3edl9eL/3+7Mbl3fqjIxoFDnhb76Bzcoi7No47FI6QRJxbHYKKQOdSPi8RFmdKIZ2BDt6sHazCNuUiY4E67cXSLvtJq1hojdhx57nPX4fMDOxL3OSMiihwmHzchu8njrSGmGGclc/Vit5BEibxEXw7bdgzy9BZqEyziLP5UH6Cn2AsV3ZBF+DnVTYlBuxNe/CGqcpSHPIdl6DeY60EXsdfCNtWLVp0rQyhhOORrU6vcKOamyEzNgUU4fuRTdTWHIj6fUkLCSDfCl2GrR433CvuAB+ipXqLM8iVzCMd29gFSiGDb4rWi/J0IE9fr88QFhEAoWHMwWzIFLMeplbAaV3ZFiIIPXjb4gXvBQx+Ax1MfUeGuEM1cYKZzMauhFv2Arg2X6k8CgCuRkUKaQgWGOo0CwYHLlS+LocJA7B3oKdMmOfOMi8wdDUy84nxQrrtU8h1egbNpyLkAbr9T1rQO/h5EtYJ0Q9wRVguXsYG30tCFwq1HJ2B4KtPdiY6cfyac3NrD4VJMQpk1ROij7uyJk/dmTIcgxjXT7+aUK27OsFOZN56BWqjIOrf4isWRHxlImOiLSh+0iaJqnCKmTPA4aMpREkVpcjx/UWGdOLSBZDNw0booC0iY/IuVoO6+NxoJjXbyVRK+LwCzYGAp7v+FWprbR5dhDZ1Z1RKowZB/y5+rbhhK/qO2bD/FMncb7kHClGRT07vxRDgGVgSo1iNrTZUnChrIQUm26zA5sUm4QZcJNa/G89bHDD+6SCFkRCem8Vkp9r1114hfbDToaBmlcQH4TMgMSFL0giM4Vww2Z2AqaGsd3qpIQB5We1bxTe7mqsh46ctIjU9kfq/gsRPuWuUfy8XkCCrRzbexTBz0yCpBRMzCzDdRfcgd/mIxdFYveiNsLNOGEMWc6qCDOFA4vCbu7WJmzXOrDF21SjEBz7x2Bm/xisQ90wf5inbCT/dVIiAf4ApbEnkB6qHqsAAAAASUVORK5CYII=': '9',
}

with open('hex_md5.js', 'r', encoding='utf-8') as f:
    ctx = execjs.compile(f.read())

num = 0

for i in range(1, 6):
    res = requests.get(f'https://match.yuanrenxue.com/api/match/4?page={i}', headers={'User-Agent': 'yuanrenxue.project'}).json()
    display_none = ctx.call('getResult', res['key'], res['value'])
    tds = re.findall('<td>(.*?)</td>', res['info'])
    imgs = []
    for td in tds:
        _imgs = re.findall(r'data:image/png;base64,(.+?)" class="img_number (.*?)" style="left:(.*?)px', td)
        for i in range(-len(_imgs), 0, 1):
            if _imgs[i][1] == display_none:
                _imgs.remove(_imgs[i])
        imgs.append(_imgs)

    step = 11.5
    for group in imgs:
        n = list('    ')
        for i in range(len(group)):
            flag = int(float(group[i][2]) / step)
            n[i + flag: i + 1 + flag] = img_dict[group[i][0]]
        num += int(''.join(n).strip())

print(num)
# 243701
```

## 猿人学5 - js混淆 乱码增强

> https://match.yuanrenxue.com/match/5
>
> 题目要求：抓取全部5页直播间热度，计算前**5**名直播间热度的**加和**
>
> 注：本页cookie有效期仅有50秒钟

又与时间戳有关，这次请求时get会带两个参数（同第一题 m和f），cookie也会带两个参数 m和RM4hZBv0dDon443M，我们惯例hook一下

![image-20230104170944236](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104170944236.png)

进入set的前一步，果然，又是强混淆，而且一眼顶针 鉴定为ob

![image-20230104171204264](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104171204264.png)

凭直觉猜出这里赋值是这样的

```js
document['cookie'] = 'm=' + _0x474032(timestamp)
```

timestamp由\_0x12eaf3生成，m的值由_0x474032生成

```js
function _0x474032(_0x233f82, _0xe2ed33, _0x3229f9) {
	return _0xe2ed33 ? _0x3229f9 ? v(_0xe2ed33, _0x233f82) : y(_0xe2ed33, _0x233f82) : _0x3229f9 ? _0x41873d(_0x233f82) : _0x37614a(_0x233f82);
}
```

调用的时候只传入了一个参数，相当于只调用了_0x37614a；把涉及到的函数都拖出来，同时手动做简化（把其中数组、常量之类的部分直接控制台找对应的值然后做替换）

```js
var window = {};
var _0x1171c8 = 0x67452301;
var _0x4dae05 = -0x10325477;
var _0x183a1d = -0x67452302;
var _0xcfa373 = 0x10325476;
window._$tT = -0xa40bd9c;
window._$Jy = 0x1b821d58;


_0x474032(_0x12eaf3())

function _0x12eaf3() {
    return Date['parse'](new Date());
}

function _0x474032(_0x233f82, _0xe2ed33, _0x3229f9) {
    console.log(_0x37614a(_0x233f82))
    return _0x37614a(_0x233f82);
}

function _0x37614a(_0x32e7c1) {
    return _0x499969(_0x41873d(_0x32e7c1));
}

function _0x41873d(_0x5a6962) {
    return _0x1ee7ec(_0x2b8a17(_0x5a6962));
}

function _0x499969(_0x82fe7e) {
    var _0x5bdda4, _0x322a73, _0xd0b5cd = '0123456789abcdef', _0x21f411 = '';
    for (_0x322a73 = 0x0; _0x322a73 < _0x82fe7e['length']; _0x322a73 += 0x1)
        _0x5bdda4 = _0x82fe7e['charCodeAt'](_0x322a73),
            _0x21f411 += _0xd0b5cd['charAt'](_0x5bdda4 >>> 0x4 & 0xf) + _0xd0b5cd['charAt'](0xf & _0x5bdda4);
    return _0x21f411;
}

function _0x1ee7ec(_0x206333) {
    return _0x12b47d(_0x11a7a2(_0x35f5f2(_0x206333), 0x8 * _0x206333['length']));
}

function _0x35f5f2(_0x243853) {
    var _0x139b8b, _0xa791a1 = [];
    for (_0xa791a1[(_0x243853['length'] >> 0x2) - 0x1] = void 0x0,
             _0x139b8b = 0x0; _0x139b8b < _0xa791a1['length']; _0x139b8b += 0x1)
        _0xa791a1[_0x139b8b] = 0x0;
    var _0x41a533 = 0x8 * _0x243853['length'];
    for (_0x139b8b = 0x0; _0x139b8b < _0x41a533; _0x139b8b += 0x8)
        _0xa791a1[_0x139b8b >> 0x5] |= (0xff & _0x243853['charCodeAt'](_0x139b8b / 0x8)) << _0x139b8b % 0x20;
    return _0xa791a1;
}

function _0x2b8a17(_0x36f847) {
    return unescape(encodeURIComponent(_0x36f847));
}

function _0x12b47d(_0x149183) {
    var _0xabbcb3, _0x1145c3 = '', _0x4fce58 = 0x20 * _0x149183['length'];
    for (_0xabbcb3 = 0x0; _0xabbcb3 < _0x4fce58; _0xabbcb3 += 0x8)
        _0x1145c3 += String['fromCharCode'](_0x149183[_0xabbcb3 >> 0x5] >>> _0xabbcb3 % 0x20 & 0xff);
    return _0x1145c3;
}

function _0x11a7a2(_0x193f00, _0x1cfe89) {
    _0x193f00[_0x1cfe89 >> 0x5] |= 0x80 << _0x1cfe89 % 0x20,
        _0x193f00[0xe + (_0x1cfe89 + 0x40 >>> 0x9 << 0x4)] = _0x1cfe89;
    try {
        var _0x42fb36 = 16;
    } catch (_0x1b1b35) {
        var _0x42fb36 = 0x1;
    }
    op = 26;
    b64pad = 1;

    var _0x1badc3, _0x38ca59, _0x431764, _0x43f1b4, _0x5722c0, _0x3e0c38 = _0x1171c8, _0xdb4d2c = _0x4dae05,
        _0x1724c5 = _0x183a1d, _0x257ec6 = _0xcfa373;
    if (window['_$6_']) {
    } else {
        window['_$6_'] = 0x20dc5d57f;
    }
    for (_0x1badc3 = 0x0; _0x1badc3 < _0x193f00['length']; _0x1badc3 += _0x42fb36)
        _0x38ca59 = _0x3e0c38,
            _0x431764 = _0xdb4d2c,
            _0x43f1b4 = _0x1724c5,
            _0x5722c0 = _0x257ec6,
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3], 0x7, 0x7d60c),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x1], 0xc, window['_$6_']),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x2], 0x11, 0x242070db),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x3], 0x16, -0x3e423112),
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x4], 0x7, -0xa83f051),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x5], 0xc, 0x4787c62a),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x6], 0x11, -0x57cfb9ed),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x7], 0x16, -0x2b96aff),
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x8], 0x7, 0x698098d8),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x9], 0xc, -0x74bb0851),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xa], 0x11, -0xa44f),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xb], 0x16, -0x76a32842),
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xc], 0x7, 0x6b901122),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xd], 0xc, -0x2678e6d),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xe], 0x11, -0x5986bc72),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xf], 0x16, 0x49b40821),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x1], 0x5, -0x9e1da9e),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x6], 0x9, -0x3fbf4cc0),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xb], 0xe, 0x265e5a51),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3], 0x14, -0x16493856),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x5], 0x5, -0x29d0efa3),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xa], 0x9, 0x2441453),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xf], 0xe, window['_$tT']),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x4], 0x14, window['_$Jy']),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x9], 0x5, 0x21e1cde6),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xe], 0x9, -0x3cc8aa0a),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x3], 0xe, -0xb2af279),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x8], 0x14, 0x455a14ed),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xd], 0x5, -0x5caa8e7b),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x2], 0x9, -0x3105c08),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x7], 0xe, 0x676f02d9),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xc], 0x14, -0x72d5b376),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x5], 0x4, -0x241282e),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x8], 0xb, -0x788e097f),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xb], 0x10, 0x6d9d6122),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xe], 0x17, -0x21ac7f4),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x1], 0x4, -0x5b4115bc * b64pad),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x4], 0xb, 0x4bdecfa9),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x7], 0x10, -0x944b4a0),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xa], 0x17, -0x41404390),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xd], 0x4, 0x289b7ec6),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3], 0xb, -0x155ed806),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x3], 0x10, -0x2b10cf7b),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x6], 0x17, 0x2d511fd9),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x9], 0x4, -0x3d12017),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xc], 0xb, -0x1924661b),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xf], 0x10, 0x1fa27cf8),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x2], 0x17, -0x3b53a99b),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3], 0x6, -0xbd6ddbc),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x7], 0xa, 0x432aff97),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xe], 0xf, -0x546bdc59),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x5], 0x15, -0x36c5fc7),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xc], 0x6, 0x655b59c3),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x3], 0xa, -0x70ef89ee),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xa], 0xf, -0x644f153),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x1], 0x15, -0x7a7ba22f),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x8], 0x6, 0x6fa87e4f),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xf], 0xa, -0x1d31920),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x6], 0xf, -0x5cfebcec),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xd], 0x15, 0x4e0811a1),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x4], 0x6, -0x8ac817e),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xb], 0xa, -1120211379),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x2], 0xf, 0x2ad7d2bb),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x9], 0x15, -0x14792c01),
            _0x3e0c38 = _0x12e4a8(_0x3e0c38, _0x38ca59),
            _0xdb4d2c = _0x12e4a8(_0xdb4d2c, _0x431764),
            _0x1724c5 = _0x12e4a8(_0x1724c5, _0x43f1b4),
            _0x257ec6 = _0x12e4a8(_0x257ec6, _0x5722c0);
    return [_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6];
}

function _0x48d200(_0x4b706e, _0x3c3a85, _0x111154, _0x311f9f, _0x5439cf, _0x38cac7, _0x26bd2e) {
    return _0xaaef84(_0x3c3a85 & _0x111154 | ~_0x3c3a85 & _0x311f9f, _0x4b706e, _0x3c3a85, _0x5439cf, _0x38cac7, _0x26bd2e);
}

function _0xaaef84(_0xaf3112, _0x2a165a, _0x532fb4, _0x10aa40, _0x41c4e7, _0x1cb4da) {
    return _0x12e4a8(_0x3634fc(_0x12e4a8(_0x12e4a8(_0x2a165a, _0xaf3112), _0x12e4a8(_0x10aa40, _0x1cb4da)), _0x41c4e7), _0x532fb4);
}

function _0x12e4a8(_0x7542c8, _0x5eada0) {
    var _0x41f81f = (0xffff & _0x7542c8) + (0xffff & _0x5eada0);
    return (_0x7542c8 >> 0x10) + (_0x5eada0 >> 0x10) + (_0x41f81f >> 0x10) << 0x10 | 0xffff & _0x41f81f;
}

function _0x3634fc(_0x5803ba, _0x1ce5b2) {
    return _0x5803ba << _0x1ce5b2 | _0x5803ba >>> 0x20 - _0x1ce5b2;
}

function _0x4b459d(_0x8d8f2a, _0x406d34, _0x53e7d7, _0x26c827, _0xec41ea, _0x52dead, _0x3f66e7) {
    return _0xaaef84(_0x53e7d7 ^ (_0x406d34 | ~_0x26c827), _0x8d8f2a, _0x406d34, _0xec41ea, _0x52dead, _0x3f66e7);
}

function _0x3180ec(_0x401705, _0x240e6a, _0x56b131, _0x5a5c20, _0x1f2a72, _0x2bfc1, _0x19741a) {
    return _0xaaef84(_0x240e6a & _0x5a5c20 | _0x56b131 & ~_0x5a5c20, _0x401705, _0x240e6a, _0x1f2a72, _0x2bfc1, _0x19741a);
}

function _0x32032f(_0x520fdf, _0x13921d, _0x1af9d5, _0x4a2311, _0xb6d40a, _0x1d58da, _0x361df0) {
    return _0xaaef84(_0x13921d ^ _0x1af9d5 ^ _0x4a2311, _0x520fdf, _0x13921d, _0xb6d40a, _0x1d58da, _0x361df0);
}

```

测试一下 和网站生成的m值一样，接下来看第二个要操心的cookie，我们把hook的cookie名称改为RM4hZBv0dDon443M

![image-20230104223840041](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104223840041.png)

成功断在这里，但是有个细节：RM4hZBv0dDon443M经过了好几次才被最终赋值，之前都是undefined，我们先只关注这最后赋值时的部分

![image-20230104224210009](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104224210009.png)

```js
try {
    _0x3d0f3f[_$Fe] = 'R' + 'M' + '4' + 'h' + 'Z' + 'B' + 'v' + '0' + 'd' + 'D' + 'o' + 'n' + '4' + '4' + '3' + 'M=' + _0x4e96b4['_$ss'] + ';\x20path=/';
} catch (_0x5399ac) {
    _0x3d0f3f[_$Fe] = 'R' + 'M' + '4' + 'h' + 'Z' + 'B' + 'v' + '0' + 'd' + 'D' + 'o' + 'n' + '4' + '4' + '3' + 'T=' + '=True;\x20path=/';
}
```

难怪我们直接搜RM4hZBv0dDon443M找不到 它又拆开了，这部分js和上面m的在同一个文件里（伏笔），和上面一样，我们找`_$ss`的赋值处 奇怪的是并没有

我们接着hook~

```js
(function() {
    Object.defineProperty(window, '_$ss', {
    set: function (val){
        debugger;
        return val;
        }
    });
})();
```

当断在这里的时候，就可以在堆栈中找到想要的值了

![image-20230104225153332](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104225153332.png)

简化一部分上面的代码（简化时注意最好能换的都换，拿不准的就控制台执行一下~

```js
var _$Ww = CryptoJS['enc']['Utf8']['parse'](_$pr['toString']());
var _0x29dd83 = CryptoJS['AES']['encrypt'](_$Ww, _$qF, {
    'mode': CryptoJS['mode']['ECB'],
    'padding': CryptoJS['pad']['Pkcs7']
});
var _$ss = _0x29dd83['toString']();
```

搜索`_$qF`，是这样的

![image-20230104232051032](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104232051032.png)

```js
var _$qF = CryptoJS['enc']['Utf8']['parse'](btoa(_$is)['slice'](0x0, 0x10));
```

再看`_$pr`，发现这玩意是个数组

![image-20230104233419567](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230104233419567.png)

代码中，每次执行跟cookie有关的操作后都会对`_$pr`进行push，push进去的值就是cookie中m的值，m共生成5次 m的值取最后一次，而RM4hZBv0dDon443M的值和这5个值都有关

那能不能for循环手动生成5次m提供给RM4hZBv0dDon443M呢？收回伏笔，这里是不行的

原因在于每一次m生成后都小小的修改了其中的一些参数，并且第四次生成时的处理方式还不一样，反映到数组中有4个不一样的值，导致单纯的循环是不行的，我们要针对性的改代码

coding时间……

```python
import execjs
import requests

hot_list = []
with open('setCookie.js', 'r', encoding='utf-8') as f:
    ctx = execjs.compile(f.read())

for i in range(1, 6):
    value = ctx.call('getRM').split(':::')
    res = requests.get(f'https://match.yuanrenxue.com/api/match/5?m={value[3]}&f={value[2]}&page={i}', headers={'User-Agent': 'yuanrenxue.project'}, cookies={'m': value[1], 'RM4hZBv0dDon443M': value[0], 'sessionid': 'v2benslruc0rmffirrkzho7drpdjbm98'}).json()
    data = [i['value'] for i in res['data']]
    hot_list.extend(data)

hot_list.sort(reverse=True)
print(hot_list)
print(sum(hot_list[:5]))
# 47313
```

```js
var CryptoJS = require("crypto-js")
var _0x1171c8 = 0x67452301;
var _0x4dae05 = -0x10325477;
var _0x183a1d = -0x67452302;
var _0xcfa373 = 0x10325476;
var _$6_ = 0x20dc5d57f;
var _$tT = -0xa40bd9c;
var _$Jy = 0x1b821d58;


function _0x12eaf3() {
    return Date['parse'](new Date());
}

function _0x474032(_0x233f82, _0xe2ed33, _0x3229f9) {
    return _0x37614a(_0x233f82);
}

function _0x37614a(_0x32e7c1) {
    return _0x499969(_0x41873d(_0x32e7c1));
}

function _0x41873d(_0x5a6962) {
    return _0x1ee7ec(_0x2b8a17(_0x5a6962));
}

function _0x499969(_0x82fe7e) {
    var _0x5bdda4, _0x322a73, _0xd0b5cd = '0123456789abcdef', _0x21f411 = '';
    for (_0x322a73 = 0x0; _0x322a73 < _0x82fe7e['length']; _0x322a73 += 0x1)
        _0x5bdda4 = _0x82fe7e['charCodeAt'](_0x322a73),
            _0x21f411 += _0xd0b5cd['charAt'](_0x5bdda4 >>> 0x4 & 0xf) + _0xd0b5cd['charAt'](0xf & _0x5bdda4);
    return _0x21f411;
}

function _0x1ee7ec(_0x206333) {
    return _0x12b47d(_0x11a7a2(_0x35f5f2(_0x206333), 0x8 * _0x206333['length']));
}

function _0x35f5f2(_0x243853) {
    var _0x139b8b, _0xa791a1 = [];
    for (_0xa791a1[(_0x243853['length'] >> 0x2) - 0x1] = void 0x0,
             _0x139b8b = 0x0; _0x139b8b < _0xa791a1['length']; _0x139b8b += 0x1)
        _0xa791a1[_0x139b8b] = 0x0;
    var _0x41a533 = 0x8 * _0x243853['length'];
    for (_0x139b8b = 0x0; _0x139b8b < _0x41a533; _0x139b8b += 0x8)
        _0xa791a1[_0x139b8b >> 0x5] |= (0xff & _0x243853['charCodeAt'](_0x139b8b / 0x8)) << _0x139b8b % 0x20;
    return _0xa791a1;
}

function _0x2b8a17(_0x36f847) {
    return unescape(encodeURIComponent(_0x36f847));
}

function _0x12b47d(_0x149183) {
    var _0xabbcb3, _0x1145c3 = '', _0x4fce58 = 0x20 * _0x149183['length'];
    for (_0xabbcb3 = 0x0; _0xabbcb3 < _0x4fce58; _0xabbcb3 += 0x8)
        _0x1145c3 += String['fromCharCode'](_0x149183[_0xabbcb3 >> 0x5] >>> _0xabbcb3 % 0x20 & 0xff);
    return _0x1145c3;
}

function _0x11a7a2(_0x193f00, _0x1cfe89) {
    _0x193f00[_0x1cfe89 >> 0x5] |= 0x80 << _0x1cfe89 % 0x20,
        _0x193f00[0xe + (_0x1cfe89 + 0x40 >>> 0x9 << 0x4)] = _0x1cfe89;

    var _0x42fb36 = 16;
    var b64pad = 1;

    var _0x1badc3, _0x38ca59, _0x431764, _0x43f1b4, _0x5722c0, _0x3e0c38 = _0x1171c8, _0xdb4d2c = _0x4dae05,
        _0x1724c5 = _0x183a1d, _0x257ec6 = _0xcfa373;
    for (_0x1badc3 = 0x0; _0x1badc3 < _0x193f00['length']; _0x1badc3 += _0x42fb36)
        _0x38ca59 = _0x3e0c38,
            _0x431764 = _0xdb4d2c,
            _0x43f1b4 = _0x1724c5,
            _0x5722c0 = _0x257ec6,
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3], 0x7, 0x7d60c),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x1], 0xc, _$6_),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x2], 0x11, 0x242070db),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x3], 0x16, -0x3e423112),
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x4], 0x7, -0xa83f051),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x5], 0xc, 0x4787c62a),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x6], 0x11, -0x57cfb9ed),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x7], 0x16, -0x2b96aff),
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x8], 0x7, 0x698098d8),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x9], 0xc, -0x74bb0851),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xa], 0x11, -0xa44f),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xb], 0x16, -0x76a32842),
            _0x3e0c38 = _0x48d200(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xc], 0x7, 0x6b901122),
            _0x257ec6 = _0x48d200(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xd], 0xc, -0x2678e6d),
            _0x1724c5 = _0x48d200(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xe], 0x11, -0x5986bc72),
            _0xdb4d2c = _0x48d200(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xf], 0x16, 0x49b40821),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x1], 0x5, -0x9e1da9e),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x6], 0x9, -0x3fbf4cc0),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xb], 0xe, 0x265e5a51),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3], 0x14, -0x16493856),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x5], 0x5, -0x29d0efa3),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xa], 0x9, 0x2441453),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xf], 0xe, _$tT),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x4], 0x14, _$Jy),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x9], 0x5, 0x21e1cde6),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xe], 0x9, -0x3cc8aa0a),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x3], 0xe, -0xb2af279),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x8], 0x14, 0x455a14ed),
            _0x3e0c38 = _0x3180ec(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xd], 0x5, -0x5caa8e7b),
            _0x257ec6 = _0x3180ec(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x2], 0x9, -0x3105c08),
            _0x1724c5 = _0x3180ec(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x7], 0xe, 0x676f02d9),
            _0xdb4d2c = _0x3180ec(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xc], 0x14, -0x72d5b376),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x5], 0x4, -0x241282e),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x8], 0xb, -0x788e097f),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xb], 0x10, 0x6d9d6122),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xe], 0x17, -0x21ac7f4),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x1], 0x4, -0x5b4115bc * b64pad),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x4], 0xb, 0x4bdecfa9),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x7], 0x10, -0x944b4a0),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xa], 0x17, -0x41404390),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xd], 0x4, 0x289b7ec6),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3], 0xb, -0x155ed806),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x3], 0x10, -0x2b10cf7b),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x6], 0x17, 0x2d511fd9),
            _0x3e0c38 = _0x32032f(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x9], 0x4, -0x3d12017),
            _0x257ec6 = _0x32032f(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xc], 0xb, -0x1924661b),
            _0x1724c5 = _0x32032f(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xf], 0x10, 0x1fa27cf8),
            _0xdb4d2c = _0x32032f(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x2], 0x17, -0x3b53a99b),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3], 0x6, -0xbd6ddbc),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x7], 0xa, 0x432aff97),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xe], 0xf, -0x546bdc59),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x5], 0x15, -0x36c5fc7),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0xc], 0x6, 0x655b59c3),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0x3], 0xa, -0x70ef89ee),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0xa], 0xf, -0x644f153),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x1], 0x15, -0x7a7ba22f),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x8], 0x6, 0x6fa87e4f),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xf], 0xa, -0x1d31920),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x6], 0xf, -0x5cfebcec),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0xd], 0x15, 0x4e0811a1),
            _0x3e0c38 = _0x4b459d(_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6, _0x193f00[_0x1badc3 + 0x4], 0x6, -0x8ac817e),
            _0x257ec6 = _0x4b459d(_0x257ec6, _0x3e0c38, _0xdb4d2c, _0x1724c5, _0x193f00[_0x1badc3 + 0xb], 0xa, -1120211379),
            _0x1724c5 = _0x4b459d(_0x1724c5, _0x257ec6, _0x3e0c38, _0xdb4d2c, _0x193f00[_0x1badc3 + 0x2], 0xf, 0x2ad7d2bb),
            _0xdb4d2c = _0x4b459d(_0xdb4d2c, _0x1724c5, _0x257ec6, _0x3e0c38, _0x193f00[_0x1badc3 + 0x9], 0x15, -0x14792c01),
            _0x3e0c38 = _0x12e4a8(_0x3e0c38, _0x38ca59),
            _0xdb4d2c = _0x12e4a8(_0xdb4d2c, _0x431764),
            _0x1724c5 = _0x12e4a8(_0x1724c5, _0x43f1b4),
            _0x257ec6 = _0x12e4a8(_0x257ec6, _0x5722c0);
    return [_0x3e0c38, _0xdb4d2c, _0x1724c5, _0x257ec6];
}

function _0x48d200(_0x4b706e, _0x3c3a85, _0x111154, _0x311f9f, _0x5439cf, _0x38cac7, _0x26bd2e) {
    return _0xaaef84(_0x3c3a85 & _0x111154 | ~_0x3c3a85 & _0x311f9f, _0x4b706e, _0x3c3a85, _0x5439cf, _0x38cac7, _0x26bd2e);
}

function _0xaaef84(_0xaf3112, _0x2a165a, _0x532fb4, _0x10aa40, _0x41c4e7, _0x1cb4da) {
    return _0x12e4a8(_0x3634fc(_0x12e4a8(_0x12e4a8(_0x2a165a, _0xaf3112), _0x12e4a8(_0x10aa40, _0x1cb4da)), _0x41c4e7), _0x532fb4);
}

function _0x12e4a8(_0x7542c8, _0x5eada0) {
    var _0x41f81f = (0xffff & _0x7542c8) + (0xffff & _0x5eada0);
    return (_0x7542c8 >> 0x10) + (_0x5eada0 >> 0x10) + (_0x41f81f >> 0x10) << 0x10 | 0xffff & _0x41f81f;
}

function _0x3634fc(_0x5803ba, _0x1ce5b2) {
    return _0x5803ba << _0x1ce5b2 | _0x5803ba >>> 0x20 - _0x1ce5b2;
}

function _0x4b459d(_0x8d8f2a, _0x406d34, _0x53e7d7, _0x26c827, _0xec41ea, _0x52dead, _0x3f66e7) {
    return _0xaaef84(_0x53e7d7 ^ (_0x406d34 | ~_0x26c827), _0x8d8f2a, _0x406d34, _0xec41ea, _0x52dead, _0x3f66e7);
}

function _0x3180ec(_0x401705, _0x240e6a, _0x56b131, _0x5a5c20, _0x1f2a72, _0x2bfc1, _0x19741a) {
    return _0xaaef84(_0x240e6a & _0x5a5c20 | _0x56b131 & ~_0x5a5c20, _0x401705, _0x240e6a, _0x1f2a72, _0x2bfc1, _0x19741a);
}

function _0x32032f(_0x520fdf, _0x13921d, _0x1af9d5, _0x4a2311, _0xb6d40a, _0x1d58da, _0x361df0) {
    return _0xaaef84(_0x13921d ^ _0x1af9d5 ^ _0x4a2311, _0x520fdf, _0x13921d, _0xb6d40a, _0x1d58da, _0x361df0);
}

function getRM() {
    var _$pr = [];
    var f_list = [];
    var m;
    for (var i = 0; i < 5; i++) {
        var _$is = new Date().valueOf().toString()
        f_list.push(_$is)
        m = _0x474032(_$is)
        _$pr.push(m)
        delete _$Jy;
        delete _$tT;
        if (i === 3) {
            _$Jy = -405537848;
            _$tT = -660478335;
            delete _$6_;
            _$6_ = -389564586;
        } else {
            _$Jy = new Date().valueOf();
            _$tT = -717253467;
        }
    }
    var _$Ww = CryptoJS['enc']['Utf8']['parse'](_$pr['toString']());
    var _$qF = CryptoJS['enc']['Utf8']['parse'](btoa(_$is)['slice'](0x0, 0x10));
    var _0x29dd83 = CryptoJS['AES']['encrypt'](_$Ww, _$qF, {
        'mode': CryptoJS['mode']['ECB'],
        'padding': CryptoJS['pad']['Pkcs7']
    });
    var _$ss = _0x29dd83['toString']();

    var res = _$ss + ':::' + m + ':::' + f_list[0] + ':::' + _$is;
    return res;
}

//_0x474032(_0x12eaf3())
// getRM()

```
