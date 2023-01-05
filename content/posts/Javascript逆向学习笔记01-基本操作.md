---
title: "Javascript逆向学习笔记Ⅰ"
slug: "javascript-reverse-study-notes-01"
description: "这里是tricks和基操合集！"
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

