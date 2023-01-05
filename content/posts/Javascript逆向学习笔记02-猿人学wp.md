---
title: "Javascript逆向学习笔记Ⅱ"
slug: "javascript-reverse-study-notes-02"
description: "猿人学js逆向题wp"
date: 2023-01-05T22:46:43+08:00
categories: ["LTS"]
series: ["前端安全"]
tags: ["js", "wp"]
draft: false
toc: true
---

本文所有参考链接附在文末，如有错漏欢迎批评指正w

另：考虑到排版和阅读上的舒适度，将题目中**较长的**代码都做了折叠，点击左边小三角即可食用w

----

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

{{% spoiler "code" %}}

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

{{% /spoiler %}}

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

{{% spoiler "2.js" %}}

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

{{% /spoiler %}}

{{% spoiler "exp.py" %}}

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

{{% /spoiler %}}

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

{{% spoiler "varReload.py" %}}

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

{{% /spoiler %}}

最终实现这样的效果：

![image-20221218011540192](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218011540192.png)

### 调用解密函数

我们已经成功把[4]到[6]中调用解密函数的部分都把变量换成了字符串（参数还原），接下来我们把用到解密函数的部分进行还原，还原的方式是找到解密函数`$dbsm_0x42c3`被调用的位置，获取参数，调用函数，替换返回值

在ast中观察，函数调用节点是CallExpression，内部callee的name是`$dbsm_0x42c3`

![image-20230103140231617](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230103140231617.png)

编写对应代码

{{% spoiler "funcReload.py" %}}

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

{{% /spoiler %}}

有个蜜汁彩蛋？由于这里乱码部分心机的加了中文字符，导致即使open处用utf-8还是会导致execjs解析失败，所以要修改subprocess.py中的encoding参数也为utf-8（切记 用完之后改回来 不然会影响其它库的正常工作的）

执行完的效果则是这样

![image-20221218021235668](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221218021235668.png)

在新生成的js中已经找不到`$dbsm_0x42c3`的身影了

### 对象调用还原

#### 字符串拼接

字符串拼接部分的节点类型为BinaryExpression，且左右子节点都是Literal

{{% spoiler "concatString.py" %}}

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

{{% /spoiler %}}

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

{{% spoiler "propertyDelete.py" %}}

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

{{% /spoiler %}}

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

{{% spoiler "propertyReload.py" %}}

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

{{% /spoiler %}}

最终效果

![image-20221219021808232](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20221219021808232.png)

### 分支流程判断

上一步生成的js文件中有很多多余的分支判断，比如上图中的`if('QcUHF' !== 'QcUHF')`，这样的bool判断是可以提前处理掉的，我们只需要递归找到这样的节点，判断其表达式中的bool值并决定是否保留此分支

代码如下：

{{% spoiler "ifReload" %}}

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

{{% /spoiler %}}

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

{{% spoiler "flowFlatten" %}}

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

{{% /spoiler %}}

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

{{% spoiler "end.js" %}}

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

{{% /spoiler %}}

### 解题

辛苦得到的美化后的js可以直接被调用

{{% spoiler "exp.py" %}}

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

{{% /spoiler %}}

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

- 元组

需要用自定义的items方法，借用元组的格式来固定顺序；详见[此处](https://zhuanlan.zhihu.com/p/360335607)

- requests.session()

上一种方式还是有点麻烦，我这里用Session()对象也可以确保headers顺序不变

{{% spoiler "exp.py" %}}

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
session = requests.session()
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

{{% /spoiler %}}

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

{{% spoiler "exp.py" %}}

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

{{% /spoiler %}}

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

{{% spoiler "getM.js" %}}

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

{{% /spoiler %}}

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

{{% spoiler "exp.py" %}}

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

{{% /spoiler %}}

{{% spoiler "setCookie.js" %}}

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

{{% /spoiler %}}

## 猿人学6 - js混淆 回溯

> https://match.yuanrenxue.com/match/6
>
> 题目要求：采集全部5页的彩票数据，计算**全部**中奖的**总**金额（包含一、二、三等奖）

惯例抓包，这次发生变化的是get的m和q参数，m是个加密的，但q很有规律

![image-20230105095125009](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230105095125009.png)

即`点击次数-timestamp`，通过这样的方式限制了重放，同时最多点5次，当q中有第6个记录时就会触发风控

找到这部分代码

![image-20230105100014864](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230105100014864.png)

m参数由r()生成，把相关的函数都拖出来（也就是整个js文件），加上必要的`window={}`，结果执行一下返回的是false？调了一会儿不知道原因，查了wp发现问题在于有两段js fuck，我们直接把它们换成执行的结果，并且把`window={}`换成`window=global`

{{% spoiler "exp.js" %}}

```python
import time

import requests
import subprocess
from functools import partial

subprocess.Popen = partial(subprocess.Popen, encoding='utf-8')

with open('w.js', 'r', encoding='utf-8') as f:
    import execjs
    ctx = execjs.compile(f.read())

q = ''
money = []
for i in range(1, 6):
    value = ctx.call('getResult', i).split(':::')
    params = {'page': i, 'm': value[0], 'q': value[1]}
    res = requests.get(f'https://match.yuanrenxue.com/api/match/6', params=params, headers={'User-Agent': 'yuanrenxue.project'}).json()
    data = [int(i['value']) * 24 for i in res['data']]
    money.extend(data)
    time.sleep(0.5)

print(sum(money))
# 6883344
```

{{% /spoiler %}}

{{% spoiler "w.js" %}}

```js
var _n;
window = global
window.o = 1
navigator = {};
!function (i) {
    var s = {};

    function n(t) {
        if (s[t])
            return s[t].exports;
        var e = s[t] = {
            i: t,
            l: !1,
            exports: {}
        };
        return i[t].call(e.exports, e, e.exports, n),
            e.l = !0,
            e.exports
    }

    _n = n;
}({
    encrypt: function (t, e, i) {
        var s, n, r;
        (r = function (t, e, i) {
            n = [e],
            (r = "function" == typeof (s = function (t) {
                    function b(t, e, i) {
                        null != t && ("number" == typeof t ? this.fromNumber(t, e, i) : null == e && "string" != typeof t ? this.fromString(t, 256) : this.fromString(t, e))
                    }

                    function y() {
                        return new b(null)
                    }

                    function e(t, e, i, s, n, r) {
                        for (; --r >= 0;) {
                            var o = e * this[t++] + i[s] + n;
                            n = Math.floor(o / 67108864),
                                i[s++] = 67108863 & o
                        }
                        return n
                    }

                    function i(t, e, i, s, n, r) {
                        for (var o = 32767 & e, a = e >> 15; --r >= 0;) {
                            var c = 32767 & this[t]
                                , l = this[t++] >> 15
                                , u = a * c + l * o;
                            c = o * c + ((32767 & u) << 15) + i[s] + (1073741823 & n),
                                n = (c >>> 30) + (u >>> 15) + a * l + (n >>> 30),
                                i[s++] = 1073741823 & c
                        }
                        return n
                    }

                    function s(t, e, i, s, n, r) {
                        for (var o = 16383 & e, a = e >> 14; --r >= 0;) {
                            var c = 16383 & this[t]
                                , l = this[t++] >> 14
                                , u = a * c + l * o;
                            c = o * c + ((16383 & u) << 14) + i[s] + n,
                                n = (c >> 28) + (u >> 14) + a * l,
                                i[s++] = 268435455 & c
                        }
                        return n
                    }

                    function c(t) {
                        return Te.charAt(t)
                    }

                    function l(t, e) {
                        var i = Ie[t.charCodeAt(e)];
                        return null == i ? -1 : i
                    }

                    function p(t) {
                        for (var e = this.t - 1; e >= 0; --e)
                            t[e] = this[e];
                        t.t = this.t,
                            t.s = this.s
                    }

                    function n(t) {
                        this.t = 1,
                            this.s = 0 > t ? -1 : 0,
                            t > 0 ? this[0] = t : -1 > t ? this[0] = t + this.DV : this.t = 0
                    }

                    function m(t) {
                        var e = y();
                        return e.fromInt(t),
                            e
                    }

                    function h(t, e) {
                        var i;
                        if (16 == e)
                            i = 4;
                        else if (8 == e)
                            i = 3;
                        else if (256 == e)
                            i = 8;
                        else if (2 == e)
                            i = 1;
                        else if (32 == e)
                            i = 5;
                        else {
                            if (4 != e)
                                return void this.fromRadix(t, e);
                            i = 2
                        }
                        this.t = 0,
                            this.s = 0;
                        for (var s = t.length, n = !1, r = 0; --s >= 0;) {
                            var o = 8 == i ? 255 & t[s] : l(t, s);
                            0 > o ? "-" == t.charAt(s) && (n = !0) : (n = !1,
                                0 == r ? this[this.t++] = o : r + i > this.DB ? (this[this.t - 1] |= (o & (1 << this.DB - r) - 1) << r,
                                    this[this.t++] = o >> this.DB - r) : this[this.t - 1] |= o << r,
                                r += i,
                            r >= this.DB && (r -= this.DB))
                        }
                        8 == i && 0 != (128 & t[0]) && (this.s = -1,
                        r > 0 && (this[this.t - 1] |= (1 << this.DB - r) - 1 << r)),
                            this.clamp(),
                        n && b.ZERO.subTo(this, this)
                    }

                    function r() {
                        for (var t = this.s & this.DM; this.t > 0 && this[this.t - 1] == t;)
                            --this.t
                    }

                    function o(t) {
                        if (this.s < 0)
                            return "-" + this.negate().toString(t);
                        var e;
                        if (16 == t)
                            e = 4;
                        else if (8 == t)
                            e = 3;
                        else if (2 == t)
                            e = 1;
                        else if (32 == t)
                            e = 5;
                        else {
                            if (4 != t)
                                return this.toRadix(t);
                            e = 2
                        }
                        var i, s = (1 << e) - 1, n = !1, r = "", o = this.t, a = this.DB - o * this.DB % e;
                        if (o-- > 0)
                            for (a < this.DB && (i = this[o] >> a) > 0 && (n = !0,
                                r = c(i)); o >= 0;)
                                e > a ? (i = (this[o] & (1 << a) - 1) << e - a,
                                    i |= this[--o] >> (a += this.DB - e)) : (i = this[o] >> (a -= e) & s,
                                0 >= a && (a += this.DB,
                                    --o)),
                                i > 0 && (n = !0),
                                n && (r += c(i));
                        return n ? r : "0"
                    }

                    function f() {
                        var t = y();
                        return b.ZERO.subTo(this, t),
                            t
                    }

                    function a() {
                        return this.s < 0 ? this.negate() : this
                    }

                    function u(t) {
                        var e = this.s - t.s;
                        if (0 != e)
                            return e;
                        var i = this.t;
                        if (e = i - t.t,
                        0 != e)
                            return this.s < 0 ? -e : e;
                        for (; --i >= 0;)
                            if (0 != (e = this[i] - t[i]))
                                return e;
                        return 0
                    }

                    function w(t) {
                        if (t === 65537) {
                            t = 60155
                        } else {
                            t = 60110
                        }
                        var e, i = 1;
                        return 0 != (e = t >>> 16) && (t = e,
                            i += 16),
                        0 != (e = t >> 8) && (t = e,
                            i += 8),
                        0 != (e = t >> 4) && (t = e,
                            i += 4),
                        0 != (e = t >> 2) && (t = e,
                            i += 2),
                        0 != (e = t >> 1) && (t = e,
                            i += 1),
                            i
                    }

                    function d() {
                        return this.t <= 0 ? 0 : this.DB * (this.t - 1) + w(this[this.t - 1] ^ this.s & this.DM)
                    }

                    function g(t, e) {
                        var i;
                        for (i = this.t - 1; i >= 0; --i)
                            e[i + t] = this[i];
                        for (i = t - 1; i >= 0; --i)
                            e[i] = 0;
                        e.t = this.t + t,
                            e.s = this.s
                    }

                    function _(t, e) {
                        for (var i = t; i < this.t; ++i)
                            e[i - t] = this[i];
                        e.t = Math.max(this.t - t, 0),
                            e.s = this.s
                    }

                    function k(t, e) {
                        var i, s = t % this.DB, n = this.DB - s, r = (1 << n) - 1, o = Math.floor(t / this.DB),
                            a = this.s << s & this.DM;
                        for (i = this.t - 1; i >= 0; --i)
                            e[i + o + 1] = this[i] >> n | a,
                                a = (this[i] & r) << s;
                        for (i = o - 1; i >= 0; --i)
                            e[i] = 0;
                        e[o] = a,
                            e.t = this.t + o + 1,
                            e.s = this.s,
                            e.clamp()
                    }

                    function x(t, e) {
                        e.s = this.s;
                        var i = Math.floor(t / this.DB);
                        if (i >= this.t)
                            return void (e.t = 0);
                        var s = t % this.DB
                            , n = this.DB - s
                            , r = (1 << s) - 1;
                        e[0] = this[i] >> s;
                        for (var o = i + 1; o < this.t; ++o)
                            e[o - i - 1] |= (this[o] & r) << n,
                                e[o - i] = this[o] >> s;
                        s > 0 && (e[this.t - i - 1] |= (this.s & r) << n),
                            e.t = this.t - i,
                            e.clamp()
                    }

                    function D(t, e) {
                        for (var i = 0, s = 0, n = Math.min(t.t, this.t); n > i;)
                            s += this[i] - t[i],
                                e[i++] = s & this.DM,
                                s >>= this.DB;
                        if (t.t < this.t) {
                            for (s -= t.s; i < this.t;)
                                s += this[i],
                                    e[i++] = s & this.DM,
                                    s >>= this.DB;
                            s += this.s
                        } else {
                            for (s += this.s; i < t.t;)
                                s -= t[i],
                                    e[i++] = s & this.DM,
                                    s >>= this.DB;
                            s -= t.s
                        }
                        e.s = 0 > s ? -1 : 0,
                            -1 > s ? e[i++] = this.DV + s : s > 0 && (e[i++] = s),
                            e.t = i,
                            e.clamp()
                    }

                    function S(t, e) {
                        var i = this.abs()
                            , s = t.abs()
                            , n = i.t;
                        for (e.t = n + s.t; --n >= 0;)
                            e[n] = 0;
                        for (n = 0; n < s.t; ++n)
                            e[n + i.t] = i.am(0, s[n], e, n, 0, i.t);
                        e.s = 0,
                            e.clamp(),
                        this.s != t.s && b.ZERO.subTo(e, e)
                    }

                    function C(t) {
                        for (var e = this.abs(), i = t.t = 2 * e.t; --i >= 0;)
                            t[i] = 0;
                        for (i = 0; i < e.t - 1; ++i) {
                            var s = e.am(i, e[i], t, 2 * i, 0, 1);
                            (t[i + e.t] += e.am(i + 1, 2 * e[i], t, 2 * i + 1, s, e.t - i - 1)) >= e.DV && (t[i + e.t] -= e.DV,
                                t[i + e.t + 1] = 1)
                        }
                        t.t > 0 && (t[t.t - 1] += e.am(i, e[i], t, 2 * i, 0, 1)),
                            t.s = 0,
                            t.clamp()
                    }

                    function T(t, e, i) {
                        var s = t.abs();
                        if (!(s.t <= 0)) {
                            var n = this.abs();
                            if (n.t < s.t)
                                return null != e && e.fromInt(0),
                                    void (null != i && this.copyTo(i));
                            null == i && (i = y());
                            var r = y()
                                , o = this.s
                                , a = t.s
                                , c = this.DB - w(s[s.t - 1]);
                            c > 0 ? (s.lShiftTo(c, r),
                                n.lShiftTo(c, i)) : (s.copyTo(r),
                                n.copyTo(i));
                            var l = r.t
                                , u = r[l - 1];
                            if (0 != u) {
                                var d = u * (1 << this.F1) + (l > 1 ? r[l - 2] >> this.F2 : 0)
                                    , p = this.FV / d
                                    , h = (1 << this.F1) / d
                                    , f = 1 << this.F2
                                    , g = i.t
                                    , m = g - l
                                    , v = null == e ? y() : e;
                                for (r.dlShiftTo(m, v),
                                     i.compareTo(v) >= 0 && (i[i.t++] = 1,
                                         i.subTo(v, i)),
                                         b.ONE.dlShiftTo(l, v),
                                         v.subTo(r, r); r.t < l;)
                                    r[r.t++] = 0;
                                for (; --m >= 0;) {
                                    var _ = i[--g] == u ? this.DM : Math.floor(i[g] * p + (i[g - 1] + f) * h);
                                    if ((i[g] += r.am(0, _, i, m, 0, l)) < _)
                                        for (r.dlShiftTo(m, v),
                                                 i.subTo(v, i); i[g] < --_;)
                                            i.subTo(v, i)
                                }
                                null != e && (i.drShiftTo(l, e),
                                o != a && b.ZERO.subTo(e, e)),
                                    i.t = l,
                                    i.clamp(),
                                c > 0 && i.rShiftTo(c, i),
                                0 > o && b.ZERO.subTo(i, i)
                            }
                        }
                    }

                    function I(t) {
                        var e = y();
                        return this.abs().divRemTo(t, null, e),
                        this.s < 0 && e.compareTo(b.ZERO) > 0 && t.subTo(e, e),
                            e
                    }

                    function $(t) {
                        this.m = t
                    }

                    function P(t) {
                        return t.s < 0 || t.compareTo(this.m) >= 0 ? t.mod(this.m) : t
                    }

                    function R(t) {
                        return t
                    }

                    function A(t) {
                        t.divRemTo(this.m, null, t)
                    }

                    function E(t, e, i) {
                        t.multiplyTo(e, i),
                            this.reduce(i)
                    }

                    function M(t, e) {
                        t.squareTo(e),
                            this.reduce(e)
                    }

                    function N() {
                        if (this.t < 1)
                            return 0;
                        var t = this[0];
                        if (0 == (1 & t))
                            return 0;
                        var e = 3 & t;
                        return e = e * (2 - (15 & t) * e) & 15,
                            e = e * (2 - (255 & t) * e) & 255,
                            e = e * (2 - ((65535 & t) * e & 65535)) & 65535,
                            e = e * (2 - t * e % this.DV) % this.DV,
                            e > 0 ? this.DV - e : -e
                    }

                    function O(t) {
                        this.m = t,
                            this.mp = t.invDigit(),
                            this.mpl = 32767 & this.mp,
                            this.mph = this.mp >> 15,
                            this.um = (1 << t.DB - 15) - 1,
                            this.mt2 = 2 * t.t
                    }

                    function B(t) {
                        var e = y();
                        return t.abs().dlShiftTo(this.m.t, e),
                            e.divRemTo(this.m, null, e),
                        t.s < 0 && e.compareTo(b.ZERO) > 0 && this.m.subTo(e, e),
                            e
                    }

                    function j(t) {
                        var e = y();
                        return t.copyTo(e),
                            this.reduce(e),
                            e
                    }

                    function L(t) {
                        for (; t.t <= this.mt2;)
                            t[t.t++] = 0;
                        for (var e = 0; e < this.m.t; ++e) {
                            var i = 32767 & t[e]
                                , s = i * this.mpl + ((i * this.mph + (t[e] >> 15) * this.mpl & this.um) << 15) & t.DM;
                            for (i = e + this.m.t,
                                     t[i] += this.m.am(0, s, t, e, 0, this.m.t); t[i] >= t.DV;)
                                t[i] -= t.DV,
                                    t[++i]++
                        }
                        t.clamp(),
                            t.drShiftTo(this.m.t, t),
                        t.compareTo(this.m) >= 0 && t.subTo(this.m, t)
                    }

                    function F(t, e) {
                        t.squareTo(e),
                            this.reduce(e)
                    }

                    function K(t, e, i) {
                        t.multiplyTo(e, i),
                            this.reduce(i)
                    }

                    function U() {
                        return 0 == (this.t > 0 ? 1 & this[0] : this.s)
                    }

                    function V(t, e) {
                        if (t > 4294967295 || 1 > t)
                            return b.ONE;
                        var i = y()
                            , s = y()
                            , n = e.convert(this)
                            , r = w(t) - 1;
                        for (n.copyTo(i); --r >= 0;)
                            if (e.sqrTo(i, s),
                            (t & 1 << r) > 0)
                                e.mulTo(s, n, i);
                            else {
                                var o = i;
                                i = s,
                                    s = o
                            }
                        return e.revert(i)
                    }


                    function z(t, e) {
                        var i;
                        return i = 256 > t || e.isEven() ? new $(e) : new O(e),
                            this.exp(t, i)
                    }

                    function q() {
                        var t = y();
                        return this.copyTo(t),
                            t
                    }

                    function H() {
                        if (this.s < 0) {
                            if (1 == this.t)
                                return this[0] - this.DV;
                            if (0 == this.t)
                                return -1
                        } else {
                            if (1 == this.t)
                                return this[0];
                            if (0 == this.t)
                                return 0
                        }
                        return (this[1] & (1 << 32 - this.DB) - 1) << this.DB | this[0]
                    }

                    function J() {
                        return 0 == this.t ? this.s : this[0] << 24 >> 24
                    }

                    function G() {
                        return 0 == this.t ? this.s : this[0] << 16 >> 16
                    }

                    function Y(t) {
                        return Math.floor(Math.LN2 * this.DB / Math.log(t))
                    }

                    function W() {
                        return this.s < 0 ? -1 : this.t <= 0 || 1 == this.t && this[0] <= 0 ? 0 : 1
                    }

                    function Z(t) {
                        if (null == t && (t = 10),
                        0 == this.signum() || 2 > t || t > 36)
                            return "0";
                        var e = this.chunkSize(t)
                            , i = Math.pow(t, e)
                            , s = m(i)
                            , n = y()
                            , r = y()
                            , o = "";
                        for (this.divRemTo(s, n, r); n.signum() > 0;)
                            o = (i + r.intValue()).toString(t).substr(1) + o,
                                n.divRemTo(s, n, r);
                        return r.intValue().toString(t) + o
                    }

                    function Q(t, e) {
                        this.fromInt(0),
                        null == e && (e = 10);
                        for (var i = this.chunkSize(e), s = Math.pow(e, i), n = !1, r = 0, o = 0, a = 0; a < t.length; ++a) {
                            var c = l(t, a);
                            0 > c ? "-" == t.charAt(a) && 0 == this.signum() && (n = !0) : (o = e * o + c,
                            ++r >= i && (this.dMultiply(s),
                                this.dAddOffset(o, 0),
                                r = 0,
                                o = 0))
                        }
                        r > 0 && (this.dMultiply(Math.pow(e, r)),
                            this.dAddOffset(o, 0)),
                        n && b.ZERO.subTo(this, this)
                    }

                    function X(t, e, i) {
                        if ("number" == typeof e)
                            if (2 > t)
                                this.fromInt(1);
                            else
                                for (this.fromNumber(t, i),
                                     this.testBit(t - 1) || this.bitwiseTo(b.ONE.shiftLeft(t - 1), at, this),
                                     this.isEven() && this.dAddOffset(1, 0); !this.isProbablePrime(e);)
                                    this.dAddOffset(2, 0),
                                    this.bitLength() > t && this.subTo(b.ONE.shiftLeft(t - 1), this);
                        else {
                            var s = []
                                , n = 7 & t;
                            s.length = (t >> 3) + 1,
                                e.nextBytes(s),
                                n > 0 ? s[0] &= (1 << n) - 1 : s[0] = 0,
                                this.fromString(s, 256)
                        }
                    }

                    function tt() {
                        var t = this.t
                            , e = [];
                        e[0] = this.s;
                        var i, s = this.DB - t * this.DB % 8, n = 0;
                        if (t-- > 0)
                            for (s < this.DB && (i = this[t] >> s) != (this.s & this.DM) >> s && (e[n++] = i | this.s << this.DB - s); t >= 0;)
                                8 > s ? (i = (this[t] & (1 << s) - 1) << 8 - s,
                                    i |= this[--t] >> (s += this.DB - 8)) : (i = this[t] >> (s -= 8) & 255,
                                0 >= s && (s += this.DB,
                                    --t)),
                                0 != (128 & i) && (i |= -256),
                                0 == n && (128 & this.s) != (128 & i) && ++n,
                                (n > 0 || i != this.s) && (e[n++] = i);
                        return e
                    }

                    function et(t) {
                        return 0 == this.compareTo(t)
                    }

                    function it(t) {
                        return this.compareTo(t) < 0 ? this : t
                    }

                    function st(t) {
                        return this.compareTo(t) > 0 ? this : t
                    }

                    function nt(t, e, i) {
                        var s, n, r = Math.min(t.t, this.t);
                        for (s = 0; r > s; ++s)
                            i[s] = e(this[s], t[s]);
                        if (t.t < this.t) {
                            for (n = t.s & this.DM,
                                     s = r; s < this.t; ++s)
                                i[s] = e(this[s], n);
                            i.t = this.t
                        } else {
                            for (n = this.s & this.DM,
                                     s = r; s < t.t; ++s)
                                i[s] = e(n, t[s]);
                            i.t = t.t
                        }
                        i.s = e(this.s, t.s),
                            i.clamp()
                    }

                    function rt(t, e) {
                        return t & e
                    }

                    function ot(t) {
                        var e = y();
                        return this.bitwiseTo(t, rt, e),
                            e
                    }

                    function at(t, e) {
                        return t | e
                    }

                    function ct(t) {
                        var e = y();
                        return this.bitwiseTo(t, at, e),
                            e
                    }

                    function lt(t, e) {
                        return t ^ e
                    }

                    function ut(t) {
                        var e = y();
                        return this.bitwiseTo(t, lt, e),
                            e
                    }

                    function dt(t, e) {
                        return t & ~e
                    }

                    function pt(t) {
                        var e = y();
                        return this.bitwiseTo(t, dt, e),
                            e
                    }

                    function ht() {
                        for (var t = y(), e = 0; e < this.t; ++e)
                            t[e] = this.DM & ~this[e];
                        return t.t = this.t,
                            t.s = ~this.s,
                            t
                    }

                    function ft(t) {
                        var e = y();
                        return 0 > t ? this.rShiftTo(-t, e) : this.lShiftTo(t, e),
                            e
                    }

                    function gt(t) {
                        var e = y();
                        return 0 > t ? this.lShiftTo(-t, e) : this.rShiftTo(t, e),
                            e
                    }

                    function mt(t) {
                        if (0 == t)
                            return -1;
                        var e = 0;
                        return 0 == (65535 & t) && (t >>= 16,
                            e += 16),
                        0 == (255 & t) && (t >>= 8,
                            e += 8),
                        0 == (15 & t) && (t >>= 4,
                            e += 4),
                        0 == (3 & t) && (t >>= 2,
                            e += 2),
                        0 == (1 & t) && ++e,
                            e
                    }

                    function vt() {
                        for (var t = 0; t < this.t; ++t)
                            if (0 != this[t])
                                return t * this.DB + mt(this[t]);
                        return this.s < 0 ? this.t * this.DB : -1
                    }

                    function _t(t) {
                        for (var e = 0; 0 != t;)
                            t &= t - 1,
                                ++e;
                        return e
                    }

                    function bt() {
                        for (var t = 0, e = this.s & this.DM, i = 0; i < this.t; ++i)
                            t += _t(this[i] ^ e);
                        return t
                    }

                    function yt(t) {
                        var e = Math.floor(t / this.DB);
                        return e >= this.t ? 0 != this.s : 0 != (this[e] & 1 << t % this.DB)
                    }

                    function wt(t, e) {
                        var i = b.ONE.shiftLeft(t);
                        return this.bitwiseTo(i, e, i),
                            i
                    }

                    function kt(t) {
                        return this.changeBit(t, at)
                    }

                    function xt(t) {
                        return this.changeBit(t, dt)
                    }

                    function Dt(t) {
                        return this.changeBit(t, lt)
                    }

                    function St(t, e) {
                        for (var i = 0, s = 0, n = Math.min(t.t, this.t); n > i;)
                            s += this[i] + t[i],
                                e[i++] = s & this.DM,
                                s >>= this.DB;
                        if (t.t < this.t) {
                            for (s += t.s; i < this.t;)
                                s += this[i],
                                    e[i++] = s & this.DM,
                                    s >>= this.DB;
                            s += this.s
                        } else {
                            for (s += this.s; i < t.t;)
                                s += t[i],
                                    e[i++] = s & this.DM,
                                    s >>= this.DB;
                            s += t.s
                        }
                        e.s = 0 > s ? -1 : 0,
                            s > 0 ? e[i++] = s : -1 > s && (e[i++] = this.DV + s),
                            e.t = i,
                            e.clamp()
                    }

                    function Ct(t) {
                        var e = y();
                        return this.addTo(t, e),
                            e
                    }

                    function Tt(t) {
                        var e = y();
                        return this.subTo(t, e),
                            e
                    }

                    function It(t) {
                        var e = y();
                        return this.multiplyTo(t, e),
                            e
                    }

                    function $t() {
                        var t = y();
                        return this.squareTo(t),
                            t
                    }

                    function Pt(t) {
                        var e = y();
                        return this.divRemTo(t, e, null),
                            e
                    }

                    function Rt(t) {
                        var e = y();
                        return this.divRemTo(t, null, e),
                            e
                    }

                    function At(t) {
                        var e = y()
                            , i = y();
                        return this.divRemTo(t, e, i),
                            [e, i]
                    }

                    function Et(t) {
                        this[this.t] = this.am(0, t - 1, this, 0, 0, this.t),
                            ++this.t,
                            this.clamp()
                    }

                    function Mt(t, e) {
                        if (0 != t) {
                            for (; this.t <= e;)
                                this[this.t++] = 0;
                            for (this[e] += t; this[e] >= this.DV;)
                                this[e] -= this.DV,
                                ++e >= this.t && (this[this.t++] = 0),
                                    ++this[e]
                        }
                    }

                    function Nt() {
                    }

                    function Ot(t) {
                        return t
                    }

                    function Bt(t, e, i) {
                        t.multiplyTo(e, i)
                    }

                    function jt(t, e) {
                        t.squareTo(e)
                    }

                    function Lt(t) {
                        return this.exp(t, new Nt)
                    }

                    function Ft(t, e, i) {
                        var s = Math.min(this.t + t.t, e);
                        for (i.s = 0,
                                 i.t = s; s > 0;)
                            i[--s] = 0;
                        var n;
                        for (n = i.t - this.t; n > s; ++s)
                            i[s + this.t] = this.am(0, t[s], i, s, 0, this.t);
                        for (n = Math.min(t.t, e); n > s; ++s)
                            this.am(0, t[s], i, s, 0, e - s);
                        i.clamp()
                    }

                    function Kt(t, e, i) {
                        --e;
                        var s = i.t = this.t + t.t - e;
                        for (i.s = 0; --s >= 0;)
                            i[s] = 0;
                        for (s = Math.max(e - this.t, 0); s < t.t; ++s)
                            i[this.t + s - e] = this.am(e - s, t[s], i, 0, 0, this.t + s - e);
                        i.clamp(),
                            i.drShiftTo(1, i)
                    }

                    function Ut(t) {
                        this.r2 = y(),
                            this.q3 = y(),
                            b.ONE.dlShiftTo(2 * t.t, this.r2),
                            this.mu = this.r2.divide(t),
                            this.m = t
                    }

                    function Vt(t) {
                        if (t.s < 0 || t.t > 2 * this.m.t)
                            return t.mod(this.m);
                        if (t.compareTo(this.m) < 0)
                            return t;
                        var e = y();
                        return t.copyTo(e),
                            this.reduce(e),
                            e
                    }

                    function zt(t) {
                        return t
                    }

                    function qt(t) {
                        for (t.drShiftTo(this.m.t - 1, this.r2),
                             t.t > this.m.t + 1 && (t.t = this.m.t + 1,
                                 t.clamp()),
                                 this.mu.multiplyUpperTo(this.r2, this.m.t + 1, this.q3),
                                 this.m.multiplyLowerTo(this.q3, this.m.t + 1, this.r2); t.compareTo(this.r2) < 0;)
                            t.dAddOffset(1, this.m.t + 1);
                        for (t.subTo(this.r2, t); t.compareTo(this.m) >= 0;)
                            t.subTo(this.m, t)
                    }

                    function Ht(t, e) {
                        t.squareTo(e),
                            this.reduce(e)
                    }

                    function Jt(t, e, i) {
                        t.multiplyTo(e, i),
                            this.reduce(i)
                    }

                    function Gt(t, e) {
                        var i, s, n = t.bitLength(), r = m(1);
                        if (0 >= n)
                            return r;
                        i = 18 > n ? 1 : 48 > n ? 3 : 144 > n ? 4 : 768 > n ? 5 : 6,
                            s = 8 > n ? new $(e) : e.isEven() ? new Ut(e) : new O(e);
                        var o = []
                            , a = 3
                            , c = i - 1
                            , l = (1 << i) - 1;
                        if (o[1] = s.convert(this),
                        i > 1) {
                            var u = y();
                            for (s.sqrTo(o[1], u); l >= a;)
                                o[a] = y(),
                                    s.mulTo(u, o[a - 2], o[a]),
                                    a += 2
                        }
                        var d, p, h = t.t - 1, f = !0, g = y();
                        for (n = w(t[h]) - 1; h >= 0;) {
                            for (n >= c ? d = t[h] >> n - c & l : (d = (t[h] & (1 << n + 1) - 1) << c - n,
                            h > 0 && (d |= t[h - 1] >> this.DB + n - c)),
                                     a = i; 0 == (1 & d);)
                                d >>= 1,
                                    --a;
                            if ((n -= a) < 0 && (n += this.DB,
                                --h),
                                f)
                                o[d].copyTo(r),
                                    f = !1;
                            else {
                                for (; a > 1;)
                                    s.sqrTo(r, g),
                                        s.sqrTo(g, r),
                                        a -= 2;
                                a > 0 ? s.sqrTo(r, g) : (p = r,
                                    r = g,
                                    g = p),
                                    s.mulTo(g, o[d], r)
                            }
                            for (; h >= 0 && 0 == (t[h] & 1 << n);)
                                s.sqrTo(r, g),
                                    p = r,
                                    r = g,
                                    g = p,
                                --n < 0 && (n = this.DB - 1,
                                    --h)
                        }
                        return s.revert(r)
                    }

                    function Yt(t) {
                        var e = this.s < 0 ? this.negate() : this.clone()
                            , i = t.s < 0 ? t.negate() : t.clone();
                        if (e.compareTo(i) < 0) {
                            var s = e;
                            e = i,
                                i = s
                        }
                        var n = e.getLowestSetBit()
                            , r = i.getLowestSetBit();
                        if (0 > r)
                            return e;
                        for (r > n && (r = n),
                             r > 0 && (e.rShiftTo(r, e),
                                 i.rShiftTo(r, i)); e.signum() > 0;)
                            (n = e.getLowestSetBit()) > 0 && e.rShiftTo(n, e),
                            (n = i.getLowestSetBit()) > 0 && i.rShiftTo(n, i),
                                e.compareTo(i) >= 0 ? (e.subTo(i, e),
                                    e.rShiftTo(1, e)) : (i.subTo(e, i),
                                    i.rShiftTo(1, i));
                        return r > 0 && i.lShiftTo(r, i),
                            i
                    }

                    function Wt(t) {
                        if (0 >= t)
                            return 0;
                        var e = this.DV % t
                            , i = this.s < 0 ? t - 1 : 0;
                        if (this.t > 0)
                            if (0 == e)
                                i = this[0] % t;
                            else
                                for (var s = this.t - 1; s >= 0; --s)
                                    i = (e * i + this[s]) % t;
                        return i
                    }

                    function Zt(t) {
                        var e = t.isEven();
                        if (this.isEven() && e || 0 == t.signum())
                            return b.ZERO;
                        for (var i = t.clone(), s = this.clone(), n = m(1), r = m(0), o = m(0), a = m(1); 0 != i.signum();) {
                            for (; i.isEven();)
                                i.rShiftTo(1, i),
                                    e ? (n.isEven() && r.isEven() || (n.addTo(this, n),
                                        r.subTo(t, r)),
                                        n.rShiftTo(1, n)) : r.isEven() || r.subTo(t, r),
                                    r.rShiftTo(1, r);
                            for (; s.isEven();)
                                s.rShiftTo(1, s),
                                    e ? (o.isEven() && a.isEven() || (o.addTo(this, o),
                                        a.subTo(t, a)),
                                        o.rShiftTo(1, o)) : a.isEven() || a.subTo(t, a),
                                    a.rShiftTo(1, a);
                            i.compareTo(s) >= 0 ? (i.subTo(s, i),
                            e && n.subTo(o, n),
                                r.subTo(a, r)) : (s.subTo(i, s),
                            e && o.subTo(n, o),
                                a.subTo(r, a))
                        }
                        return 0 != s.compareTo(b.ONE) ? b.ZERO : a.compareTo(t) >= 0 ? a.subtract(t) : a.signum() < 0 ? (a.addTo(t, a),
                            a.signum() < 0 ? a.add(t) : a) : a
                    }

                    function Qt(t) {
                        var e, i = this.abs();
                        if (1 == i.t && i[0] <= $e[$e.length - 1]) {
                            for (e = 0; e < $e.length; ++e)
                                if (i[0] == $e[e])
                                    return !0;
                            return !1
                        }
                        if (i.isEven())
                            return !1;
                        for (e = 1; e < $e.length;) {
                            for (var s = $e[e], n = e + 1; n < $e.length && Pe > s;)
                                s *= $e[n++];
                            for (s = i.modInt(s); n > e;)
                                if (s % $e[e++] == 0)
                                    return !1
                        }
                        return i.millerRabin(t)
                    }

                    function Xt(t) {
                        var e = this.subtract(b.ONE)
                            , i = e.getLowestSetBit();
                        if (0 >= i)
                            return !1;
                        var s = e.shiftRight(i);
                        t = t + 1 >> 1,
                        t > $e.length && (t = $e.length);
                        for (var n = y(), r = 0; t > r; ++r) {
                            var o = n.modPow(s, this);
                            if (0 != o.compareTo(b.ONE) && 0 != o.compareTo(e)) {
                                for (var a = 1; a++ < i && 0 != o.compareTo(e);)
                                    if (o = o.modPowInt(2, this),
                                    0 == o.compareTo(b.ONE))
                                        return !1;
                                if (0 != o.compareTo(e))
                                    return !1
                            }
                        }
                        return !0
                    }

                    function te() {
                        this.i = 0,
                            this.j = 0,
                            this.S = []
                    }

                    function ee(t) {
                        var e, i, s;
                        for (e = 0; 256 > e; ++e)
                            this.S[e] = e;
                        for (i = 0,
                                 e = 0; 256 > e; ++e)
                            i = i + this.S[e] + t[e % t.length] & 255,
                                s = this.S[e],
                                this.S[e] = this.S[i],
                                this.S[i] = s;
                        this.i = 0,
                            this.j = 0
                    }

                    function ie() {
                        var t;
                        return this.i = this.i + 1 & 255,
                            this.j = this.j + this.S[this.i] & 255,
                            t = this.S[this.i],
                            this.S[this.i] = this.S[this.j],
                            this.S[this.j] = t,
                            this.S[t + this.S[this.i] & 255]
                    }

                    function se() {
                        return new te
                    }

                    function ne() {
                        if (null == Re) {
                            for (Re = se(); Me > Ee;) {
                                Ae[Ee++] = 255 & t
                            }
                            for (Re.init(Ae),
                                     Ee = 0; Ee < Ae.length; ++Ee)
                                Ae[Ee] = 0;
                            Ee = 0
                        }
                        return Re.next()
                    }

                    function re(t) {
                        var e;
                        for (e = 0; e < t.length; ++e)
                            t[e] = ne()
                    }

                    function oe() {
                    }

                    function ae(t, e) {
                        return new b(t, e)
                    }

                    function ce(t, e) {
                        if (e < t.length + 11)
                            return console.error("Message too long for RSA"),
                                null;
                        for (var i = [], s = t.length - 1; s >= 0 && e > 0;) {
                            var n = t.charCodeAt(s--);
                            128 > n ? i[--e] = n : n > 127 && 2048 > n ? (i[--e] = 63 & n | 128,
                                i[--e] = n >> 6 | 192) : (i[--e] = 63 & n | 128,
                                i[--e] = n >> 6 & 63 | 128,
                                i[--e] = n >> 12 | 224)
                        }
                        i[--e] = 0;
                        for (var r = new oe, o = []; e > 2;) {
                            for (o[0] = 0; 0 == o[0];)
                                r.nextBytes(o);
                            i[--e] = o[0]
                        }
                        return i[--e] = 2,
                            i[--e] = 0,
                            new b(i)
                    }

                    function le() {
                        this.n = null,
                            this.e = 0,
                            this.d = null,
                            this.p = null,
                            this.q = null,
                            this.dmp1 = null,
                            this.dmq1 = null,
                            this.coeff = null
                    }

                    function ue(t, e) {
                        null != t && null != e && t.length > 0 && e.length > 0 ? (this.n = ae(t, 16),
                            this.e = parseInt(e, 16)) : console.error("Invalid RSA public key")
                    }

                    function de(t) {
                        return t.modPowInt(this.e, this.n)
                    }

                    function pe(t) {
                        var e = ce(t, this.n.bitLength() + 7 >> 3);
                        if (null == e)
                            return null;
                        var i = this.doPublic(e);
                        if (null == i)
                            return null;
                        var s = i.toString(16);
                        return 0 == (1 & s.length) ? s : "0" + s
                    }

                    function he(t, e) {
                        for (var i = t.toByteArray(), s = 0; s < i.length && 0 == i[s];)
                            ++s;
                        if (i.length - s != e - 1 || 2 != i[s])
                            return null;
                        for (++s; 0 != i[s];)
                            if (++s >= i.length)
                                return null;
                        for (var n = ""; ++s < i.length;) {
                            var r = 255 & i[s];
                            128 > r ? n += String.fromCharCode(r) : r > 191 && 224 > r ? (n += String.fromCharCode((31 & r) << 6 | 63 & i[s + 1]),
                                ++s) : (n += String.fromCharCode((15 & r) << 12 | (63 & i[s + 1]) << 6 | 63 & i[s + 2]),
                                s += 2)
                        }
                        return n
                    }

                    function fe(t, e, i) {
                        null != t && null != e && t.length > 0 && e.length > 0 ? (this.n = ae(t, 16),
                            this.e = parseInt(e, 16),
                            this.d = ae(i, 16)) : console.error("Invalid RSA private key")
                    }

                    function ge(t, e, i, s, n, r, o, a) {
                        null != t && null != e && t.length > 0 && e.length > 0 ? (this.n = ae(t, 16),
                            this.e = parseInt(e, 16),
                            this.d = ae(i, 16),
                            this.p = ae(s, 16),
                            this.q = ae(n, 16),
                            this.dmp1 = ae(r, 16),
                            this.dmq1 = ae(o, 16),
                            this.coeff = ae(a, 16)) : console.error("Invalid RSA private key")
                    }

                    function me(t, e) {
                        var i = new oe
                            , s = t >> 1;
                        this.e = parseInt(e, 16);
                        for (var n = new b(e, 16); ;) {
                            for (; this.p = new b(t - s, 1, i),
                                   0 != this.p.subtract(b.ONE).gcd(n).compareTo(b.ONE) || !this.p.isProbablePrime(10);)
                                ;
                            for (; this.q = new b(s, 1, i),
                                   0 != this.q.subtract(b.ONE).gcd(n).compareTo(b.ONE) || !this.q.isProbablePrime(10);)
                                ;
                            if (this.p.compareTo(this.q) <= 0) {
                                var r = this.p;
                                this.p = this.q,
                                    this.q = r
                            }
                            var o = this.p.subtract(b.ONE)
                                , a = this.q.subtract(b.ONE)
                                , c = o.multiply(a);
                            if (0 == c.gcd(n).compareTo(b.ONE)) {
                                this.n = this.p.multiply(this.q),
                                    this.d = n.modInverse(c),
                                    this.dmp1 = this.d.mod(o),
                                    this.dmq1 = this.d.mod(a),
                                    this.coeff = this.q.modInverse(this.p);
                                break
                            }
                        }
                    }

                    function ve(t) {
                        if (null == this.p || null == this.q)
                            return t.modPow(this.d, this.n);
                        for (var e = t.mod(this.p).modPow(this.dmp1, this.p), i = t.mod(this.q).modPow(this.dmq1, this.q); e.compareTo(i) < 0;)
                            e = e.add(this.p);
                        return e.subtract(i).multiply(this.coeff).mod(this.p).multiply(this.q).add(i)
                    }

                    function _e(t) {
                        var e = ae(t, 16)
                            , i = this.doPrivate(e);
                        return null == i ? null : he(i, this.n.bitLength() + 7 >> 3)
                    }

                    function be(t) {
                        var e, i, s = "";
                        for (e = 0; e + 3 <= t.length; e += 3)
                            i = parseInt(t.substring(e, e + 3), 16),
                                s += je.charAt(i >> 6) + je.charAt(63 & i);
                        for (e + 1 == t.length ? (i = parseInt(t.substring(e, e + 1), 16),
                            s += je.charAt(i << 2)) : e + 2 == t.length && (i = parseInt(t.substring(e, e + 2), 16),
                            s += je.charAt(i >> 2) + je.charAt((3 & i) << 4)); (3 & s.length) > 0;)
                            s += Le;
                        return s
                    }

                    function ye(t) {
                        var e, i, s = "", n = 0;
                        for (e = 0; e < t.length && t.charAt(e) != Le; ++e)
                            v = je.indexOf(t.charAt(e)),
                            v < 0 || (0 == n ? (s += c(v >> 2),
                                i = 3 & v,
                                n = 1) : 1 == n ? (s += c(i << 2 | v >> 4),
                                i = 15 & v,
                                n = 2) : 2 == n ? (s += c(i),
                                s += c(v >> 2),
                                i = 3 & v,
                                n = 3) : (s += c(i << 2 | v >> 4),
                                s += c(15 & v),
                                n = 0));
                        return 1 == n && (s += c(i << 2)),
                            s
                    }

                    try {
                        var we, ke,
                            xe = false;
                        xe && "Microsoft Internet Explorer" == navigator.appName ? (b.prototype.am = i,
                            we = 26) : xe && "Netscape" != navigator.appName ? (b.prototype.am = e,
                            we = 26) : (b.prototype.am = s,
                            we = 28),
                            b.prototype.DB = we,
                            b.prototype.DM = (1 << we) - 1,
                            b.prototype.DV = 1 << we;
                    } catch (e) {
                    }
                    var De = 52;
                    b.prototype.FV = Math.pow(2, De),
                        b.prototype.F1 = De - we,
                        b.prototype.F2 = 2 * we - De;
                    var Se, Ce, Te = "0123456789abcdefghijklmnopqrstuvwxyz", Ie = [];
                    for (Se = "0".charCodeAt(0),
                             Ce = 0; 9 >= Ce; ++Ce)
                        Ie[Se++] = Ce;
                    for (Se = "a".charCodeAt(0),
                             Ce = 10; 36 > Ce; ++Ce)
                        Ie[Se++] = Ce;
                    for (Se = "A".charCodeAt(0),
                             Ce = 10; 36 > Ce; ++Ce)
                        Ie[Se++] = Ce;
                    $.prototype.convert = P,
                        $.prototype.revert = R,
                        $.prototype.reduce = A,
                        $.prototype.mulTo = E,
                        $.prototype.sqrTo = M,
                        O.prototype.convert = B,
                        O.prototype.revert = j,
                        O.prototype.reduce = L,
                        O.prototype.mulTo = K,
                        O.prototype.sqrTo = F,
                        b.prototype.copyTo = p,
                        b.prototype.fromInt = n,
                        b.prototype.fromString = h,
                        b.prototype.clamp = r,
                        b.prototype.dlShiftTo = g,
                        b.prototype.drShiftTo = _,
                        b.prototype.lShiftTo = k,
                        b.prototype.rShiftTo = x,
                        b.prototype.subTo = D,
                        b.prototype.multiplyTo = S,
                        b.prototype.squareTo = C,
                        b.prototype.divRemTo = T,
                        b.prototype.invDigit = N,
                        b.prototype.isEven = U,
                        b.prototype.exp = V,
                        b.prototype.toString = o,
                        b.prototype.negate = f,
                        b.prototype.abs = a,
                        b.prototype.compareTo = u,
                        b.prototype.bitLength = d,
                        b.prototype.mod = I,
                        b.prototype.modPowInt = z,
                        b.ZERO = m(0),
                        b.ONE = m(1),
                        Nt.prototype.convert = Ot,
                        Nt.prototype.revert = Ot,
                        Nt.prototype.mulTo = Bt,
                        Nt.prototype.sqrTo = jt,
                        Ut.prototype.convert = Vt,
                        Ut.prototype.revert = zt,
                        Ut.prototype.reduce = qt,
                        Ut.prototype.mulTo = Jt,
                        Ut.prototype.sqrTo = Ht;
                    var $e = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101, 103, 107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163, 167, 173, 179, 181, 191, 193, 197, 199, 211, 223, 227, 229, 233, 239, 241, 251, 257, 263, 269, 271, 277, 281, 283, 293, 307, 311, 313, 317, 331, 337, 347, 349, 353, 359, 367, 373, 379, 383, 389, 397, 401, 409, 419, 421, 431, 433, 439, 443, 449, 457, 461, 463, 467, 479, 487, 491, 499, 503, 509, 521, 523, 541, 547, 557, 563, 569, 571, 577, 587, 593, 599, 601, 607, 613, 617, 619, 631, 641, 643, 647, 653, 659, 661, 673, 677, 683, 691, 701, 709, 719, 727, 733, 739, 743, 751, 757, 761, 769, 773, 787, 797, 809, 811, 821, 823, 827, 829, 839, 853, 857, 859, 863, 877, 881, 883, 887, 907, 911, 919, 929, 937, 941, 947, 953, 967, 971, 977, 983, 991, [][(![] + [])[!+[] + !![] + !![]] + ([] + {})[+!![]] + (!![] + [])[+!![]] + (!![] + [])[+[]]][([] + {})[!+[] + !![] + !![] + !![] + !![]] + ([] + {})[+!![]] + ([][[]] + [])[+!![]] + (![] + [])[!+[] + !![] + !![]] + (!![] + [])[+[]] + (!![] + [])[+!![]] + ([][[]] + [])[+[]] + ([] + {})[!+[] + !![] + !![] + !![] + !![]] + (!![] + [])[+[]] + ([] + {})[+!![]] + (!![] + [])[+!![]]]((!+[] + !![] + !![] + !![] + !![] + !![] + !![] + !![] + !![] + []) + (!+[] + !![] + !![] + !![] + !![] + !![] + !![] + !![] + !![] + []) + (!+[] + !![] + !![] + !![] + !![] + !![] + !![] + []))(+[])]
                        , Pe = (1 << 26) / $e[$e.length - 1];
                    b.prototype.chunkSize = Y,
                        b.prototype.toRadix = Z,
                        b.prototype.fromRadix = Q,
                        b.prototype.fromNumber = X,
                        b.prototype.bitwiseTo = nt,
                        b.prototype.changeBit = wt,
                        b.prototype.addTo = St,
                        b.prototype.dMultiply = Et,
                        b.prototype.dAddOffset = Mt,
                        b.prototype.multiplyLowerTo = Ft,
                        b.prototype.multiplyUpperTo = Kt,
                        b.prototype.modInt = Wt,
                        b.prototype.millerRabin = Xt,
                        b.prototype.clone = q,
                        b.prototype.intValue = H,
                        b.prototype.byteValue = J,
                        b.prototype.shortValue = G,
                        b.prototype.signum = W,
                        b.prototype.toByteArray = tt,
                        b.prototype.equals = et,
                        b.prototype.min = it,
                        b.prototype.max = st,
                        b.prototype.and = ot,
                        b.prototype.or = ct,
                        b.prototype.xor = ut,
                        b.prototype.andNot = pt,
                        b.prototype.not = ht,
                        b.prototype.shiftLeft = ft,
                        b.prototype.shiftRight = gt,
                        b.prototype.getLowestSetBit = vt,
                        b.prototype.bitCount = bt,
                        b.prototype.testBit = yt,
                        b.prototype.setBit = kt,
                        b.prototype.clearBit = xt,
                        b.prototype.flipBit = Dt,
                        b.prototype.add = Ct,
                        b.prototype.subtract = Tt,
                        b.prototype.multiply = It,
                        b.prototype.divide = Pt,
                        b.prototype.remainder = Rt,
                        b.prototype.divideAndRemainder = At,
                        b.prototype.modPow = Gt,
                        b.prototype.modInverse = Zt,
                        b.prototype.pow = Lt,
                        b.prototype.gcd = Yt,
                        b.prototype.isProbablePrime = Qt,
                        b.prototype.square = $t,
                        te.prototype.init = ee,
                        te.prototype.next = ie;
                    var Re, Ae, Ee, Me = 256;
                    if (null == Ae) {
                        Ae = [],
                            Ee = 0;
                        var Ne;
                        var Be = function (t) {
                            if (this.count = this.count || 0,
                            this.count >= 256 || Ee >= Me)
                                try {
                                    var e = t.x + t.y;
                                    Ae[Ee++] = 255 & e,
                                        this.count += 1
                                } catch (y) {
                                }
                        };
                        window.addEventListener ? window.addEventListener("mousemove", Be, !1) : window.attachEvent && window.attachEvent("onmousemove", Be)
                    }
                    oe.prototype.nextBytes = re,
                        le.prototype.doPublic = de,
                        le.prototype.setPublic = ue,
                        le.prototype.encrypt = pe,
                        le.prototype.doPrivate = ve,
                        le.prototype.setPrivate = fe,
                        le.prototype.setPrivateEx = ge,
                        le.prototype.generate = me,
                        le.prototype.decrypt = _e,
                        function () {
                            var t = function (t, e, n) {
                                var r = new oe
                                    , o = t >> 1;
                                this.e = parseInt(e, 16);
                                var a = new b(e, 16)
                                    , c = this
                                    , l = function () {
                                    var e = function () {
                                        if (c.p.compareTo(c.q) <= 0) {
                                            var t = c.p;
                                            c.p = c.q,
                                                c.q = t
                                        }
                                        var e = c.p.subtract(b.ONE)
                                            , i = c.q.subtract(b.ONE)
                                            , s = e.multiply(i);
                                        0 == s.gcd(a).compareTo(b.ONE) ? (c.n = c.p.multiply(c.q),
                                            c.d = a.modInverse(s),
                                            c.dmp1 = c.d.mod(e),
                                            c.dmq1 = c.d.mod(i),
                                            c.coeff = c.q.modInverse(c.p),
                                            setTimeout(function () {
                                                n()
                                            }, 0)) : setTimeout(l, 0)
                                    }
                                        , i = function () {
                                        c.q = y(),
                                            c.q.fromNumberAsync(o, 1, r, function () {
                                                c.q.subtract(b.ONE).gcda(a, function (t) {
                                                    0 == t.compareTo(b.ONE) && c.q.isProbablePrime(10) ? setTimeout(e, 0) : setTimeout(i, 0)
                                                })
                                            })
                                    }
                                        , s = function () {
                                        c.p = y(),
                                            c.p.fromNumberAsync(t - o, 1, r, function () {
                                                c.p.subtract(b.ONE).gcda(a, function (t) {
                                                    0 == t.compareTo(b.ONE) && c.p.isProbablePrime(10) ? setTimeout(i, 0) : setTimeout(s, 0)
                                                })
                                            })
                                    };
                                    setTimeout(s, 0)
                                };
                                setTimeout(l, 0)
                            };
                            le.prototype.generateAsync = t;
                            var e = function (t, e) {
                                var i = this.s < 0 ? this.negate() : this.clone()
                                    , s = t.s < 0 ? t.negate() : t.clone();
                                if (i.compareTo(s) < 0) {
                                    var n = i;
                                    i = s,
                                        s = n
                                }
                                var r = i.getLowestSetBit()
                                    , o = s.getLowestSetBit();
                                if (0 > o)
                                    return void e(i);
                                o > r && (o = r),
                                o > 0 && (i.rShiftTo(o, i),
                                    s.rShiftTo(o, s));
                                var a = function () {
                                    (r = i.getLowestSetBit()) > 0 && i.rShiftTo(r, i),
                                    (r = s.getLowestSetBit()) > 0 && s.rShiftTo(r, s),
                                        i.compareTo(s) >= 0 ? (i.subTo(s, i),
                                            i.rShiftTo(1, i)) : (s.subTo(i, s),
                                            s.rShiftTo(1, s)),
                                        i.signum() > 0 ? setTimeout(a, 0) : (o > 0 && s.lShiftTo(o, s),
                                            setTimeout(function () {
                                                e(s)
                                            }, 0))
                                };
                                setTimeout(a, 10)
                            };
                            b.prototype.gcda = e;
                            var i = function (t, e, i, s) {
                                if ("number" == typeof e)
                                    if (2 > t)
                                        this.fromInt(1);
                                    else {
                                        this.fromNumber(t, i),
                                        this.testBit(t - 1) || this.bitwiseTo(b.ONE.shiftLeft(t - 1), at, this),
                                        this.isEven() && this.dAddOffset(1, 0);
                                        var n = this
                                            , r = function () {
                                            n.dAddOffset(2, 0),
                                            n.bitLength() > t && n.subTo(b.ONE.shiftLeft(t - 1), n),
                                                n.isProbablePrime(e) ? setTimeout(function () {
                                                    s()
                                                }, 0) : setTimeout(r, 0)
                                        };
                                        setTimeout(r, 0)
                                    }
                                else {
                                    var o = []
                                        , a = 7 & t;
                                    o.length = (t >> 3) + 1,
                                        e.nextBytes(o),
                                        a > 0 ? o[0] &= (1 << a) - 1 : o[0] = 0,
                                        this.fromString(o, 256)
                                }
                            };
                            b.prototype.fromNumberAsync = i
                        }();
                    var je = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
                        , Le = "="
                        , Fe = Fe || {};
                    Fe.env = Fe.env || {};
                    var Ke = Fe
                        , Ue = Object.prototype
                        , Ve = "[object Function]"
                        , ze = ["toString", "valueOf"];
                    Fe.env.parseUA = function (t) {
                        var e, i = function (t) {
                            var e = 0;
                            return parseFloat(t.replace(/\./g, function () {
                                return 1 == e++ ? "" : "."
                            }))
                        }, s = navigator, n = {
                            ie: 0,
                            opera: 0,
                            gecko: 0,
                            webkit: 0,
                            chrome: 0,
                            mobile: null,
                            air: 0,
                            ipad: 0,
                            iphone: 0,
                            ipod: 0,
                            ios: null,
                            android: 0,
                            webos: 0,
                            caja: s && s.cajaVersion,
                            secure: !1,
                            os: null
                        };
                        try {
                            r = t || navigator && navigator.userAgent,
                                o = window && window,
                                a = o && o.href;
                        } catch (e) {
                        }
                        return n.secure = a && 0 === a.toLowerCase().indexOf("https"),
                        r && (/windows|win32/i.test(r) ? n.os = "windows" : /macintosh/i.test(r) ? n.os = "macintosh" : /rhino/i.test(r) && (n.os = "rhino"),
                        /KHTML/.test(r) && (n.webkit = 1),
                            e = r.match(/AppleWebKit\/([^\s]*)/),
                        e && e[1] && (n.webkit = i(e[1]),
                            / Mobile\//.test(r) ? (n.mobile = "Apple",
                                e = r.match(/OS ([^\s]*)/),
                            e && e[1] && (e = i(e[1].replace("_", "."))),
                                n.ios = e,
                                n.ipad = n.ipod = n.iphone = 0,
                                e = r.match(/iPad|iPod|iPhone/),
                            e && e[0] && (n[e[0].toLowerCase()] = n.ios)) : (e = r.match(/NokiaN[^\/]*|Android \d\.\d|webOS\/\d\.\d/),
                            e && (n.mobile = e[0]),
                            /webOS/.test(r) && (n.mobile = "WebOS",
                                e = r.match(/webOS\/([^\s]*);/),
                            e && e[1] && (n.webos = i(e[1]))),
                            / Android/.test(r) && (n.mobile = "Android",
                                e = r.match(/Android ([^\s]*);/),
                            e && e[1] && (n.android = i(e[1])))),
                            e = r.match(/Chrome\/([^\s]*)/),
                            e && e[1] ? n.chrome = i(e[1]) : (e = r.match(/AdobeAIR\/([^\s]*)/),
                            e && (n.air = e[0]))),
                        n.webkit || (e = r.match(/Opera[\s\/]([^\s]*)/),
                            e && e[1] ? (n.opera = i(e[1]),
                                e = r.match(/Version\/([^\s]*)/),
                            e && e[1] && (n.opera = i(e[1])),
                                e = r.match(/Opera Mini[^;]*/),
                            e && (n.mobile = e[0])) : (e = r.match(/MSIE\s([^;]*)/),
                                e && e[1] ? n.ie = i(e[1]) : (e = r.match(/Gecko\/([^\s]*)/),
                                e && (n.gecko = 1,
                                    e = r.match(/rv:([^\s\)]*)/),
                                e && e[1] && (n.gecko = i(e[1]))))))),
                            n
                    }
                        ,
                        Fe.env.ua = Fe.env.parseUA(),
                        Fe.isFunction = function (t) {
                            return "function" == typeof t || Ue.toString.apply(t) === Ve
                        }
                        ,
                        Fe._IEEnumFix = Fe.env.ua.ie ? function (t, e) {
                                var i, s, n;
                                for (i = 0; i < ze.length; i += 1)
                                    s = ze[i],
                                        n = e[s],
                                    Ke.isFunction(n) && n != Ue[s] && (t[s] = n)
                            }
                            : function () {
                            }
                        ,
                        Fe.extend = function (t, e, i) {
                            if (!e || !t)
                                throw new Error("extend failed, please check that all dependencies are included.");
                            var s, n = function () {
                            };
                            if (n.prototype = e.prototype,
                                t.prototype = new n,
                                t.prototype.constructor = t,
                                t.superclass = e.prototype,
                            e.prototype.constructor == Ue.constructor && (e.prototype.constructor = e),
                                i) {
                                for (s in i)
                                    Ke.hasOwnProperty(i, s) && (t.prototype[s] = i[s]);
                                Ke._IEEnumFix(t.prototype, i)
                            }
                        }
                        ,
                    "undefined" != typeof KJUR && KJUR || (KJUR = {}),
                    "undefined" != typeof KJUR.asn1 && KJUR.asn1 || (KJUR.asn1 = {}),
                        KJUR.asn1.ASN1Util = new function () {
                            this.integerToByteHex = function (t) {
                                var e = t.toString(16);
                                return e.length % 2 == 1 && (e = "0" + e),
                                    e
                            }
                                ,
                                this.bigIntToMinTwosComplementsHex = function (t) {
                                    var e = t.toString(16);
                                    if ("-" != e.substr(0, 1))
                                        e.length % 2 == 1 ? e = "0" + e : e.match(/^[0-7]/) || (e = "00" + e);
                                    else {
                                        var i = e.substr(1)
                                            , s = i.length;
                                        s % 2 == 1 ? s += 1 : e.match(/^[0-7]/) || (s += 2);
                                        for (var n = "", r = 0; s > r; r++)
                                            n += "f";
                                        var o = new b(n, 16)
                                            , a = o.xor(t).add(b.ONE);
                                        e = a.toString(16).replace(/^-/, "")
                                    }
                                    return e
                                }
                                ,
                                this.getPEMStringFromHex = function (t, e) {
                                    var i = CryptoJS.enc.Hex.parse(t)
                                        , s = CryptoJS.enc.Base64.stringify(i)
                                        , n = s.replace(/(.{64})/g, "$1\r\n");
                                    return n = n.replace(/\r\n$/, ""),
                                    "-----BEGIN " + e + "-----\r\n" + n + "\r\n-----END " + e + "-----\r\n"
                                }
                        }
                        ,
                        KJUR.asn1.ASN1Object = function () {
                            var n = "";
                            this.getLengthHexFromValue = function () {
                                if ("undefined" == typeof this.hV || null == this.hV)
                                    throw "this.hV is null or undefined.";
                                if (this.hV.length % 2 == 1)
                                    throw "value hex must be even length: n=" + n.length + ",v=" + this.hV;
                                var t = this.hV.length / 2
                                    , e = t.toString(16);
                                if (e.length % 2 == 1 && (e = "0" + e),
                                128 > t)
                                    return e;
                                var i = e.length / 2;
                                if (i > 15)
                                    throw "ASN.1 length too long to represent by 8x: n = " + t.toString(16);
                                var s = 128 + i;
                                return s.toString(16) + e
                            }
                                ,
                                this.getEncodedHex = function () {
                                    return (null == this.hTLV || this.isModified) && (this.hV = this.getFreshValueHex(),
                                        this.hL = this.getLengthHexFromValue(),
                                        this.hTLV = this.hT + this.hL + this.hV,
                                        this.isModified = !1),
                                        this.hTLV
                                }
                                ,
                                this.getValueHex = function () {
                                    return this.getEncodedHex(),
                                        this.hV
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return ""
                                }
                        }
                        ,
                        KJUR.asn1.DERAbstractString = function (t) {
                            KJUR.asn1.DERAbstractString.superclass.constructor.call(this);
                            this.getString = function () {
                                return this.s
                            }
                                ,
                                this.setString = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.s = t,
                                        this.hV = stohex(this.s)
                                }
                                ,
                                this.setStringHex = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.s = null,
                                        this.hV = t
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return this.hV
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.str ? this.setString(t.str) : "undefined" != typeof t.hex && this.setStringHex(t.hex))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERAbstractString, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERAbstractTime = function (t) {
                            KJUR.asn1.DERAbstractTime.superclass.constructor.call(this);
                            this.localDateToUTC = function (t) {
                                utc = t.getTime() + 6e4 * t.getTimezoneOffset();
                                var e = new Date(utc);
                                return e
                            }
                                ,
                                this.formatDate = function (t, e) {
                                    var i = this.zeroPadding
                                        , s = this.localDateToUTC(t)
                                        , n = String(s.getFullYear());
                                    "utc" == e && (n = n.substr(2, 2));
                                    var r = i(String(s.getMonth() + 1), 2)
                                        , o = i(String(s.getDate()), 2)
                                        , a = i(String(s.getHours()), 2)
                                        , c = i(String(s.getMinutes()), 2)
                                        , l = i(String(s.getSeconds()), 2);
                                    return n + r + o + a + c + l + "Z"
                                }
                                ,
                                this.zeroPadding = function (t, e) {
                                    return t.length >= e ? t : new Array(e - t.length + 1).join("0") + t
                                }
                                ,
                                this.getString = function () {
                                    return this.s
                                }
                                ,
                                this.setString = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.s = t,
                                        this.hV = stohex(this.s)
                                }
                                ,
                                this.setByDateValue = function (t, e, i, s, n, r) {
                                    var o = new Date(Date.UTC(t, e - 1, i, s, n, r, 0));
                                    this.setByDate(o)
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return this.hV
                                }
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERAbstractTime, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERAbstractStructured = function (t) {
                            KJUR.asn1.DERAbstractString.superclass.constructor.call(this);
                            this.setByASN1ObjectArray = function (t) {
                                this.hTLV = null,
                                    this.isModified = !0,
                                    this.asn1Array = t
                            }
                                ,
                                this.appendASN1Object = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.asn1Array.push(t)
                                }
                                ,
                                this.asn1Array = [],
                            "undefined" != typeof t && "undefined" != typeof t.array && (this.asn1Array = t.array)
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERAbstractStructured, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERBoolean = function () {
                            KJUR.asn1.DERBoolean.superclass.constructor.call(this),
                                this.hT = "01",
                                this.hTLV = "0101ff"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERBoolean, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERInteger = function (t) {
                            KJUR.asn1.DERInteger.superclass.constructor.call(this),
                                this.hT = "02",
                                this.setByBigInteger = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.hV = KJUR.asn1.ASN1Util.bigIntToMinTwosComplementsHex(t)
                                }
                                ,
                                this.setByInteger = function (t) {
                                    var e = new b(String(t), 10);
                                    this.setByBigInteger(e)
                                }
                                ,
                                this.setValueHex = function (t) {
                                    this.hV = t
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return this.hV
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.bigint ? this.setByBigInteger(t.bigint) : "undefined" != typeof t["int"] ? this.setByInteger(t["int"]) : "undefined" != typeof t.hex && this.setValueHex(t.hex))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERInteger, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERBitString = function (t) {
                            KJUR.asn1.DERBitString.superclass.constructor.call(this),
                                this.hT = "03",
                                this.setHexValueIncludingUnusedBits = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.hV = t
                                }
                                ,
                                this.setUnusedBitsAndHexValue = function (t, e) {
                                    if (0 > t || t > 7)
                                        throw "unused bits shall be from 0 to 7: u = " + t;
                                    var i = "0" + t;
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.hV = i + e
                                }
                                ,
                                this.setByBinaryString = function (t) {
                                    t = t.replace(/0+$/, "");
                                    var e = 8 - t.length % 8;
                                    8 == e && (e = 0);
                                    for (var i = 0; e >= i; i++)
                                        t += "0";
                                    for (var s = "", i = 0; i < t.length - 1; i += 8) {
                                        var n = t.substr(i, 8)
                                            , r = parseInt(n, 2).toString(16);
                                        1 == r.length && (r = "0" + r),
                                            s += r
                                    }
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.hV = "0" + e + s
                                }
                                ,
                                this.setByBooleanArray = function (t) {
                                    for (var e = "", i = 0; i < t.length; i++)
                                        e += 1 == t[i] ? "1" : "0";
                                    this.setByBinaryString(e)
                                }
                                ,
                                this.newFalseArray = function (t) {
                                    for (var e = new Array(t), i = 0; t > i; i++)
                                        e[i] = !1;
                                    return e
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return this.hV
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.hex ? this.setHexValueIncludingUnusedBits(t.hex) : "undefined" != typeof t.bin ? this.setByBinaryString(t.bin) : "undefined" != typeof t.array && this.setByBooleanArray(t.array))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERBitString, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DEROctetString = function (t) {
                            KJUR.asn1.DEROctetString.superclass.constructor.call(this, t),
                                this.hT = "04"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DEROctetString, KJUR.asn1.DERAbstractString),
                        KJUR.asn1.DERNull = function () {
                            KJUR.asn1.DERNull.superclass.constructor.call(this),
                                this.hT = "05",
                                this.hTLV = "0500"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERNull, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERObjectIdentifier = function (t) {
                            var c = function (t) {
                                var e = t.toString(16);
                                return 1 == e.length && (e = "0" + e),
                                    e
                            }
                                , r = function (t) {
                                var e = ""
                                    , i = new b(t, 10)
                                    , s = i.toString(2)
                                    , n = 7 - s.length % 7;
                                7 == n && (n = 0);
                                for (var r = "", o = 0; n > o; o++)
                                    r += "0";
                                s = r + s;
                                for (var o = 0; o < s.length - 1; o += 7) {
                                    var a = s.substr(o, 7);
                                    o != s.length - 7 && (a = "1" + a),
                                        e += c(parseInt(a, 2))
                                }
                                return e
                            };
                            KJUR.asn1.DERObjectIdentifier.superclass.constructor.call(this),
                                this.hT = "06",
                                this.setValueHex = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.s = null,
                                        this.hV = t
                                }
                                ,
                                this.setValueOidString = function (t) {
                                    if (!t.match(/^[0-9.]+$/))
                                        throw "malformed oid string: " + t;
                                    var e = ""
                                        , i = t.split(".")
                                        , s = 40 * parseInt(i[0]) + parseInt(i[1]);
                                    e += c(s),
                                        i.splice(0, 2);
                                    for (var n = 0; n < i.length; n++)
                                        e += r(i[n]);
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.s = null,
                                        this.hV = e
                                }
                                ,
                                this.setValueName = function (t) {
                                    if ("undefined" == typeof KJUR.asn1.x509.OID.name2oidList[t])
                                        throw "DERObjectIdentifier oidName undefined: " + t;
                                    var e = KJUR.asn1.x509.OID.name2oidList[t];
                                    this.setValueOidString(e)
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return this.hV
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.oid ? this.setValueOidString(t.oid) : "undefined" != typeof t.hex ? this.setValueHex(t.hex) : "undefined" != typeof t.name && this.setValueName(t.name))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERObjectIdentifier, KJUR.asn1.ASN1Object),
                        KJUR.asn1.DERUTF8String = function (t) {
                            KJUR.asn1.DERUTF8String.superclass.constructor.call(this, t),
                                this.hT = "0c"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERUTF8String, KJUR.asn1.DERAbstractString),
                        KJUR.asn1.DERNumericString = function (t) {
                            KJUR.asn1.DERNumericString.superclass.constructor.call(this, t),
                                this.hT = "12"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERNumericString, KJUR.asn1.DERAbstractString),
                        KJUR.asn1.DERPrintableString = function (t) {
                            KJUR.asn1.DERPrintableString.superclass.constructor.call(this, t),
                                this.hT = "13"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERPrintableString, KJUR.asn1.DERAbstractString),
                        KJUR.asn1.DERTeletexString = function (t) {
                            KJUR.asn1.DERTeletexString.superclass.constructor.call(this, t),
                                this.hT = "14"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERTeletexString, KJUR.asn1.DERAbstractString),
                        KJUR.asn1.DERIA5String = function (t) {
                            KJUR.asn1.DERIA5String.superclass.constructor.call(this, t),
                                this.hT = "16"
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERIA5String, KJUR.asn1.DERAbstractString),
                        KJUR.asn1.DERUTCTime = function (t) {
                            KJUR.asn1.DERUTCTime.superclass.constructor.call(this, t),
                                this.hT = "17",
                                this.setByDate = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.date = t,
                                        this.s = this.formatDate(this.date, "utc"),
                                        this.hV = stohex(this.s)
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.str ? this.setString(t.str) : "undefined" != typeof t.hex ? this.setStringHex(t.hex) : "undefined" != typeof t.date && this.setByDate(t.date))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERUTCTime, KJUR.asn1.DERAbstractTime),
                        KJUR.asn1.DERGeneralizedTime = function (t) {
                            KJUR.asn1.DERGeneralizedTime.superclass.constructor.call(this, t),
                                this.hT = "18",
                                this.setByDate = function (t) {
                                    this.hTLV = null,
                                        this.isModified = !0,
                                        this.date = t,
                                        this.s = this.formatDate(this.date, "gen"),
                                        this.hV = stohex(this.s)
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.str ? this.setString(t.str) : "undefined" != typeof t.hex ? this.setStringHex(t.hex) : "undefined" != typeof t.date && this.setByDate(t.date))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERGeneralizedTime, KJUR.asn1.DERAbstractTime),
                        KJUR.asn1.DERSequence = function (t) {
                            KJUR.asn1.DERSequence.superclass.constructor.call(this, t),
                                this.hT = "30",
                                this.getFreshValueHex = function () {
                                    for (var t = "", e = 0; e < this.asn1Array.length; e++) {
                                        var i = this.asn1Array[e];
                                        t += i.getEncodedHex()
                                    }
                                    return this.hV = t,
                                        this.hV
                                }
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERSequence, KJUR.asn1.DERAbstractStructured),
                        KJUR.asn1.DERSet = function (t) {
                            KJUR.asn1.DERSet.superclass.constructor.call(this, t),
                                this.hT = "31",
                                this.getFreshValueHex = function () {
                                    for (var t = [], e = 0; e < this.asn1Array.length; e++) {
                                        var i = this.asn1Array[e];
                                        t.push(i.getEncodedHex())
                                    }
                                    return t.sort(),
                                        this.hV = t.join(""),
                                        this.hV
                                }
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERSet, KJUR.asn1.DERAbstractStructured),
                        KJUR.asn1.DERTaggedObject = function (t) {
                            KJUR.asn1.DERTaggedObject.superclass.constructor.call(this),
                                this.hT = "a0",
                                this.hV = "",
                                this.isExplicit = !0,
                                this.asn1Object = null,
                                this.setASN1Object = function (t, e, i) {
                                    this.hT = e,
                                        this.isExplicit = t,
                                        this.asn1Object = i,
                                        this.isExplicit ? (this.hV = this.asn1Object.getEncodedHex(),
                                            this.hTLV = null,
                                            this.isModified = !0) : (this.hV = null,
                                            this.hTLV = i.getEncodedHex(),
                                            this.hTLV = this.hTLV.replace(/^../, e),
                                            this.isModified = !1)
                                }
                                ,
                                this.getFreshValueHex = function () {
                                    return this.hV
                                }
                                ,
                            "undefined" != typeof t && ("undefined" != typeof t.tag && (this.hT = t.tag),
                            "undefined" != typeof t.explicit && (this.isExplicit = t.explicit),
                            "undefined" != typeof t.obj && (this.asn1Object = t.obj,
                                this.setASN1Object(this.isExplicit, this.hT, this.asn1Object)))
                        }
                        ,
                        Fe.extend(KJUR.asn1.DERTaggedObject, KJUR.asn1.ASN1Object),
                        function (c) {
                            "use strict";
                            var l, t = {};
                            t.decode = function (t) {
                                var e;
                                if (l === c) {
                                    var i = "0123456789ABCDEF"
                                        , s = " \f\n\r  \u2028\u2029";
                                    for (l = [],
                                             e = 0; 16 > e; ++e)
                                        l[i.charAt(e)] = e;
                                    for (i = i.toLowerCase(),
                                             e = 10; 16 > e; ++e)
                                        l[i.charAt(e)] = e;
                                    for (e = 0; e < s.length; ++e)
                                        l[s.charAt(e)] = -1
                                }
                                var n = []
                                    , r = 0
                                    , o = 0;
                                for (e = 0; e < t.length; ++e) {
                                    var a = t.charAt(e);
                                    if ("=" == a)
                                        break;
                                    if (a = l[a],
                                    -1 != a) {
                                        if (a === c)
                                            throw "Illegal character at offset " + e;
                                        r |= a,
                                            ++o >= 2 ? (n[n.length] = r,
                                                r = 0,
                                                o = 0) : r <<= 4
                                    }
                                }
                                if (o)
                                    throw "Hex encoding incomplete: 4 bits missing";
                                return n
                            }
                                ,
                                window.Hex = t
                        }(),
                        function (c) {
                            "use strict";
                            var l, i = {};
                            i.decode = function (t) {
                                var e;
                                if (l === c) {
                                    var i = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
                                        , s = "= \f\n\r  \u2028\u2029";
                                    for (l = [],
                                             e = 0; 64 > e; ++e)
                                        l[i.charAt(e)] = e;
                                    for (e = 0; e < s.length; ++e)
                                        l[s.charAt(e)] = -1
                                }
                                var n = []
                                    , r = 0
                                    , o = 0;
                                for (e = 0; e < t.length; ++e) {
                                    var a = t.charAt(e);
                                    if ("=" == a)
                                        break;
                                    if (a = l[a],
                                    -1 != a) {
                                        if (a === c)
                                            throw "Illegal character at offset " + e;
                                        r |= a,
                                            ++o >= 4 ? (n[n.length] = r >> 16,
                                                n[n.length] = r >> 8 & 255,
                                                n[n.length] = 255 & r,
                                                r = 0,
                                                o = 0) : r <<= 6
                                    }
                                }
                                switch (o) {
                                    case 1:
                                        throw "Base64 encoding incomplete: at least 2 bits missing";
                                    case 2:
                                        n[n.length] = r >> 10;
                                        break;
                                    case 3:
                                        n[n.length] = r >> 16,
                                            n[n.length] = r >> 8 & 255
                                }
                                return n
                            }
                                ,
                                i.re = /-----BEGIN [^-]+-----([A-Za-z0-9+\/=\s]+)-----END [^-]+-----|begin-base64[^\n]+\n([A-Za-z0-9+\/=\s]+)====/,
                                i.unarmor = function (t) {
                                    var e = i.re.exec(t);
                                    if (e)
                                        if (e[1])
                                            t = e[1];
                                        else {
                                            if (!e[2])
                                                throw "RegExp out of sync";
                                            t = e[2]
                                        }
                                    return i.decode(t)
                                }
                                ,
                                window.Base64 = i
                        }(),
                        function (o) {
                            "use strict";

                            function l(t, e) {
                                t instanceof l ? (this.enc = t.enc,
                                    this.pos = t.pos) : (this.enc = t,
                                    this.pos = e)
                            }

                            function u(t, e, i, s, n) {
                                this.stream = t,
                                    this.header = e,
                                    this.length = i,
                                    this.tag = s,
                                    this.sub = n
                            }

                            var r = 100
                                , a = "…"
                                , d = {
                                tag: function (t, e) {
                                },
                                text: function (t) {
                                }
                            };
                            l.prototype.get = function (t) {
                                if (t === o && (t = this.pos++),
                                t >= this.enc.length)
                                    throw "Requesting byte offset " + t + " on a stream of length " + this.enc.length;
                                return this.enc[t]
                            }
                                ,
                                l.prototype.hexDigits = "0123456789ABCDEF",
                                l.prototype.hexByte = function (t) {
                                    return this.hexDigits.charAt(t >> 4 & 15) + this.hexDigits.charAt(15 & t)
                                }
                                ,
                                l.prototype.hexDump = function (t, e, i) {
                                    for (var s = "", n = t; e > n; ++n)
                                        if (s += this.hexByte(this.get(n)),
                                        i !== !0)
                                            switch (15 & n) {
                                                case 7:
                                                    s += " ";
                                                    break;
                                                case 15:
                                                    s += "\n";
                                                    break;
                                                default:
                                                    s += " "
                                            }
                                    return s
                                }
                                ,
                                l.prototype.parseStringISO = function (t, e) {
                                    for (var i = "", s = t; e > s; ++s)
                                        i += String.fromCharCode(this.get(s));
                                    return i
                                }
                                ,
                                l.prototype.parseStringUTF = function (t, e) {
                                    for (var i = "", s = t; e > s;) {
                                        var n = this.get(s++);
                                        i += 128 > n ? String.fromCharCode(n) : n > 191 && 224 > n ? String.fromCharCode((31 & n) << 6 | 63 & this.get(s++)) : String.fromCharCode((15 & n) << 12 | (63 & this.get(s++)) << 6 | 63 & this.get(s++))
                                    }
                                    return i
                                }
                                ,
                                l.prototype.parseStringBMP = function (t, e) {
                                    for (var i = "", s = t; e > s; s += 2) {
                                        var n = this.get(s)
                                            , r = this.get(s + 1);
                                        i += String.fromCharCode((n << 8) + r)
                                    }
                                    return i
                                }
                                ,
                                l.prototype.reTime = /^((?:1[89]|2\d)?\d\d)(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])([01]\d|2[0-3])(?:([0-5]\d)(?:([0-5]\d)(?:[.,](\d{1,3}))?)?)?(Z|[-+](?:[0]\d|1[0-2])([0-5]\d)?)?$/,
                                l.prototype.parseTime = function (t, e) {
                                    var i = this.parseStringISO(t, e)
                                        , s = this.reTime.exec(i);
                                    return s ? (i = s[1] + "-" + s[2] + "-" + s[3] + " " + s[4],
                                    s[5] && (i += ":" + s[5],
                                    s[6] && (i += ":" + s[6],
                                    s[7] && (i += "." + s[7]))),
                                    s[8] && (i += " UTC",
                                    "Z" != s[8] && (i += s[8],
                                    s[9] && (i += ":" + s[9]))),
                                        i) : "Unrecognized time: " + i
                                }
                                ,
                                l.prototype.parseInteger = function (t, e) {
                                    var i = e - t;
                                    if (i > 4) {
                                        i <<= 3;
                                        var s = this.get(t);
                                        if (0 === s)
                                            i -= 8;
                                        else
                                            for (; 128 > s;)
                                                s <<= 1,
                                                    --i;
                                        return "(" + i + " bit)"
                                    }
                                    for (var n = 0, r = t; e > r; ++r)
                                        n = n << 8 | this.get(r);
                                    return n
                                }
                                ,
                                l.prototype.parseBitString = function (t, e) {
                                    var i = this.get(t)
                                        , s = (e - t - 1 << 3) - i
                                        , n = "(" + s + " bit)";
                                    if (20 >= s) {
                                        var r = i;
                                        n += " ";
                                        for (var o = e - 1; o > t; --o) {
                                            for (var a = this.get(o), c = r; 8 > c; ++c)
                                                n += a >> c & 1 ? "1" : "0";
                                            r = 0
                                        }
                                    }
                                    return n
                                }
                                ,
                                l.prototype.parseOctetString = function (t, e) {
                                    var i = e - t
                                        , s = "(" + i + " byte) ";
                                    i > r && (e = t + r);
                                    for (var n = t; e > n; ++n)
                                        s += this.hexByte(this.get(n));
                                    return i > r && (s += a),
                                        s
                                }
                                ,
                                l.prototype.parseOID = function (t, e) {
                                    for (var i = "", s = 0, n = 0, r = t; e > r; ++r) {
                                        var o = this.get(r);
                                        if (s = s << 7 | 127 & o,
                                            n += 7,
                                            !(128 & o)) {
                                            if ("" === i) {
                                                var a = 80 > s ? 40 > s ? 0 : 1 : 2;
                                                i = a + "." + (s - 40 * a)
                                            } else
                                                i += "." + (n >= 31 ? "bigint" : s);
                                            s = n = 0
                                        }
                                    }
                                    return i
                                }
                                ,
                                u.prototype.typeName = function () {
                                    if (this.tag === o)
                                        return "unknown";
                                    var t = this.tag >> 6
                                        , e = (this.tag >> 5 & 1,
                                    31 & this.tag);
                                    switch (t) {
                                        case 0:
                                            switch (e) {
                                                case 0:
                                                    return "EOC";
                                                case 1:
                                                    return "BOOLEAN";
                                                case 2:
                                                    return "INTEGER";
                                                case 3:
                                                    return "BIT_STRING";
                                                case 4:
                                                    return "OCTET_STRING";
                                                case 5:
                                                    return "NULL";
                                                case 6:
                                                    return "OBJECT_IDENTIFIER";
                                                case 7:
                                                    return "ObjectDescriptor";
                                                case 8:
                                                    return "EXTERNAL";
                                                case 9:
                                                    return "REAL";
                                                case 10:
                                                    return "ENUMERATED";
                                                case 11:
                                                    return "EMBEDDED_PDV";
                                                case 12:
                                                    return "UTF8String";
                                                case 16:
                                                    return "SEQUENCE";
                                                case 17:
                                                    return "SET";
                                                case 18:
                                                    return "NumericString";
                                                case 19:
                                                    return "PrintableString";
                                                case 20:
                                                    return "TeletexString";
                                                case 21:
                                                    return "VideotexString";
                                                case 22:
                                                    return "IA5String";
                                                case 23:
                                                    return "UTCTime";
                                                case 24:
                                                    return "GeneralizedTime";
                                                case 25:
                                                    return "GraphicString";
                                                case 26:
                                                    return "VisibleString";
                                                case 27:
                                                    return "GeneralString";
                                                case 28:
                                                    return "UniversalString";
                                                case 30:
                                                    return "BMPString";
                                                default:
                                                    return "Universal_" + e.toString(16)
                                            }
                                        case 1:
                                            return "Application_" + e.toString(16);
                                        case 2:
                                            return "[" + e + "]";
                                        case 3:
                                            return "Private_" + e.toString(16)
                                    }
                                }
                                ,
                                u.prototype.reSeemsASCII = /^[ -~]+$/,
                                u.prototype.content = function () {
                                    if (this.tag === o)
                                        return null;
                                    var t = this.tag >> 6
                                        , e = 31 & this.tag
                                        , i = this.posContent()
                                        , s = Math.abs(this.length);
                                    if (0 !== t) {
                                        if (null !== this.sub)
                                            return "(" + this.sub.length + " elem)";
                                        var n = this.stream.parseStringISO(i, i + Math.min(s, r));
                                        return this.reSeemsASCII.test(n) ? n.substring(0, 2 * r) + (n.length > 2 * r ? a : "") : this.stream.parseOctetString(i, i + s)
                                    }
                                    switch (e) {
                                        case 1:
                                            return 0 === this.stream.get(i) ? "false" : "true";
                                        case 2:
                                            return this.stream.parseInteger(i, i + s);
                                        case 3:
                                            return this.sub ? "(" + this.sub.length + " elem)" : this.stream.parseBitString(i, i + s);
                                        case 4:
                                            return this.sub ? "(" + this.sub.length + " elem)" : this.stream.parseOctetString(i, i + s);
                                        case 6:
                                            return this.stream.parseOID(i, i + s);
                                        case 16:
                                        case 17:
                                            return "(" + this.sub.length + " elem)";
                                        case 12:
                                            return this.stream.parseStringUTF(i, i + s);
                                        case 18:
                                        case 19:
                                        case 20:
                                        case 21:
                                        case 22:
                                        case 26:
                                            return this.stream.parseStringISO(i, i + s);
                                        case 30:
                                            return this.stream.parseStringBMP(i, i + s);
                                        case 23:
                                        case 24:
                                            return this.stream.parseTime(i, i + s)
                                    }
                                    return null
                                }
                                ,
                                u.prototype.toString = function () {
                                    return this.typeName() + "@" + this.stream.pos + "[header:" + this.header + ",length:" + this.length + ",sub:" + (null === this.sub ? "null" : this.sub.length) + "]"
                                }
                                ,
                                u.prototype.print = function (t) {
                                    if (t === o && (t = ""),
                                    null !== this.sub) {
                                        t += " ";
                                        for (var e = 0, i = this.sub.length; i > e; ++e)
                                            this.sub[e].print(t)
                                    }
                                }
                                ,
                                u.prototype.toPrettyString = function (t) {
                                    t === o && (t = "");
                                    var e = t + this.typeName() + " @" + this.stream.pos;
                                    if (this.length >= 0 && (e += "+"),
                                        e += this.length,
                                        32 & this.tag ? e += " (constructed)" : 3 != this.tag && 4 != this.tag || null === this.sub || (e += " (encapsulates)"),
                                        e += "\n",
                                    null !== this.sub) {
                                        t += " ";
                                        for (var i = 0, s = this.sub.length; s > i; ++i)
                                            e += this.sub[i].toPrettyString(t)
                                    }
                                    return e
                                }
                                ,
                                u.prototype.toDOM = function () {
                                    var t = d.tag("div", "node");
                                    t.asn1 = this;
                                    var e = d.tag("div", "head")
                                        , i = this.typeName().replace(/_/g, " ");
                                    e.innerHTML = i;
                                    var s = this.content();
                                    if (null !== s) {
                                        s = String(s).replace(/</g, "&lt;");
                                        var n = d.tag("span", "preview");
                                        n.appendChild(d.text(s)),
                                            e.appendChild(n)
                                    }
                                    t.appendChild(e),
                                        this.node = t,
                                        this.head = e;
                                    var r = d.tag("div", "value");
                                    if (i = "Offset: " + this.stream.pos + "<br/>",
                                        i += "Length: " + this.header + "+",
                                        i += this.length >= 0 ? this.length : -this.length + " (undefined)",
                                        32 & this.tag ? i += "<br/>(constructed)" : 3 != this.tag && 4 != this.tag || null === this.sub || (i += "<br/>(encapsulates)"),
                                    null !== s && (i += "<br/>Value:<br/><b>" + s + "</b>",
                                    "object" == typeof oids && 6 == this.tag)) {
                                        var o = oids[s];
                                        o && (o.d && (i += "<br/>" + o.d),
                                        o.c && (i += "<br/>" + o.c),
                                        o.w && (i += "<br/>(warning!)"))
                                    }
                                    r.innerHTML = i,
                                        t.appendChild(r);
                                    var a = d.tag("div", "sub");
                                    if (null !== this.sub)
                                        for (var c = 0, l = this.sub.length; l > c; ++c)
                                            a.appendChild(this.sub[c].toDOM());
                                    return t.appendChild(a),
                                        e.onclick = function () {
                                            t.className = "node collapsed" == t.className ? "node" : "node collapsed"
                                        }
                                        ,
                                        t
                                }
                                ,
                                u.prototype.posStart = function () {
                                    return this.stream.pos
                                }
                                ,
                                u.prototype.posContent = function () {
                                    return this.stream.pos + this.header
                                }
                                ,
                                u.prototype.posEnd = function () {
                                    return this.stream.pos + this.header + Math.abs(this.length)
                                }
                                ,
                                u.prototype.fakeHover = function (t) {
                                    this.node.className += " hover",
                                    t && (this.head.className += " hover")
                                }
                                ,
                                u.prototype.fakeOut = function (t) {
                                    var e = / ?hover/;
                                    this.node.className = this.node.className.replace(e, ""),
                                    t && (this.head.className = this.head.className.replace(e, ""))
                                }
                                ,
                                u.prototype.toHexDOM_sub = function (t, e, i, s, n) {
                                    if (!(s >= n)) {
                                        var r = d.tag("span", e);
                                        r.appendChild(d.text(i.hexDump(s, n))),
                                            t.appendChild(r)
                                    }
                                }
                                ,
                                u.prototype.toHexDOM = function (e) {
                                    var t = d.tag("span", "hex");
                                    if (e === o && (e = t),
                                        this.head.hexNode = t,
                                        this.head.onmouseover = function () {
                                            this.hexNode.className = "hexCurrent"
                                        }
                                        ,
                                        this.head.onmouseout = function () {
                                            this.hexNode.className = "hex"
                                        }
                                        ,
                                        t.asn1 = this,
                                        t.onmouseover = function () {
                                            var t = !e.selected;
                                            t && (e.selected = this.asn1,
                                                this.className = "hexCurrent"),
                                                this.asn1.fakeHover(t)
                                        }
                                        ,
                                        t.onmouseout = function () {
                                            var t = e.selected == this.asn1;
                                            this.asn1.fakeOut(t),
                                            t && (e.selected = null,
                                                this.className = "hex")
                                        }
                                        ,
                                        this.toHexDOM_sub(t, "tag", this.stream, this.posStart(), this.posStart() + 1),
                                        this.toHexDOM_sub(t, this.length >= 0 ? "dlen" : "ulen", this.stream, this.posStart() + 1, this.posContent()),
                                    null === this.sub)
                                        t.appendChild(d.text(this.stream.hexDump(this.posContent(), this.posEnd())));
                                    else if (this.sub.length > 0) {
                                        var i = this.sub[0]
                                            , s = this.sub[this.sub.length - 1];
                                        this.toHexDOM_sub(t, "intro", this.stream, this.posContent(), i.posStart());
                                        for (var n = 0, r = this.sub.length; r > n; ++n)
                                            t.appendChild(this.sub[n].toHexDOM(e));
                                        this.toHexDOM_sub(t, "outro", this.stream, s.posEnd(), this.posEnd())
                                    }
                                    return t
                                }
                                ,
                                u.prototype.toHexString = function (t) {
                                    return this.stream.hexDump(this.posStart(), this.posEnd(), !0)
                                }
                                ,
                                u.decodeLength = function (t) {
                                    var e = t.get()
                                        , i = 127 & e;
                                    if (i == e)
                                        return i;
                                    if (i > 3)
                                        throw "Length over 24 bits not supported at position " + (t.pos - 1);
                                    if (0 === i)
                                        return -1;
                                    e = 0;
                                    for (var s = 0; i > s; ++s)
                                        e = e << 8 | t.get();
                                    return e
                                }
                                ,
                                u.hasContent = function (t, e, i) {
                                    if (32 & t)
                                        return !0;
                                    if (3 > t || t > 4)
                                        return !1;
                                    var s = new l(i);
                                    3 == t && s.get();
                                    var n = s.get();
                                    if (n >> 6 & 1)
                                        return !1;
                                    try {
                                        var r = u.decodeLength(s);
                                        return s.pos - i.pos + r == e
                                    } catch (p) {
                                        return !1
                                    }
                                }
                                ,
                                u.decode = function (t) {
                                    t instanceof l || (t = new l(t, 0));
                                    var e = new l(t)
                                        , i = t.get()
                                        , s = u.decodeLength(t)
                                        , n = t.pos - e.pos
                                        , r = null;
                                    if (u.hasContent(i, s, t)) {
                                        var o = t.pos;
                                        if (3 == i && t.get(),
                                            r = [],
                                        s >= 0) {
                                            for (var a = o + s; t.pos < a;)
                                                r[r.length] = u.decode(t);
                                            if (t.pos != a)
                                                throw "Content size is not correct for container starting at offset " + o
                                        } else
                                            try {
                                                for (; ;) {
                                                    var c = u.decode(t);
                                                    if (0 === c.tag)
                                                        break;
                                                    r[r.length] = c
                                                }
                                                s = o - t.pos
                                            } catch (h) {
                                                throw "Exception while decoding undefined length content: " + h
                                            }
                                    } else
                                        t.pos += s;
                                    return new u(e, n, s, i, r)
                                }
                                ,
                                u.test = function () {
                                    for (var t = [{
                                        value: [39],
                                        expected: 39
                                    }, {
                                        value: [129, 201],
                                        expected: 201
                                    }, {
                                        value: [131, 254, 220, 186],
                                        expected: 16702650
                                    }], e = 0, i = t.length; i > e; ++e) {
                                        var s = new l(t[e].value, 0)
                                            , n = u.decodeLength(s);
                                    }
                                }
                                ,
                                window.ASN1 = u
                        }(),
                        ASN1.prototype.getHexStringValue = function () {
                            var t = this.toHexString()
                                , e = 2 * this.header
                                , i = 2 * this.length;
                            return t.substr(e, i)
                        }
                        ,
                        le.prototype.parseKey = function (t) {
                            try {
                                var e = 0
                                    , i = 0
                                    , s = /^\s*(?:[0-9A-Fa-f][0-9A-Fa-f]\s*)+$/
                                    , n = s.test(t) ? Hex.decode(t) : Base64.unarmor(t)
                                    , r = ASN1.decode(n);
                                if (3 === r.sub.length && (r = r.sub[2].sub[0]),
                                9 === r.sub.length) {
                                    e = r.sub[1].getHexStringValue(),
                                        this.n = ae(e, 16),
                                        i = r.sub[2].getHexStringValue(),
                                        this.e = parseInt(i, 16);
                                    var o = r.sub[3].getHexStringValue();
                                    this.d = ae(o, 16);
                                    var a = r.sub[4].getHexStringValue();
                                    this.p = ae(a, 16);
                                    var c = r.sub[5].getHexStringValue();
                                    this.q = ae(c, 16);
                                    var l = r.sub[6].getHexStringValue();
                                    this.dmp1 = ae(l, 16);
                                    var u = r.sub[7].getHexStringValue();
                                    this.dmq1 = ae(u, 16);
                                    var d = r.sub[8].getHexStringValue();
                                    this.coeff = ae(d, 16)
                                } else {
                                    if (2 !== r.sub.length)
                                        return !1;
                                    var p = r.sub[1]
                                        , h = p.sub[0];
                                    e = h.sub[0].getHexStringValue(),
                                        this.n = ae(e, 16),
                                        i = h.sub[1].getHexStringValue(),
                                        this.e = parseInt(i, 16)
                                }
                                return !0
                            } catch (f) {
                                return !1
                            }
                        }
                        ,
                        le.prototype.getPrivateBaseKey = function () {
                            var t = {
                                array: [new KJUR.asn1.DERInteger({
                                    "int": 0
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.n
                                }), new KJUR.asn1.DERInteger({
                                    "int": this.e
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.d
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.p
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.q
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.dmp1
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.dmq1
                                }), new KJUR.asn1.DERInteger({
                                    bigint: this.coeff
                                })]
                            }
                                , e = new KJUR.asn1.DERSequence(t);
                            return e.getEncodedHex()
                        }
                        ,
                        le.prototype.getPrivateBaseKeyB64 = function () {
                            return be(this.getPrivateBaseKey())
                        }
                        ,
                        le.prototype.getPublicBaseKey = function () {
                            var t = {
                                array: [new KJUR.asn1.DERObjectIdentifier({
                                    oid: "1.2.840.113549.1.1.1"
                                }), new KJUR.asn1.DERNull]
                            }
                                , e = new KJUR.asn1.DERSequence(t);
                            t = {
                                array: [new KJUR.asn1.DERInteger({
                                    bigint: this.n
                                }), new KJUR.asn1.DERInteger({
                                    "int": this.e
                                })]
                            };
                            var i = new KJUR.asn1.DERSequence(t);
                            t = {
                                hex: "00" + i.getEncodedHex()
                            };
                            var s = new KJUR.asn1.DERBitString(t);
                            t = {
                                array: [e, s]
                            };
                            var n = new KJUR.asn1.DERSequence(t);
                            return n.getEncodedHex()
                        }
                        ,
                        le.prototype.getPublicBaseKeyB64 = function () {
                            return be(this.getPublicBaseKey())
                        }
                        ,
                        le.prototype.wordwrap = function (t, e) {
                            if (e = e || 64,
                                !t)
                                return t;
                            var i = "(.{1," + e + "})( +|$\n?)|(.{1," + e + "})";
                            return t.match(RegExp(i, "g")).join("\n")
                        }
                        ,
                        le.prototype.getPrivateKey = function () {
                            var t = "-----BEGIN RSA PRIVATE KEY-----\n";
                            return t += this.wordwrap(this.getPrivateBaseKeyB64()) + "\n",
                                t += "-----END RSA PRIVATE KEY-----"
                        }
                        ,
                        le.prototype.getPublicKey = function () {
                            var t = "-----BEGIN PUBLIC KEY-----\n";
                            return t += this.wordwrap(this.getPublicBaseKeyB64()) + "\n",
                                t += "-----END PUBLIC KEY-----"
                        }
                        ,
                        le.prototype.hasPublicKeyProperty = function (t) {
                            return t = t || {},
                            t.hasOwnProperty("n") && t.hasOwnProperty("e")
                        }
                        ,
                        le.prototype.hasPrivateKeyProperty = function (t) {
                            return t = t || {},
                            t.hasOwnProperty("n") && t.hasOwnProperty("e") && t.hasOwnProperty("d") && t.hasOwnProperty("p") && t.hasOwnProperty("q") && t.hasOwnProperty("dmp1") && t.hasOwnProperty("dmq1") && t.hasOwnProperty("coeff")
                        }
                        ,
                        le.prototype.parsePropertiesFrom = function (t) {
                            this.n = t.n,
                                this.e = t.e,
                            t.hasOwnProperty("d") && (this.d = t.d,
                                this.p = t.p,
                                this.q = t.q,
                                this.dmp1 = t.dmp1,
                                this.dmq1 = t.dmq1,
                                this.coeff = t.coeff)
                        }
                    ;
                    var qe = function (t) {
                        le.call(this),
                        t && ("string" == typeof t ? this.parseKey(t) : (this.hasPrivateKeyProperty(t) || this.hasPublicKeyProperty(t)) && this.parsePropertiesFrom(t))
                    };
                    (qe.prototype = new le).constructor = qe;
                    var He = function (t) {
                        t = t || {},
                            this.default_key_size = parseInt(t.default_key_size) || 1024,
                            this.default_public_exponent = t.default_public_exponent || "010001",
                            this.log = t.log || !1,
                            this.key = null
                    };
                    He.prototype.setKey = function (t) {
                        this.log && this.key && console.warn("A key was already set, overriding existing."),
                            this.key = new qe(t)
                    }
                        ,
                        He.prototype.setPrivateKey = function (t) {
                            this.setKey(t)
                        }
                        ,
                        He.prototype.setPublicKey = function (t) {
                            this.setKey(t)
                        }
                        ,
                        He.prototype.decrypt = function (t) {
                            try {
                                return this.getKey().decrypt(ye(t))
                            } catch (b) {
                                return !1
                            }
                        }
                        ,
                        He.prototype.encrypt = function (t) {
                            try {
                                return be(this.getKey().encrypt(t))
                            } catch (b) {
                                return !1
                            }
                        }
                        ,
                        He.prototype.getKey = function (t) {
                            if (!this.key) {
                                if (this.key = new qe,
                                t && "[object Function]" === {}.toString.call(t))
                                    return void this.key.generateAsync(this.default_key_size, this.default_public_exponent, t);
                                this.key.generate(this.default_key_size, this.default_public_exponent)
                            }
                            return this.key
                        }
                        ,
                        He.prototype.getPrivateKey = function () {
                            return this.getKey().getPrivateKey()
                        }
                        ,
                        He.prototype.getPrivateKeyB64 = function () {
                            return this.getKey().getPrivateBaseKeyB64()
                        }
                        ,
                        He.prototype.getPublicKey = function () {
                            return this.getKey().getPublicKey()
                        }
                        ,
                        He.prototype.getPublicKeyB64 = function () {
                            return this.getKey().getPublicBaseKeyB64()
                        }
                        ,
                        He.version = "2.3.1",
                        t.JSEncrypt = He
                }
            ) ? s.apply(e, n) : s) === undefined || (i.exports = r)
        }
            .call(e, i, e, t)) === undefined || (t.exports = r)
    },
    jsencrypt: function (t, e, r) {
        var i;
        (i = function (t, e, i) {
            var s = r("encrypt");

            function n() {
                void 0 !== s && (this.jsencrypt = new s.JSEncrypt,
                    this.jsencrypt.setPublicKey("-----BEGIN PUBLIC KEY-----MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDq04c6My441Gj0UFKgrqUhAUg+kQZeUeWSPlAU9fr4HBPDldAeqzx1UR99KJHuQh/zs1HOamE2dgX9z/2oXcJaqoRIA/FXysx+z2YlJkSk8XQLcQ8EBOkp//MZrixam7lCYpNOjadQBb2Ot0U/Ky+jF2p+Ie8gSZ7/u+Wnr5grywIDAQAB-----END PUBLIC KEY-----"))
            }

            n.prototype.encode = function (t, e) {
                var i = e ? e + "|" + t : t;
                return encodeURIComponent(this.jsencrypt.encrypt(i))
            }
                ,
                i.exports = n
        }
            .call(e, r, e, t)) === undefined || (t.exports = i)
    }
});

function z(pwd, time) {
    var n = _n("jsencrypt");
    var g = (new n);
    var r = g.encode(pwd, time);
    return r;
}

function r(param1, param2) {
    if (window.o >= 6) {
        alert('不要戳这么多下，人家好痛嘛~');
        location.reload();
    }
    return z(param1, param2);
}

function getResult(i) {
    time = Date.parse(new Date())
    m = z(time, i);
    t = i + '-' + time + "|";
    return m + ':::' + t;
}

// console.log(getResult(1))

```

{{% /spoiler %}}

有几个坑点要注意：上面的python代码中并没有和浏览器运行一样每点击一次就把之前的时间戳加上，因为这样的限制存在且仅存在于手动点击中，我们这样爬虫是不需要的；另一点是通过requests发包时直接在url里写参数会过不了风控，而单独加进params里就可以过了（虽然我还不知道是什么原理）

最终要的是1，2，3等奖的金额之和，通过前端代码可以直到1等奖是3等奖的15倍，2等奖是3等奖的5倍，加起来就是api请求得到金额的24倍

## 猿人学7 - 动态字体 随风漂移

> https://match.yuanrenxue.com/match/7
>
> 题目要求：采集这5页中**胜点**列的数据，找出胜点**最高**的召唤师，将召唤师姓名填入答案中

这次要求相加的数字是css+特殊字体控制的，api响应会有woff和data两个字段；对动态css字体加密我并不是很懂，所以参考了一些wp

简单来说，就是api每次请求都会返回一份字体文件和代表数字的数据，根据这份字体文件中的对应关系来显示数字，在python中我们可以使用fontTools这个库来处理；一般的处理方式都是先把字体存为xml格式，再从extraNames节点中提取对应的数据与具体显示数字的映射，但本题每次字体文件中的extraNames节点都不是按照0~9的规律进行排列，那怎么获取映射关系呢？

观察得知，每一个字的字形中on的值是一样的，以数字1为例

![image-20230105164734838](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230105164734838.png)

两份字体文件中虽然名字不一样，一个叫uni一个叫unif821，但两者on字段值是一样的，我们只要存一份这样的映射，在之后的文件中进行匹配替换即可；部分字形的on字段有很多数据，我们仅取10位进行比对（已经够用了）

还有一个要注意的点，最后要的是胜点最高的玩家姓名，而这一点在api返回值里并没有，还是有前端代码的参与

![image-20230105153642107](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230105153642107.png)

愉快的coding时间！

{{% spoiler "exp.py" %}}

```python
import base64

import requests
from fontTools.ttLib import TTFont
from xml.dom.minidom import parse

font_map = {'1010010010': 0, '1001101111': 1, '1001101010': 2, '1010110010': 3, '1111111111': 4, '1110101001': 5, '1010101010': 6, '1111111': 7, '1010101011': 8, '1001010100': 9}
names = ['极镀ギ紬荕', '爷灬霸气傀儡', '梦战苍穹', '傲世哥', 'мaη肆風聲', '一刀メ隔世', '横刀メ绝杀', 'Q不死你R死你', '魔帝殤邪', '封刀不再战', '倾城孤狼', '戎马江湖', '狂得像风', '影之哀伤', '謸氕づ独尊', '傲视狂杀', '追风之梦', '枭雄在世', '傲视之巅', '黑夜刺客', '占你心为王', '爷来取你狗命', '御风踏血', '凫矢暮城', '孤影メ残刀', '野区霸王', '噬血啸月', '风逝无迹', '帅的睡不着', '血色杀戮者', '冷视天下', '帅出新高度', '風狆瑬蒗', '灵魂禁锢', 'ヤ地狱篮枫ゞ', '溅血メ破天', '剑尊メ杀戮', '塞外う飛龍', '哥‘K纯帅', '逆風祈雨', '恣意踏江山', '望断、天涯路', '地獄惡灵', '疯狂メ孽杀', '寂月灭影', '骚年霸称帝王', '狂杀メ无赦', '死灵的哀伤', '撩妹界扛把子', '霸刀☆藐视天下', '潇洒又能打', '狂卩龙灬巅丷峰', '羁旅天涯.', '南宫沐风', '风恋绝尘', '剑下孤魂', '一蓑烟雨', '领域★倾战', '威龙丶断魂神狙', '辉煌战绩', '屎来运赚', '伱、Bu够档次', '九音引魂箫', '骨子里的傲气', '霸海断长空', '没枪也很狂', '死魂★之灵']

lp = {}
for page in range(1, 6):
    res = requests.get(f'http://match.yuanrenxue.com/api/match/7?page={page}', headers={'User-Agent': 'yuanrenxue.project'}).json()

    with open(f'{page}.woff', 'wb') as f:
        f.write(base64.b64decode(res['woff']))
    font = TTFont(f'{page}.woff')
    font.saveXML(f'{page}.xml')
    font_list = parse(f'{page}.xml').documentElement.getElementsByTagName('TTGlyph')[1:]

    current_map = {}
    for i in font_list:
        name = i.getAttribute('name')
        pt = i.getElementsByTagName('pt')
        value = ''
        if len(pt) < 10:
            for j in range(len(pt)):
                value += pt[j].getAttribute('on')
        else:
            for j in range(10):
                value += pt[j].getAttribute('on')
        value = font_map[value]
        current_map[name.replace('uni', '&#x')] = value
    _lp = [i['value'].strip(' ').split(' ') for i in res['data']]

    flag = 1
    for i in range(len(_lp)):
        for j in range(len(_lp[i])):
            _lp[i][j] = str(current_map[_lp[i][j]])
        _lp[i] = ''.join(_lp[i])
        print(_lp[i])
        lp[names[flag + (page - 1) * 10]] = int(_lp[i])
        flag += 1

print(sorted(lp.items(), key=lambda kv: (kv[1], kv[0]), reverse=True))
# 冷视天下
```

{{% /spoiler %}}

写代码真的很开心（）

~~虽然我写的慢 但不妨碍我自我感觉良好~~

## 猿人学8 - 验证码 图文点选

> https://match.yuanrenxue.com/match/8
>
> 题目要求：通过验证码、抓取出现的数字，将5页全部数字中出现频率**最高**的数字（即：众数）填入答案

这个有点难了TAT

![image-20230105174126795](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230105174126795.png)

抓包可知api请求返回的是这一整张验证码的图片，用`<div>`标签把整个图片划成了30*30的方格，点击时的坐标会作为m参数再发出请求，如果验证成功会有Set-Cookie，带上这个sessionid就可以得到想要的数字了

我们一个个解决难点：

首先是图片处理，鉴于我不是misc手（）这里的代码是摘自已有wp的，有两种处理方式，纯PIL或cv2+numpy

- 纯PIL

请参考[这篇文章](https://www.52pojie.cn/thread-1288315-1-1.html)

- cv2+numpy

{{% spoiler "exp.py" %}}

```python
import base64
import re

import cv2
import numpy as np
import requests


def picProcess(img):
    _pic = cv2.imread(img)
    row, col = _pic.shape[0:2]
    _pic[np.all(_pic == [0, 0, 0], axis=-1)] = (255, 255, 255)  # 去掉黑点

    colors, counts = np.unique(np.array(_pic).reshape(-1, 3), axis=0, return_counts=True)  # 去除数组中的重复数字 排序
    info = {counts[i]: colors[i].tolist() for i, v in enumerate(counts) if 500 < int(v) < 2200}  # 筛选出现次数在500~2200之间的像素点
    remove_bg = info.values()
    pic = np.zeros((col, row, 3), np.uint8) + 255  # 生成全白底图
    for rgb in remove_bg:  # 循环 将不是噪点的像素盖到全白底图之上
        pic[np.all(_pic == rgb, axis=-1)] = _pic[np.all(_pic == rgb, axis=-1)]

    # 去掉线条 二值化
    line = []
    for x in range(row):
        for y in range(col):
            tmp = pic[y, x].tolist()
            if tmp != [0, 0, 0]:
                if 110 < x < 120 or 210 < x < 220:
                    line.append(tmp)
                if 100 < y < 110 or 200 < y < 210:
                    line.append(tmp)
    remove_line = np.unique(np.array(line).reshape(-1, 3), axis=0)
    for rgb in remove_line:
        pic[np.all(pic == rgb, axis=-1)] = [255, 255, 255]
    pic[np.any(pic != [255, 255, 255], axis=-1)] = [0, 0, 0]

    # 腐蚀 图片加深
    kernel = np.ones((2, 3), 'uint8')
    erode_pic = cv2.erode(pic, kernel, cv2.BORDER_REFLECT, iterations=2)
    cv2.imshow('Eroded Pic', erode_pic)
    cv2.waitKey(0)
    cv2.destroyAllWindows()


def verify():
    res = session.get('https://match.yuanrenxue.com/api/match/8_verify').json()['html']
    _words = re.findall(r'<p>(.*?)</p>', res)
    pic = re.findall(r'data:image/jpeg;base64,(.+?)"', res)[0]
    with open('verify_pic.png', 'wb') as f:
        f.write(base64.b64decode(pic.encode()))
    print(_words)
    return _words


def getResult(page, index_list):
    click_dict = {
        '1': 126, '2': 136, '3': 146,
        '4': 426, '5': 466, '6': 477,
        '7': 726, '8': 737, '9': 776
    }
    answer = '|'.join([str(click_dict[index]) for index in index_list]) + '|'
    params = {
        'page': page,
        'answer': answer
    }
    res = session.get('http://match.yuanrenxue.com/api/match/8', params=params).json()['data']
    try:
        value = [i['value'] for i in res]
        print(f'Page {page}: {value}')
        return value
    except:
        print(f'Page {page} meets error')
        getResult(page, index_list)


if __name__ == '__main__':
    session = requests.session()
    session.headers = {'User-Agent': 'yuanrenxue.project'}
    answer = []
    for i in range(1, 6):
        words = verify()
        picProcess('verify_pic.png')
        index = input('=>')
        answer.extend(getResult(i, list(index)))

    print(max(set(answer), key=answer.count))
# 7453
```

{{% /spoiler %}}

这里并没有用ocr，因为涉及到生僻字就比较弱鸡，而且这里也只有5个验证码，人力勉强可以（）

----

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[某网站Web端爬虫攻防大赛题目交流](https://www.52pojie.cn/thread-1288315-1-1.html)

[猿人学第二题,手撕OB混淆给你看(Step1-开篇)](https://juejin.cn/post/7073837033206054948)  |  [猿人学第二题,手撕OB混淆给你看(step2-字符串数字回填)](https://juejin.cn/post/7074076991422464036)  |  [猿人学第二题,手撕OB混淆给你看(step3-函数调用还原)](https://juejin.cn/post/7074397764108419102)  |  [猿人学第二题,手撕OB混淆给你看(step4-对象调用还原)](https://juejin.cn/post/7074768595825197092)  |  [猿人学第二题,手撕OB混淆给你看(step5-分支流程判断)](https://juejin.cn/post/7075165582949089287)  |  [猿人学第二题,手撕OB混淆给你看(step06-控制流平坦化)](https://juejin.cn/post/7075527384841060383)

[猿人学第5题,hook任意cookie被设置的瞬间](https://juejin.cn/post/7078321272202985485)

[Python反反爬之动态字体---随风漂移](https://syjun.vip/archives/283.html)

[Python反反爬之图文点击---生僻字验证码](https://syjun.vip/archives/284.html)

{{% /spoiler %}}