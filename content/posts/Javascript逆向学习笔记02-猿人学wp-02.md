---
title: "Javascript逆向学习笔记Ⅱ"
slug: "javascript-reverse-study-notes-02-02"
description: "猿人学js逆向题wp | 05~09"
date: 2023-01-05T22:48:43+08:00
categories: ["LTS"]
series: ["前端安全"]
tags: ["js", "wp"]
draft: true
toc: true
---

*书接上文，被迫因为渲染问题拆分为两个md

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

{{% spoiler "exp.py" %}}

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

最终要的是1，2，3等奖的金额之和，通过前端代码可知1等奖是3等奖的15倍，2等奖是3等奖的5倍，加起来就是api请求得到金额的24倍

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

唯一的难点就是图片处理了，鉴于我不是misc手（）这部分的代码是摘自已有wp的，有两种处理方式，纯PIL或cv2+numpy

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

## 猿人学9 - js混淆 动态cookie2

> https://match.yuanrenxue.com/match/9
>
> 题目要求：采集5个页面，所有商家的评论数量，计算**平均数**填入答案中

这次又是加密cookie 字段还是m，惯例hook发现根本断不下来，我不知道他是怎么检测cookie是否失效的，只要我开一次f12就陷入无限循环中，即使我清除缓存硬刷新也提示cookie失效，唯一在hook的地方断下来还没找到m

抓包，和之前动态cookie的一样 请求/match/9会有巨量的混淆过的js代码，拖出来看看

![image-20230106102221146](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230106102221146.png)

一眼顶针 鉴定为ob混淆，虽然用解混淆工具不能完全还原控制流，但可以做到部分字面量还原，比如cookie部分：

```js
for (var m = 1; d[$b("0x38", "qQT*") + "as"](m, 4); m++) {
    res = d[$b("0x31", "I$S%") + "kN"](decrypt, '1673231820') + "r";
}
document["coo" + $b("0x26", "*9)]")] = d["QXuQR"](d[$b("0x112", "2G^G") + "LJ"]("m=" + (m - 1)[$b("0x60", "jG$O") + $b("0x48", "9bt3") + "ng"(), res), d[$b("0x4c", "esrd") + "QU"]);
```

手动进行二次简化

```js
for (var m = 0x1; m <= 3; m++) {
	res = decrypt('1673231820') + 'r';
}
document['cookie'] = ('m=' + (m - 0x1)['toString']()) + res + '; path=/'
```

这里核心的的`decrypt`是开头引入的udc.js中的一个函数，参数是请求时的时间戳，不过是写死的 并不是js再生成的，除此之外这个js文件就没啥用了

按理说分析到这里代码就不难了，只需要构造一个一样的函数来调用udc中的decrypt即可，我们用解混淆的工具来处理，但是运行总是报错

```
RangeError: Invalid string length
```

代码中的`+=`总是超过v8的长度限制，能力有限不知道怎么改，查找wp才发现是我们解混淆还不够狠（）这里使用这位师傅修改过的[ob-decrypt](https://github.com/skygongque/ob-decrypt)，将生成的代码做以下修改

- 删除`window["decrypt"] = _0x4f6d79;`之后的所有代码
- 删除以下部分代码（是针对环境的检测）

```js
if (window["crypto"] && window["crypto"]["getRandomValues"]) {} else {
  global = new Array();
  window = new Array();
}
```

- 开头补环境

```js
window = global
var navigator = {}
```

- 末尾添加

```js
function getResult(timestamp){
  for (var m = 0x1; m <= 3; m++) {
	var res = decrypt(timestamp) + 'r';
  }
  var cookie = (m - 0x1)['toString']() + res;
  console.log(cookie)
  return cookie
}
```

编写最后的exp，因为timestamp是写死在返回内容里的 所以还要用个正则

```python
import requests
import re
import subprocess
from functools import partial

subprocess.Popen = partial(subprocess.Popen, encoding='utf-8')
session = requests.session()
headers = {'User-Agent': 'yuanrenxue.project'}
session.headers.update(headers)

url = "http://match.yuanrenxue.com/match/9"
_res = session.get(url).text
if _timestamp := re.search(r"decrypt,'(\d+)'", _res, re.S):
    timestamp = _timestamp.group(1)
    with open('udc_deob2.js', 'r', encoding='utf-8') as f:
        import execjs
        ctx = execjs.compile(f.read())
        cookie = ctx.call('getResult', timestamp)

    m = requests.cookies.RequestsCookieJar()
    m.set('m', cookie)
    session.cookies.update(m)

    value = []
    for page in range(1, 6):
        res = session.get(f'http://match.yuanrenxue.com/api/match/9?page={page}')
        value += res.json()['data']

    total = 0
    count = 0
    for i in value:
        count += 1
        total += i['value']

    print('total', total)
    print('average', total / count)
else:
    print('no timestamp')
    exit(1)
# 4900
```

```js
// atob函数，后面可能会判断其是否存在，勿删！
window = global
var navigator = {}
!(function () {
  var _0x4f1af6 = typeof window !== "undefined" ? window : typeof process === "object" && typeof require === "function" && typeof global === "object" ? global : this;
  var _0x1e4ec6 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=";
  _0x4f1af6.atob || (_0x4f1af6.atob = function (_0x4968c4) {
    var _0x21936c = String(_0x4968c4).replace(/=+$/, "");
    for (var _0x38f546 = 0x0, _0x51294c, _0x4744d8, _0x13e6de = 0x0, _0x5794e1 = ""; _0x4744d8 = _0x21936c.charAt(_0x13e6de++); ~_0x4744d8 && (_0x51294c = _0x38f546 % 0x4 ? _0x51294c * 0x40 + _0x4744d8 : _0x4744d8, _0x38f546++ % 0x4) ? _0x5794e1 += String.fromCharCode(0xff & _0x51294c >> (-0x2 * _0x38f546 & 0x6)) : 0x0) {
      _0x4744d8 = _0x1e4ec6.indexOf(_0x4744d8);
    }
    return _0x5794e1;
  });
})();
!function (_0x53bae5, _0x153ef4) {
  if ("object" == typeof exports && "undefined" != typeof module) {
    _0x153ef4(exports);
  } else {
    if ("function" == typeof define && define["amd"]) {
      define(["exports"], _0x153ef4);
    } else {
      _0x153ef4(_0x53bae5["JSEncrypt"] = {});
    }
  }
}(this, function (_0x20544c) {
  "use strict";
  var _0x58c808 = "0123456789abcdefghijklmnopqrstuvwxyz";
  function _0x4e31bb(_0x10a3c5) {
    return _0x58c808["charAt"](_0x10a3c5);
  }
  function _0x4273b2(_0x264e76, _0x11e1ea) {
    return _0x264e76 & _0x11e1ea;
  }
  function _0xfb232b(_0x5495a1, _0xc02b9e) {
    return _0x5495a1 | _0xc02b9e;
  }
  function _0x3eba95(_0x539dd8, _0x59bb2f) {
    return _0x539dd8 ^ _0x59bb2f;
  }
  function _0x1e8fa0(_0x7b2e5b, _0x119827) {
    return _0x7b2e5b & ~_0x119827;
  }
  function _0x525b4a(_0x5a7bd7) {
    if (0 == _0x5a7bd7) return -1;
    var _0x1049e5 = 0;
    return 0 == (65535 & _0x5a7bd7) && (_0x5a7bd7 >>= 16, _0x1049e5 += 16), 0 == (255 & _0x5a7bd7) && (_0x5a7bd7 >>= 8, _0x1049e5 += 8), 0 == (15 & _0x5a7bd7) && (_0x5a7bd7 >>= 4, _0x1049e5 += 4), 0 == (3 & _0x5a7bd7) && (_0x5a7bd7 >>= 2, _0x1049e5 += 2), 0 == (1 & _0x5a7bd7) && ++_0x1049e5, _0x1049e5;
  }
  function _0xbc2d31(_0x371ef4) {
    for (var _0x280ad5 = 0; 0 != _0x371ef4;) {
      _0x371ef4 &= _0x371ef4 - 1;
      ++_0x280ad5;
    }
    return _0x280ad5;
  }
  var _0x407614 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
  function _0x125db2(_0x3d3880) {
    var _0x1b5e9b,
      _0x4c43ce,
      _0x1b4ee6 = "";
    for (_0x1b5e9b = 0; _0x1b5e9b + 3 <= _0x3d3880["length"]; _0x1b5e9b += 3) {
      _0x4c43ce = parseInt(_0x3d3880["substring"](_0x1b5e9b, _0x1b5e9b + 3), 16);
      _0x1b4ee6 += _0x407614["charAt"](_0x4c43ce >> 6) + _0x407614["charAt"](63 & _0x4c43ce);
    }
    for (_0x1b5e9b + 1 == _0x3d3880["length"] ? (_0x4c43ce = parseInt(_0x3d3880["substring"](_0x1b5e9b, _0x1b5e9b + 1), 16), _0x1b4ee6 += _0x407614["charAt"](_0x4c43ce << 2)) : _0x1b5e9b + 2 == _0x3d3880["length"] && (_0x4c43ce = parseInt(_0x3d3880["substring"](_0x1b5e9b, _0x1b5e9b + 2), 16), _0x1b4ee6 += _0x407614["charAt"](_0x4c43ce >> 2) + _0x407614["charAt"]((3 & _0x4c43ce) << 4)); 0 < (3 & _0x1b4ee6["length"]);) _0x1b4ee6 += "=";
    return _0x1b4ee6;
  }
  function _0x5c2128(_0x3ae9d8) {
    var _0x5dbe6d,
      _0x483fc9 = "",
      _0x196962 = 0,
      _0x4eb25a = 0;
    for (_0x5dbe6d = 0; _0x5dbe6d < _0x3ae9d8["length"] && "=" != _0x3ae9d8["charAt"](_0x5dbe6d); ++_0x5dbe6d) {
      var _0x4ce454 = _0x407614["indexOf"](_0x3ae9d8["charAt"](_0x5dbe6d));
      _0x4ce454 < 0 || (0 == _0x196962 ? (_0x483fc9 += _0x4e31bb(_0x4ce454 >> 2), _0x4eb25a = 3 & _0x4ce454, _0x196962 = 1) : 1 == _0x196962 ? (_0x483fc9 += _0x4e31bb(_0x4eb25a << 2 | _0x4ce454 >> 4), _0x4eb25a = 15 & _0x4ce454, _0x196962 = 2) : 2 == _0x196962 ? (_0x483fc9 += _0x4e31bb(_0x4eb25a), _0x483fc9 += _0x4e31bb(_0x4ce454 >> 2), _0x4eb25a = 3 & _0x4ce454, _0x196962 = 3) : (_0x483fc9 += _0x4e31bb(_0x4eb25a << 2 | _0x4ce454 >> 4), _0x483fc9 += _0x4e31bb(15 & _0x4ce454), _0x196962 = 0));
    }
    return 1 == _0x196962 && (_0x483fc9 += _0x4e31bb(_0x4eb25a << 2)), _0x483fc9;
  }
  var _0x465910,
    _0xd5e875,
    _0x556c8d = function (_0x3d0df2, _0x17b610) {
      return (_0x556c8d = Object["setPrototypeOf"] || {
        "__proto__": []
      } instanceof Array && function (_0x2cf4e9, _0x556f9f) {
        _0x2cf4e9["__proto__"] = _0x556f9f;
      } || function (_0x13aece, _0x3bc240) {
        for (var _0x3ee4b8 in _0x3bc240) _0x3bc240["hasOwnProperty"](_0x3ee4b8) && (_0x13aece[_0x3ee4b8] = _0x3bc240[_0x3ee4b8]);
      })(_0x3d0df2, _0x17b610);
    },
    _0x5a02a1 = {
      "decode": function (_0xb9450d) {
        var _0xed6233;
        if (void 0 === _0xd5e875) {
          var _0x4250e6 = "= \f\n\r\t \u2028\u2029";
          for (_0xd5e875 = Object["create"](null), _0xed6233 = 0; _0xed6233 < 64; ++_0xed6233) _0xd5e875["ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"["charAt"](_0xed6233)] = _0xed6233;
          for (_0xed6233 = 0; _0xed6233 < _0x4250e6["length"]; ++_0xed6233) _0xd5e875[_0x4250e6["charAt"](_0xed6233)] = -1;
        }
        var _0x327d03 = [],
          _0xde2f63 = 0,
          _0x5e70c2 = 0;
        for (_0xed6233 = 0; _0xed6233 < _0xb9450d["length"]; ++_0xed6233) {
          var _0x1772b4 = _0xb9450d["charAt"](_0xed6233);
          if ("=" == _0x1772b4) break;
          if (-1 != (_0x1772b4 = _0xd5e875[_0x1772b4])) {
            if (void 0 === _0x1772b4) throw new Error("Illegal character at offset " + _0xed6233);
            _0xde2f63 |= _0x1772b4;
            if (4 <= ++_0x5e70c2) {
              _0x327d03[_0x327d03["length"]] = _0xde2f63 >> 16, _0x327d03[_0x327d03["length"]] = _0xde2f63 >> 8 & 255, _0x327d03[_0x327d03["length"]] = 255 & _0xde2f63, _0x5e70c2 = _0xde2f63 = 0;
            } else {
              _0xde2f63 <<= 6;
            }
          }
        }
        switch (_0x5e70c2) {
          case 1:
            throw new Error("Base64 encoding incomplete: at least 2 bits missing");
          case 2:
            _0x327d03[_0x327d03["length"]] = _0xde2f63 >> 10;
            break;
          case 3:
            _0x327d03[_0x327d03["length"]] = _0xde2f63 >> 16;
            _0x327d03[_0x327d03["length"]] = _0xde2f63 >> 8 & 255;
        }
        return _0x327d03;
      },
      "re": /-----BEGIN [^-]+-----([A-Za-z0-9+\/=\s]+)-----END [^-]+-----|begin-base64[^\n]+\n([A-Za-z0-9+\/=\s]+)====/,
      "unarmor": function (_0x22af7d) {
        var _0x5bdf97 = _0x5a02a1["re"]["exec"](_0x22af7d);
        if (_0x5bdf97) if (_0x5bdf97[1]) _0x22af7d = _0x5bdf97[1];else {
          if (!_0x5bdf97[2]) throw new Error("RegExp out of sync");
          _0x22af7d = _0x5bdf97[2];
        }
        return _0x5a02a1["decode"](_0x22af7d);
      }
    },
    _0x325070 = 10000000000000,
    _0x3b9155 = function () {
      function _0x5bc042(_0x4edd84) {
        this["buf"] = [+_0x4edd84 || 0];
      }
      return _0x5bc042["prototype"]["mulAdd"] = function (_0x136455, _0x38d44a) {
        var _0x25cdbb,
          _0x4b209c,
          _0x223e40 = this["buf"],
          _0x107191 = _0x223e40["length"];
        for (_0x25cdbb = 0; _0x25cdbb < _0x107191; ++_0x25cdbb) {
          if ((_0x4b209c = _0x223e40[_0x25cdbb] * _0x136455 + _0x38d44a) < _0x325070) {
            _0x38d44a = 0;
          } else {
            _0x4b209c -= (_0x38d44a = 0 | _0x4b209c / _0x325070) * _0x325070;
          }
          _0x223e40[_0x25cdbb] = _0x4b209c;
        }
        0 < _0x38d44a && (_0x223e40[_0x25cdbb] = _0x38d44a);
      }, _0x5bc042["prototype"]["sub"] = function (_0x200f20) {
        var _0x458211,
          _0x2d10d4,
          _0x35886b = this["buf"],
          _0x23e1f0 = _0x35886b["length"];
        for (_0x458211 = 0; _0x458211 < _0x23e1f0; ++_0x458211) {
          if ((_0x2d10d4 = _0x35886b[_0x458211] - _0x200f20) < 0) {
            _0x2d10d4 += _0x325070, _0x200f20 = 1;
          } else {
            _0x200f20 = 0;
          }
          _0x35886b[_0x458211] = _0x2d10d4;
        }
        for (; 0 === _0x35886b[_0x35886b["length"] - 1];) _0x35886b["pop"]();
      }, _0x5bc042["prototype"]["toString"] = function (_0x594d75) {
        if (10 != (_0x594d75 || 10)) throw new Error("only base 10 is supported");
        for (var _0x3268ae = this["buf"], _0xb46b04 = _0x3268ae[_0x3268ae["length"] - 1]["toString"](), _0x56a312 = _0x3268ae["length"] - 2; 0 <= _0x56a312; --_0x56a312) _0xb46b04 += (_0x325070 + _0x3268ae[_0x56a312])["toString"]()["substring"](1);
        return _0xb46b04;
      }, _0x5bc042["prototype"]["valueOf"] = function () {
        for (var _0x5bc042 = this["buf"], _0x481f52 = 0, _0x15cd06 = _0x5bc042["length"] - 1; 0 <= _0x15cd06; --_0x15cd06) _0x481f52 = _0x481f52 * _0x325070 + _0x5bc042[_0x15cd06];
        return _0x481f52;
      }, _0x5bc042["prototype"]["simplify"] = function () {
        var _0x5bc042 = this["buf"];
        return 1 == _0x5bc042["length"] ? _0x5bc042[0] : this;
      }, _0x5bc042;
    }(),
    _0x22eb45 = "…",
    _0x5a96f4 = /^(\d\d)(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])([01]\d|2[0-3])(?:([0-5]\d)(?:([0-5]\d)(?:[.,](\d{1,3}))?)?)?(Z|[-+](?:[0]\d|1[0-2])([0-5]\d)?)?$/,
    _0x256f93 = /^(\d\d\d\d)(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])([01]\d|2[0-3])(?:([0-5]\d)(?:([0-5]\d)(?:[.,](\d{1,3}))?)?)?(Z|[-+](?:[0]\d|1[0-2])([0-5]\d)?)?$/;
  function _0x52a054(_0x12d041, _0x24a766) {
    return _0x12d041["length"] > _0x24a766 && (_0x12d041 = _0x12d041["substring"](0, _0x24a766) + _0x22eb45), _0x12d041;
  }
  var _0x10b29a,
    _0x6f15d5 = function () {
      function _0x22068b(_0x2fd898, _0x383975) {
        this["hexDigits"] = "0123456789ABCDEF";
        if (_0x2fd898 instanceof _0x22068b) {
          this["enc"] = _0x2fd898["enc"], this["pos"] = _0x2fd898["pos"];
        } else {
          this["enc"] = _0x2fd898, this["pos"] = _0x383975;
        }
      }
      return _0x22068b["prototype"]["get"] = function (_0x5823f1) {
        if (void 0 === _0x5823f1 && (_0x5823f1 = this["pos"]++), _0x5823f1 >= this["enc"]["length"]) throw new Error("Requesting byte offset " + _0x5823f1 + " on a stream of length " + this["enc"]["length"]);
        return "string" == typeof this["enc"] ? this["enc"]["charCodeAt"](_0x5823f1) : this["enc"][_0x5823f1];
      }, _0x22068b["prototype"]["hexByte"] = function (_0x1aab46) {
        return this["hexDigits"]["charAt"](_0x1aab46 >> 4 & 15) + this["hexDigits"]["charAt"](15 & _0x1aab46);
      }, _0x22068b["prototype"]["hexDump"] = function (_0x1c3cb1, _0x1fb83a, _0x5c0107) {
        for (var _0x56bcdb = "", _0x1a5d02 = _0x1c3cb1; _0x1a5d02 < _0x1fb83a; ++_0x1a5d02) if (_0x56bcdb += this["hexByte"](this["get"](_0x1a5d02)), !0 !== _0x5c0107) switch (15 & _0x1a5d02) {
          case 7:
            _0x56bcdb += "  ";
            break;
          case 15:
            _0x56bcdb += "\n";
            break;
          default:
            _0x56bcdb += " ";
        }
        return _0x56bcdb;
      }, _0x22068b["prototype"]["isASCII"] = function (_0x378015, _0x3fc4f1) {
        for (var _0x34a0e8 = _0x378015; _0x34a0e8 < _0x3fc4f1; ++_0x34a0e8) {
          var _0x115402 = this["get"](_0x34a0e8);
          if (_0x115402 < 32 || 176 < _0x115402) return !1;
        }
        return !0;
      }, _0x22068b["prototype"]["parseStringISO"] = function (_0x10ba28, _0x36143f) {
        for (var _0x4438c1 = "", _0x30c77d = _0x10ba28; _0x30c77d < _0x36143f; ++_0x30c77d) _0x4438c1 += String["fromCharCode"](this["get"](_0x30c77d));
        return _0x4438c1;
      }, _0x22068b["prototype"]["parseStringUTF"] = function (_0x12a1ad, _0x268684) {
        for (var _0x3f7963 = "", _0x55eef7 = _0x12a1ad; _0x55eef7 < _0x268684;) {
          var _0x588802 = this["get"](_0x55eef7++);
          _0x3f7963 += _0x588802 < 128 ? String["fromCharCode"](_0x588802) : 191 < _0x588802 && _0x588802 < 224 ? String["fromCharCode"]((31 & _0x588802) << 6 | 63 & this["get"](_0x55eef7++)) : String["fromCharCode"]((15 & _0x588802) << 12 | (63 & this["get"](_0x55eef7++)) << 6 | 63 & this["get"](_0x55eef7++));
        }
        return _0x3f7963;
      }, _0x22068b["prototype"]["parseStringBMP"] = function (_0x2c3465, _0x4f9411) {
        for (var _0x5d0426, _0x2f17e7, _0x36c739 = "", _0x221f11 = _0x2c3465; _0x221f11 < _0x4f9411;) {
          _0x5d0426 = this["get"](_0x221f11++);
          _0x2f17e7 = this["get"](_0x221f11++);
          _0x36c739 += String["fromCharCode"](_0x5d0426 << 8 | _0x2f17e7);
        }
        return _0x36c739;
      }, _0x22068b["prototype"]["parseTime"] = function (_0x48e12c, _0xaea7b4, _0x17adc5) {
        var _0x3d8dd8 = this["parseStringISO"](_0x48e12c, _0xaea7b4),
          _0x3cefb8 = (_0x17adc5 ? _0x5a96f4 : _0x256f93)["exec"](_0x3d8dd8);
        return _0x3cefb8 ? (_0x17adc5 && (_0x3cefb8[1] = +_0x3cefb8[1], _0x3cefb8[1] += +_0x3cefb8[1] < 70 ? 2000 : 1900), _0x3d8dd8 = _0x3cefb8[1] + "-" + _0x3cefb8[2] + "-" + _0x3cefb8[3] + " " + _0x3cefb8[4], _0x3cefb8[5] && (_0x3d8dd8 += ":" + _0x3cefb8[5], _0x3cefb8[6] && (_0x3d8dd8 += ":" + _0x3cefb8[6], _0x3cefb8[7] && (_0x3d8dd8 += "." + _0x3cefb8[7]))), _0x3cefb8[8] && (_0x3d8dd8 += " UTC", "Z" != _0x3cefb8[8] && (_0x3d8dd8 += _0x3cefb8[8], _0x3cefb8[9] && (_0x3d8dd8 += ":" + _0x3cefb8[9]))), _0x3d8dd8) : "Unrecognized time: " + _0x3d8dd8;
      }, _0x22068b["prototype"]["parseInteger"] = function (_0x32b610, _0x35f119) {
        for (var _0x2c8e7d, _0x38d373 = this["get"](_0x32b610), _0x13c3f1 = 127 < _0x38d373, _0x29fedd = _0x13c3f1 ? 255 : 0, _0x5726c2 = ""; _0x38d373 == _0x29fedd && ++_0x32b610 < _0x35f119;) _0x38d373 = this["get"](_0x32b610);
        if (0 == (_0x2c8e7d = _0x35f119 - _0x32b610)) return _0x13c3f1 ? -1 : 0;
        if (4 < _0x2c8e7d) {
          for (_0x5726c2 = _0x38d373, _0x2c8e7d <<= 3; 0 == (128 & (+_0x5726c2 ^ _0x29fedd));) {
            _0x5726c2 = +_0x5726c2 << 1;
            --_0x2c8e7d;
          }
          _0x5726c2 = "(" + _0x2c8e7d + " bit)\n";
        }
        _0x13c3f1 && (_0x38d373 -= 256);
        for (var _0x43f83d = new _0x3b9155(_0x38d373), _0x725f7e = _0x32b610 + 1; _0x725f7e < _0x35f119; ++_0x725f7e) _0x43f83d["mulAdd"](256, this["get"](_0x725f7e));
        return _0x5726c2 + _0x43f83d["toString"]();
      }, _0x22068b["prototype"]["parseBitString"] = function (_0x584802, _0x125a75, _0x4330aa) {
        for (var _0x2b49b7 = this["get"](_0x584802), _0x5a3d66 = "(" + ((_0x125a75 - _0x584802 - 1 << 3) - _0x2b49b7) + " bit)\n", _0x37def6 = "", _0x68b524 = _0x584802 + 1; _0x68b524 < _0x125a75; ++_0x68b524) {
          for (var _0x458110 = this["get"](_0x68b524), _0x5edd1e = _0x68b524 == _0x125a75 - 1 ? _0x2b49b7 : 0, _0x44cc33 = 7; _0x5edd1e <= _0x44cc33; --_0x44cc33) _0x37def6 += _0x458110 >> _0x44cc33 & 1 ? "1" : "0";
          if (_0x37def6["length"] > _0x4330aa) return _0x5a3d66 + _0x52a054(_0x37def6, _0x4330aa);
        }
        return _0x5a3d66 + _0x37def6;
      }, _0x22068b["prototype"]["parseOctetString"] = function (_0x3f709b, _0x5249ed, _0xfca96d) {
        if (this["isASCII"](_0x3f709b, _0x5249ed)) return _0x52a054(this["parseStringISO"](_0x3f709b, _0x5249ed), _0xfca96d);
        var _0x4063a5 = _0x5249ed - _0x3f709b,
          _0x2f9857 = "(" + _0x4063a5 + " byte)\n";
        (_0xfca96d /= 2) < _0x4063a5 && (_0x5249ed = _0x3f709b + _0xfca96d);
        for (var _0x5a4777 = _0x3f709b; _0x5a4777 < _0x5249ed; ++_0x5a4777) _0x2f9857 += this["hexByte"](this["get"](_0x5a4777));
        return _0xfca96d < _0x4063a5 && (_0x2f9857 += _0x22eb45), _0x2f9857;
      }, _0x22068b["prototype"]["parseOID"] = function (_0x32a2a6, _0x26433c, _0xe66585) {
        for (var _0x3af3a5 = "", _0x109e46 = new _0x3b9155(), _0x10e125 = 0, _0x1b5e95 = _0x32a2a6; _0x1b5e95 < _0x26433c; ++_0x1b5e95) {
          var _0x27b0d0 = this["get"](_0x1b5e95);
          if (_0x109e46["mulAdd"](128, 127 & _0x27b0d0), _0x10e125 += 7, !(128 & _0x27b0d0)) {
            if ("" === _0x3af3a5) {
              if ((_0x109e46 = _0x109e46["simplify"]()) instanceof _0x3b9155) {
                _0x109e46["sub"](80);
                _0x3af3a5 = "2." + _0x109e46["toString"]();
              } else {
                var _0x473690 = _0x109e46 < 80 ? _0x109e46 < 40 ? 0 : 1 : 2;
                _0x3af3a5 = _0x473690 + "." + (_0x109e46 - 40 * _0x473690);
              }
            } else _0x3af3a5 += "." + _0x109e46["toString"]();
            if (_0x3af3a5["length"] > _0xe66585) return _0x52a054(_0x3af3a5, _0xe66585);
            _0x109e46 = new _0x3b9155();
            _0x10e125 = 0;
          }
        }
        return 0 < _0x10e125 && (_0x3af3a5 += ".incomplete"), _0x3af3a5;
      }, _0x22068b;
    }(),
    _0x408860 = function () {
      function _0x341895(_0x2c0c31, _0x401ea8, _0xbf59c8, _0x1ba541, _0x912511) {
        if (!(_0x1ba541 instanceof _0x463aba)) throw new Error("Invalid tag value.");
        this["stream"] = _0x2c0c31;
        this["header"] = _0x401ea8;
        this["length"] = _0xbf59c8;
        this["tag"] = _0x1ba541;
        this["sub"] = _0x912511;
      }
      return _0x341895["prototype"]["typeName"] = function () {
        switch (this["tag"]["tagClass"]) {
          case 0:
            switch (this["tag"]["tagNumber"]) {
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
            }
            return "Universal_" + this["tag"]["tagNumber"]["toString"]();
          case 1:
            return "Application_" + this["tag"]["tagNumber"]["toString"]();
          case 2:
            return "[" + this["tag"]["tagNumber"]["toString"]() + "]";
          case 3:
            return "Private_" + this["tag"]["tagNumber"]["toString"]();
        }
      }, _0x341895["prototype"]["content"] = function (_0x6e4ee1) {
        if (void 0 === this["tag"]) return null;
        void 0 === _0x6e4ee1 && (_0x6e4ee1 = 1 / 0);
        var _0x5b9d1b = this["posContent"](),
          _0x1baaf9 = Math["abs"](this["length"]);
        if (!this["tag"]["isUniversal"]()) return null !== this["sub"] ? "(" + this["sub"]["length"] + " elem)" : this["stream"]["parseOctetString"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9, _0x6e4ee1);
        switch (this["tag"]["tagNumber"]) {
          case 1:
            return 0 === this["stream"]["get"](_0x5b9d1b) ? "false" : "true";
          case 2:
            return this["stream"]["parseInteger"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9);
          case 3:
            return this["sub"] ? "(" + this["sub"]["length"] + " elem)" : this["stream"]["parseBitString"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9, _0x6e4ee1);
          case 4:
            return this["sub"] ? "(" + this["sub"]["length"] + " elem)" : this["stream"]["parseOctetString"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9, _0x6e4ee1);
          case 6:
            return this["stream"]["parseOID"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9, _0x6e4ee1);
          case 16:
          case 17:
            return null !== this["sub"] ? "(" + this["sub"]["length"] + " elem)" : "(no elem)";
          case 12:
            return _0x52a054(this["stream"]["parseStringUTF"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9), _0x6e4ee1);
          case 18:
          case 19:
          case 20:
          case 21:
          case 22:
          case 26:
            return _0x52a054(this["stream"]["parseStringISO"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9), _0x6e4ee1);
          case 30:
            return _0x52a054(this["stream"]["parseStringBMP"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9), _0x6e4ee1);
          case 23:
          case 24:
            return this["stream"]["parseTime"](_0x5b9d1b, _0x5b9d1b + _0x1baaf9, 23 == this["tag"]["tagNumber"]);
        }
        return null;
      }, _0x341895["prototype"]["toString"] = function () {
        return this["typeName"]() + "@" + this["stream"]["pos"] + "[header:" + this["header"] + ",length:" + this["length"] + ",sub:" + (null === this["sub"] ? "null" : this["sub"]["length"]) + "]";
      }, _0x341895["prototype"]["toPrettyString"] = function (_0x3682c4) {
        void 0 === _0x3682c4 && (_0x3682c4 = "");
        var _0x42fe60 = _0x3682c4 + this["typeName"]() + " @" + this["stream"]["pos"];
        if (0 <= this["length"] && (_0x42fe60 += "+"), _0x42fe60 += this["length"], this["tag"]["tagConstructed"] ? _0x42fe60 += " (constructed)" : !this["tag"]["isUniversal"]() || 3 != this["tag"]["tagNumber"] && 4 != this["tag"]["tagNumber"] || null === this["sub"] || (_0x42fe60 += " (encapsulates)"), _0x42fe60 += "\n", null !== this["sub"]) {
          _0x3682c4 += "  ";
          for (var _0x3aa62e = 0, _0x1a1999 = this["sub"]["length"]; _0x3aa62e < _0x1a1999; ++_0x3aa62e) _0x42fe60 += this["sub"][_0x3aa62e]["toPrettyString"](_0x3682c4);
        }
        return _0x42fe60;
      }, _0x341895["prototype"]["posStart"] = function () {
        return this["stream"]["pos"];
      }, _0x341895["prototype"]["posContent"] = function () {
        return this["stream"]["pos"] + this["header"];
      }, _0x341895["prototype"]["posEnd"] = function () {
        return this["stream"]["pos"] + this["header"] + Math["abs"](this["length"]);
      }, _0x341895["prototype"]["toHexString"] = function () {
        return this["stream"]["hexDump"](this["posStart"](), this["posEnd"](), !0);
      }, _0x341895["decodeLength"] = function (_0x2dc1ea) {
        var _0x305e45 = _0x2dc1ea["get"](),
          _0x3cf1d5 = 127 & _0x305e45;
        if (_0x3cf1d5 == _0x305e45) return _0x3cf1d5;
        if (6 < _0x3cf1d5) throw new Error("Length over 48 bits not supported at position " + (_0x2dc1ea["pos"] - 1));
        if (0 === _0x3cf1d5) return null;
        for (var _0x29d41e = _0x305e45 = 0; _0x29d41e < _0x3cf1d5; ++_0x29d41e) _0x305e45 = 256 * _0x305e45 + _0x2dc1ea["get"]();
        return _0x305e45;
      }, _0x341895["prototype"]["getHexStringValue"] = function () {
        return this["toHexString"]()["substr"](2 * this["header"], 2 * this["length"]);
      }, _0x341895["decode"] = function (_0xb52542) {
        var _0x353bc2;
        if (_0xb52542 instanceof _0x6f15d5) {
          _0x353bc2 = _0xb52542;
        } else {
          _0x353bc2 = new _0x6f15d5(_0xb52542, 0);
        }
        var _0x811f = new _0x6f15d5(_0x353bc2),
          _0x3cc364 = new _0x463aba(_0x353bc2),
          _0x32ebe8 = _0x341895["decodeLength"](_0x353bc2),
          _0x701f57 = _0x353bc2["pos"],
          _0x40bafc = _0x701f57 - _0x811f["pos"],
          _0x32b6cf = null,
          _0x445e77 = function () {
            var _0xb52542 = [];
            if (null !== _0x32ebe8) {
              for (var _0x196f12 = _0x701f57 + _0x32ebe8; _0x353bc2["pos"] < _0x196f12;) _0xb52542[_0xb52542["length"]] = _0x341895["decode"](_0x353bc2);
              if (_0x353bc2["pos"] != _0x196f12) throw new Error("Content size is not correct for container starting at offset " + _0x701f57);
            } else try {
              for (;;) {
                var _0x1757b5 = _0x341895["decode"](_0x353bc2);
                if (_0x1757b5["tag"]["isEOC"]()) break;
                _0xb52542[_0xb52542["length"]] = _0x1757b5;
              }
              _0x32ebe8 = _0x701f57 - _0x353bc2["pos"];
            } catch (_0x4a78b4) {
              throw new Error("Exception while decoding undefined length content: " + _0x4a78b4);
            }
            return _0xb52542;
          };
        if (_0x3cc364["tagConstructed"]) _0x32b6cf = _0x445e77();else if (_0x3cc364["isUniversal"]() && (3 == _0x3cc364["tagNumber"] || 4 == _0x3cc364["tagNumber"])) try {
          if (3 == _0x3cc364["tagNumber"] && 0 != _0x353bc2["get"]()) throw new Error("BIT STRINGs with unused bits cannot encapsulate.");
          _0x32b6cf = _0x445e77();
          for (var _0x54cfa8 = 0; _0x54cfa8 < _0x32b6cf["length"]; ++_0x54cfa8) if (_0x32b6cf[_0x54cfa8]["tag"]["isEOC"]()) throw new Error("EOC is not supposed to be actual content.");
        } catch (_0x29f704) {
          _0x32b6cf = null;
        }
        if (null === _0x32b6cf) {
          if (null === _0x32ebe8) throw new Error("We can't skip over an invalid tag with undefined length at offset " + _0x701f57);
          _0x353bc2["pos"] = _0x701f57 + Math["abs"](_0x32ebe8);
        }
        return new _0x341895(_0x811f, _0x40bafc, _0x32ebe8, _0x3cc364, _0x32b6cf);
      }, _0x341895;
    }(),
    _0x463aba = function () {
      function _0x4eb230(_0x4cc1b4) {
        var _0x2513f2 = _0x4cc1b4["get"]();
        if (this["tagClass"] = _0x2513f2 >> 6, this["tagConstructed"] = 0 != (32 & _0x2513f2), this["tagNumber"] = 31 & _0x2513f2, 31 == this["tagNumber"]) {
          for (var _0x1e3706 = new _0x3b9155(); _0x2513f2 = _0x4cc1b4["get"](), _0x1e3706["mulAdd"](128, 127 & _0x2513f2), 128 & _0x2513f2;);
          this["tagNumber"] = _0x1e3706["simplify"]();
        }
      }
      return _0x4eb230["prototype"]["isUniversal"] = function () {
        return 0 === this["tagClass"];
      }, _0x4eb230["prototype"]["isEOC"] = function () {
        return 0 === this["tagClass"] && 0 === this["tagNumber"];
      }, _0x4eb230;
    }(),
    _0x16c700 = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101, 103, 107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163, 167, 173, 179, 181, 191, 193, 197, 199, 211, 223, 227, 229, 233, 239, 241, 251, 257, 263, 269, 271, 277, 281, 283, 293, 307, 311, 313, 317, 331, 337, 347, 349, 353, 359, 367, 373, 379, 383, 389, 397, 401, 409, 419, 421, 431, 433, 439, 443, 449, 457, 461, 463, 467, 479, 487, 491, 499, 503, 509, 521, 523, 541, 547, 557, 563, 569, 571, 577, 587, 593, 599, 601, 607, 613, 617, 619, 631, 641, 643, 647, 653, 659, 661, 673, 677, 683, 691, 701, 709, 719, 727, 733, 739, 743, 751, 757, 761, 769, 773, 787, 797, 809, 811, 821, 823, 827, 829, 839, 853, 857, 859, 863, 877, 881, 883, 887, 907, 911, 919, 929, 937, 941, 947, 953, 967, 971, 977, 983, 991, 997],
    _0x1777d0 = (1 << 26) / _0x16c700[_0x16c700["length"] - 1],
    _0x2935af = function () {
      function _0x46cc13(_0x3c1c1e, _0x381598, _0x832dc7) {
        null != _0x3c1c1e && ("number" == typeof _0x3c1c1e ? this["fromNumber"](_0x3c1c1e, _0x381598, _0x832dc7) : this["fromString"](_0x3c1c1e, null == _0x381598 && "string" != typeof _0x3c1c1e ? 256 : _0x381598));
      }
      return _0x46cc13["prototype"]["toString"] = function (_0x10c0f4) {
        if (this["s"] < 0) return "-" + this["negate"]()["toString"](_0x10c0f4);
        var _0x2fc7ac;
        if (16 == _0x10c0f4) _0x2fc7ac = 4;else if (8 == _0x10c0f4) _0x2fc7ac = 3;else if (2 == _0x10c0f4) _0x2fc7ac = 1;else if (32 == _0x10c0f4) _0x2fc7ac = 5;else {
          if (4 != _0x10c0f4) return this["toRadix"](_0x10c0f4);
          _0x2fc7ac = 2;
        }
        var _0x8d32c1,
          _0x5d5f39 = (1 << _0x2fc7ac) - 1,
          _0x39cdad = !1,
          _0x4d768c = "",
          _0x4073a9 = this["t"],
          _0x5e3992 = this["DB"] - _0x4073a9 * this["DB"] % _0x2fc7ac;
        if (0 < _0x4073a9--) for (_0x5e3992 < this["DB"] && 0 < (_0x8d32c1 = this[_0x4073a9] >> _0x5e3992) && (_0x39cdad = !0, _0x4d768c = _0x4e31bb(_0x8d32c1)); 0 <= _0x4073a9;) {
          if (_0x5e3992 < _0x2fc7ac) {
            _0x8d32c1 = (this[_0x4073a9] & (1 << _0x5e3992) - 1) << _0x2fc7ac - _0x5e3992, _0x8d32c1 |= this[--_0x4073a9] >> (_0x5e3992 += this["DB"] - _0x2fc7ac);
          } else {
            _0x8d32c1 = this[_0x4073a9] >> (_0x5e3992 -= _0x2fc7ac) & _0x5d5f39, _0x5e3992 <= 0 && (_0x5e3992 += this["DB"], --_0x4073a9);
          }
          0 < _0x8d32c1 && (_0x39cdad = !0);
          _0x39cdad && (_0x4d768c += _0x4e31bb(_0x8d32c1));
        }
        return _0x39cdad ? _0x4d768c : "0";
      }, _0x46cc13["prototype"]["negate"] = function () {
        var _0x5ae427 = _0x425021();
        return _0x46cc13["ZERO"]["subTo"](this, _0x5ae427), _0x5ae427;
      }, _0x46cc13["prototype"]["abs"] = function () {
        return this["s"] < 0 ? this["negate"]() : this;
      }, _0x46cc13["prototype"]["compareTo"] = function (_0x269c23) {
        var _0x46545c = this["s"] - _0x269c23["s"];
        if (0 != _0x46545c) return _0x46545c;
        var _0xa6017d = this["t"];
        if (0 != (_0x46545c = _0xa6017d - _0x269c23["t"])) return this["s"] < 0 ? -_0x46545c : _0x46545c;
        for (; 0 <= --_0xa6017d;) if (0 != (_0x46545c = this[_0xa6017d] - _0x269c23[_0xa6017d])) return _0x46545c;
        return 0;
      }, _0x46cc13["prototype"]["bitLength"] = function () {
        return this["t"] <= 0 ? 0 : this["DB"] * (this["t"] - 1) + _0x312495(this[this["t"] - 1] ^ this["s"] & this["DM"]);
      }, _0x46cc13["prototype"]["mod"] = function (_0x4c88c2) {
        var _0x26f4a3 = _0x425021();
        global = "";
        return this["abs"]()["divRemTo"](_0x4c88c2, null, _0x26f4a3), this["s"] < 0 && 0 < _0x26f4a3["compareTo"](_0x46cc13["ZERO"]) && _0x4c88c2["subTo"](_0x26f4a3, _0x26f4a3), _0x26f4a3;
      }, _0x46cc13["prototype"]["modPowInt"] = function (_0x22576f, _0x14fc0d) {
        var _0xc38aa2;
        return _0x22576f < 256 || _0x14fc0d["isEven"]() ? _0xc38aa2 = new _0x3d96c6(_0x14fc0d) : _0xc38aa2 = new _0x4cb267(_0x14fc0d), this["exp"](_0x22576f, _0xc38aa2);
      }, _0x46cc13["prototype"]["clone"] = function () {
        var _0x46cc13 = _0x425021();
        return this["copyTo"](_0x46cc13), _0x46cc13;
      }, _0x46cc13["prototype"]["intValue"] = function () {
        if (this["s"] < 0) {
          if (1 == this["t"]) return this[0] - this["DV"];
          if (0 == this["t"]) return -1;
        } else {
          if (1 == this["t"]) return this[0];
          if (0 == this["t"]) return 0;
        }
        return (this[1] & (1 << 32 - this["DB"]) - 1) << this["DB"] | this[0];
      }, _0x46cc13["prototype"]["byteValue"] = function () {
        return 0 == this["t"] ? this["s"] : this[0] << 24 >> 24;
      }, _0x46cc13["prototype"]["shortValue"] = function () {
        return 0 == this["t"] ? this["s"] : this[0] << 16 >> 16;
      }, _0x46cc13["prototype"]["signum"] = function () {
        return this["s"] < 0 ? -1 : this["t"] <= 0 || 1 == this["t"] && this[0] <= 0 ? 0 : 1;
      }, _0x46cc13["prototype"]["toByteArray"] = function () {
        var _0x46cc13 = this["t"],
          _0x342052 = [];
        _0x342052[0] = this["s"];
        var _0x1304e5,
          _0x3d47a8 = this["DB"] - _0x46cc13 * this["DB"] % 8,
          _0x58ab94 = 0;
        if (0 < _0x46cc13--) for (_0x3d47a8 < this["DB"] && (_0x1304e5 = this[_0x46cc13] >> _0x3d47a8) != (this["s"] & this["DM"]) >> _0x3d47a8 && (_0x342052[_0x58ab94++] = _0x1304e5 | this["s"] << this["DB"] - _0x3d47a8); 0 <= _0x46cc13;) {
          if (_0x3d47a8 < 8) {
            _0x1304e5 = (this[_0x46cc13] & (1 << _0x3d47a8) - 1) << 8 - _0x3d47a8, _0x1304e5 |= this[--_0x46cc13] >> (_0x3d47a8 += this["DB"] - 8);
          } else {
            _0x1304e5 = this[_0x46cc13] >> (_0x3d47a8 -= 8) & 255, _0x3d47a8 <= 0 && (_0x3d47a8 += this["DB"], --_0x46cc13);
          }
          0 != (128 & _0x1304e5) && (_0x1304e5 |= -256);
          0 == _0x58ab94 && (128 & this["s"]) != (128 & _0x1304e5) && ++_0x58ab94;
          (0 < _0x58ab94 || _0x1304e5 != this["s"]) && (_0x342052[_0x58ab94++] = _0x1304e5);
        }
        return _0x342052;
      }, _0x46cc13["prototype"]["equals"] = function (_0x1b1c5e) {
        return 0 == this["compareTo"](_0x1b1c5e);
      }, _0x46cc13["prototype"]["min"] = function (_0x4aada4) {
        return this["compareTo"](_0x4aada4) < 0 ? this : _0x4aada4;
      }, _0x46cc13["prototype"]["max"] = function (_0xf2be4f) {
        return 0 < this["compareTo"](_0xf2be4f) ? this : _0xf2be4f;
      }, _0x46cc13["prototype"]["and"] = function (_0x1c23d9) {
        var _0x3109cd = _0x425021();
        return this["bitwiseTo"](_0x1c23d9, _0x4273b2, _0x3109cd), _0x3109cd;
      }, _0x46cc13["prototype"]["or"] = function (_0x591665) {
        var _0x51740a = _0x425021();
        return this["bitwiseTo"](_0x591665, _0xfb232b, _0x51740a), _0x51740a;
      }, _0x46cc13["prototype"]["xor"] = function (_0x317049) {
        var _0x2cf3b9 = _0x425021();
        return this["bitwiseTo"](_0x317049, _0x3eba95, _0x2cf3b9), _0x2cf3b9;
      }, _0x46cc13["prototype"]["andNot"] = function (_0x1cfd77) {
        var _0x314e07 = _0x425021();
        return this["bitwiseTo"](_0x1cfd77, _0x1e8fa0, _0x314e07), _0x314e07;
      }, _0x46cc13["prototype"]["not"] = function () {
        for (var _0x46cc13 = _0x425021(), _0x5605c0 = 0; _0x5605c0 < this["t"]; ++_0x5605c0) _0x46cc13[_0x5605c0] = this["DM"] & ~this[_0x5605c0];
        return _0x46cc13["t"] = this["t"], _0x46cc13["s"] = ~this["s"], _0x46cc13;
      }, _0x46cc13["prototype"]["shiftLeft"] = function (_0x2a3de4) {
        var _0x4278c1 = _0x425021();
        return _0x2a3de4 < 0 ? this["rShiftTo"](-_0x2a3de4, _0x4278c1) : this["lShiftTo"](_0x2a3de4, _0x4278c1), _0x4278c1;
      }, _0x46cc13["prototype"]["shiftRight"] = function (_0x38e950) {
        var _0x4a2a69 = _0x425021();
        return _0x38e950 < 0 ? this["lShiftTo"](-_0x38e950, _0x4a2a69) : this["rShiftTo"](_0x38e950, _0x4a2a69), _0x4a2a69;
      }, _0x46cc13["prototype"]["getLowestSetBit"] = function () {
        for (var _0x46cc13 = 0; _0x46cc13 < this["t"]; ++_0x46cc13) if (0 != this[_0x46cc13]) return _0x46cc13 * this["DB"] + _0x525b4a(this[_0x46cc13]);
        return this["s"] < 0 ? this["t"] * this["DB"] : -1;
      }, _0x46cc13["prototype"]["bitCount"] = function () {
        for (var _0x46cc13 = 0, _0x2a78c4 = this["s"] & this["DM"], _0x35f563 = 0; _0x35f563 < this["t"]; ++_0x35f563) _0x46cc13 += _0xbc2d31(this[_0x35f563] ^ _0x2a78c4);
        return _0x46cc13;
      }, _0x46cc13["prototype"]["testBit"] = function (_0x383e28) {
        var _0x49e0fb = Math["floor"](_0x383e28 / this["DB"]);
        return _0x49e0fb >= this["t"] ? 0 != this["s"] : 0 != (this[_0x49e0fb] & 1 << _0x383e28 % this["DB"]);
      }, _0x46cc13["prototype"]["setBit"] = function (_0x4a7b8f) {
        return this["changeBit"](_0x4a7b8f, _0xfb232b);
      }, _0x46cc13["prototype"]["clearBit"] = function (_0x24bb4b) {
        return this["changeBit"](_0x24bb4b, _0x1e8fa0);
      }, _0x46cc13["prototype"]["flipBit"] = function (_0x5a7e36) {
        return this["changeBit"](_0x5a7e36, _0x3eba95);
      }, _0x46cc13["prototype"]["add"] = function (_0x14f082) {
        var _0x3d6724 = _0x425021();
        return this["addTo"](_0x14f082, _0x3d6724), _0x3d6724;
      }, _0x46cc13["prototype"]["subtract"] = function (_0x3965b2) {
        var _0x1977d7 = _0x425021();
        return this["subTo"](_0x3965b2, _0x1977d7), _0x1977d7;
      }, _0x46cc13["prototype"]["multiply"] = function (_0x56db7f) {
        var _0x4fbf4a = _0x425021();
        return this["multiplyTo"](_0x56db7f, _0x4fbf4a), _0x4fbf4a;
      }, _0x46cc13["prototype"]["divide"] = function (_0x29845c) {
        var _0x3b7914 = _0x425021();
        return this["divRemTo"](_0x29845c, _0x3b7914, null), _0x3b7914;
      }, _0x46cc13["prototype"]["remainder"] = function (_0x31a121) {
        var _0x532485 = _0x425021();
        return this["divRemTo"](_0x31a121, null, _0x532485), _0x532485;
      }, _0x46cc13["prototype"]["divideAndRemainder"] = function (_0x2c81d5) {
        var _0x21ad99 = _0x425021(),
          _0x5b6b76 = _0x425021();
        return this["divRemTo"](_0x2c81d5, _0x21ad99, _0x5b6b76), [_0x21ad99, _0x5b6b76];
      }, _0x46cc13["prototype"]["modPow"] = function (_0x4b0895, _0x4004b5) {
        var _0x5966fd,
          _0x426897,
          _0x3b52dc = _0x4b0895["bitLength"](),
          _0x172a2d = _0x46df6e(1);
        if (_0x3b52dc <= 0) return _0x172a2d;
        if (_0x3b52dc < 18) {
          _0x5966fd = 1;
        } else {
          if (_0x3b52dc < 48) {
            _0x5966fd = 3;
          } else {
            if (_0x3b52dc < 144) {
              _0x5966fd = 4;
            } else {
              if (_0x3b52dc < 768) {
                _0x5966fd = 5;
              } else {
                _0x5966fd = 6;
              }
            }
          }
        }
        if (_0x3b52dc < 8) {
          _0x426897 = new _0x3d96c6(_0x4004b5);
        } else {
          if (_0x4004b5["isEven"]()) {
            _0x426897 = new _0x51d9ef(_0x4004b5);
          } else {
            _0x426897 = new _0x4cb267(_0x4004b5);
          }
        }
        var _0x55a302 = [],
          _0x4e6e4e = 3,
          _0x492603 = _0x5966fd - 1,
          _0x1aa350 = (1 << _0x5966fd) - 1;
        if (_0x55a302[1] = _0x426897["convert"](this), 1 < _0x5966fd) {
          var _0x42ae20 = _0x425021();
          for (_0x426897["sqrTo"](_0x55a302[1], _0x42ae20); _0x4e6e4e <= _0x1aa350;) {
            _0x55a302[_0x4e6e4e] = _0x425021();
            _0x426897["mulTo"](_0x42ae20, _0x55a302[_0x4e6e4e - 2], _0x55a302[_0x4e6e4e]);
            _0x4e6e4e += 2;
          }
        }
        var _0x3f5c04,
          _0x2c46b3,
          _0x21d1c7 = _0x4b0895["t"] - 1,
          _0x3fd0a7 = !0,
          _0x57d3b0 = _0x425021();
        for (_0x3b52dc = _0x312495(_0x4b0895[_0x21d1c7]) - 1; 0 <= _0x21d1c7;) {
          for (_0x492603 <= _0x3b52dc ? _0x3f5c04 = _0x4b0895[_0x21d1c7] >> _0x3b52dc - _0x492603 & _0x1aa350 : (_0x3f5c04 = (_0x4b0895[_0x21d1c7] & (1 << _0x3b52dc + 1) - 1) << _0x492603 - _0x3b52dc, 0 < _0x21d1c7 && (_0x3f5c04 |= _0x4b0895[_0x21d1c7 - 1] >> this["DB"] + _0x3b52dc - _0x492603)), _0x4e6e4e = _0x5966fd; 0 == (1 & _0x3f5c04);) {
            _0x3f5c04 >>= 1;
            --_0x4e6e4e;
          }
          if ((_0x3b52dc -= _0x4e6e4e) < 0 && (_0x3b52dc += this["DB"], --_0x21d1c7), _0x3fd0a7) {
            _0x55a302[_0x3f5c04]["copyTo"](_0x172a2d);
            _0x3fd0a7 = !1;
          } else {
            for (; 1 < _0x4e6e4e;) {
              _0x426897["sqrTo"](_0x172a2d, _0x57d3b0);
              _0x426897["sqrTo"](_0x57d3b0, _0x172a2d);
              _0x4e6e4e -= 2;
            }
            if (0 < _0x4e6e4e) {
              _0x426897["sqrTo"](_0x172a2d, _0x57d3b0);
            } else {
              _0x2c46b3 = _0x172a2d, _0x172a2d = _0x57d3b0, _0x57d3b0 = _0x2c46b3;
            }
            _0x426897["mulTo"](_0x57d3b0, _0x55a302[_0x3f5c04], _0x172a2d);
          }
          for (; 0 <= _0x21d1c7 && 0 == (_0x4b0895[_0x21d1c7] & 1 << _0x3b52dc);) {
            _0x426897["sqrTo"](_0x172a2d, _0x57d3b0);
            _0x2c46b3 = _0x172a2d;
            _0x172a2d = _0x57d3b0;
            _0x57d3b0 = _0x2c46b3;
            --_0x3b52dc < 0 && (_0x3b52dc = this["DB"] - 1, --_0x21d1c7);
          }
        }
        return _0x426897["revert"](_0x172a2d);
      }, _0x46cc13["prototype"]["modInverse"] = function (_0x4090e0) {
        var _0x2a8c7f = _0x4090e0["isEven"]();
        if (this["isEven"]() && _0x2a8c7f || 0 == _0x4090e0["signum"]()) return _0x46cc13["ZERO"];
        for (var _0x5d781d = _0x4090e0["clone"](), _0x128b08 = this["clone"](), _0x3f6520 = _0x46df6e(1), _0x4cefba = _0x46df6e(0), _0x46a6be = _0x46df6e(0), _0x5a1244 = _0x46df6e(1); 0 != _0x5d781d["signum"]();) {
          for (; _0x5d781d["isEven"]();) {
            _0x5d781d["rShiftTo"](1, _0x5d781d);
            if (_0x2a8c7f) {
              _0x3f6520["isEven"]() && _0x4cefba["isEven"]() || (_0x3f6520["addTo"](this, _0x3f6520), _0x4cefba["subTo"](_0x4090e0, _0x4cefba)), _0x3f6520["rShiftTo"](1, _0x3f6520);
            } else {
              _0x4cefba["isEven"]() || _0x4cefba["subTo"](_0x4090e0, _0x4cefba);
            }
            _0x4cefba["rShiftTo"](1, _0x4cefba);
          }
          for (; _0x128b08["isEven"]();) {
            _0x128b08["rShiftTo"](1, _0x128b08);
            if (_0x2a8c7f) {
              _0x46a6be["isEven"]() && _0x5a1244["isEven"]() || (_0x46a6be["addTo"](this, _0x46a6be), _0x5a1244["subTo"](_0x4090e0, _0x5a1244)), _0x46a6be["rShiftTo"](1, _0x46a6be);
            } else {
              _0x5a1244["isEven"]() || _0x5a1244["subTo"](_0x4090e0, _0x5a1244);
            }
            _0x5a1244["rShiftTo"](1, _0x5a1244);
          }
          if (0 <= _0x5d781d["compareTo"](_0x128b08)) {
            _0x5d781d["subTo"](_0x128b08, _0x5d781d), _0x2a8c7f && _0x3f6520["subTo"](_0x46a6be, _0x3f6520), _0x4cefba["subTo"](_0x5a1244, _0x4cefba);
          } else {
            _0x128b08["subTo"](_0x5d781d, _0x128b08), _0x2a8c7f && _0x46a6be["subTo"](_0x3f6520, _0x46a6be), _0x5a1244["subTo"](_0x4cefba, _0x5a1244);
          }
        }
        return 0 != _0x128b08["compareTo"](_0x46cc13["ONE"]) ? _0x46cc13["ZERO"] : 0 <= _0x5a1244["compareTo"](_0x4090e0) ? _0x5a1244["subtract"](_0x4090e0) : _0x5a1244["signum"]() < 0 ? (_0x5a1244["addTo"](_0x4090e0, _0x5a1244), _0x5a1244["signum"]() < 0 ? _0x5a1244["add"](_0x4090e0) : _0x5a1244) : _0x5a1244;
      }, _0x46cc13["prototype"]["pow"] = function (_0x5971fc) {
        return this["exp"](_0x5971fc, new _0x24d53a());
      }, _0x46cc13["prototype"]["gcd"] = function (_0x3b294d) {
        var _0x3228f0 = this["s"] < 0 ? this["negate"]() : this["clone"](),
          _0x4d6175 = _0x3b294d["s"] < 0 ? _0x3b294d["negate"]() : _0x3b294d["clone"]();
        if (_0x3228f0["compareTo"](_0x4d6175) < 0) {
          var _0x36d4a0 = _0x3228f0;
          _0x3228f0 = _0x4d6175;
          _0x4d6175 = _0x36d4a0;
        }
        var _0x1836f7 = _0x3228f0["getLowestSetBit"](),
          _0x23d7c4 = _0x4d6175["getLowestSetBit"]();
        if (_0x23d7c4 < 0) return _0x3228f0;
        for (_0x1836f7 < _0x23d7c4 && (_0x23d7c4 = _0x1836f7), 0 < _0x23d7c4 && (_0x3228f0["rShiftTo"](_0x23d7c4, _0x3228f0), _0x4d6175["rShiftTo"](_0x23d7c4, _0x4d6175)); 0 < _0x3228f0["signum"]();) {
          0 < (_0x1836f7 = _0x3228f0["getLowestSetBit"]()) && _0x3228f0["rShiftTo"](_0x1836f7, _0x3228f0);
          0 < (_0x1836f7 = _0x4d6175["getLowestSetBit"]()) && _0x4d6175["rShiftTo"](_0x1836f7, _0x4d6175);
          if (0 <= _0x3228f0["compareTo"](_0x4d6175)) {
            _0x3228f0["subTo"](_0x4d6175, _0x3228f0), _0x3228f0["rShiftTo"](1, _0x3228f0);
          } else {
            _0x4d6175["subTo"](_0x3228f0, _0x4d6175), _0x4d6175["rShiftTo"](1, _0x4d6175);
          }
        }
        return 0 < _0x23d7c4 && _0x4d6175["lShiftTo"](_0x23d7c4, _0x4d6175), _0x4d6175;
      }, _0x46cc13["prototype"]["isProbablePrime"] = function (_0x5c47f2) {
        var _0x2de4ba,
          _0x55b63f = this["abs"]();
        if (1 == _0x55b63f["t"] && _0x55b63f[0] <= _0x16c700[_0x16c700["length"] - 1]) {
          for (_0x2de4ba = 0; _0x2de4ba < _0x16c700["length"]; ++_0x2de4ba) if (_0x55b63f[0] == _0x16c700[_0x2de4ba]) return !0;
          return !1;
        }
        if (_0x55b63f["isEven"]()) return !1;
        for (_0x2de4ba = 1; _0x2de4ba < _0x16c700["length"];) {
          for (var _0x4c925d = _0x16c700[_0x2de4ba], _0x1a3d2e = _0x2de4ba + 1; _0x1a3d2e < _0x16c700["length"] && _0x4c925d < _0x1777d0;) _0x4c925d *= _0x16c700[_0x1a3d2e++];
          for (_0x4c925d = _0x55b63f["modInt"](_0x4c925d); _0x2de4ba < _0x1a3d2e;) if (_0x4c925d % _0x16c700[_0x2de4ba++] == 0) return !1;
        }
        return _0x55b63f["millerRabin"](_0x5c47f2);
      }, _0x46cc13["prototype"]["copyTo"] = function (_0x2cbf8b) {
        for (var _0x11be19 = this["t"] - 1; 0 <= _0x11be19; --_0x11be19) _0x2cbf8b[_0x11be19] = this[_0x11be19];
        _0x2cbf8b["t"] = this["t"];
        _0x2cbf8b["s"] = this["s"];
      }, _0x46cc13["prototype"]["fromInt"] = function (_0x40577b) {
        this["t"] = 1;
        if (_0x40577b < 0) {
          this["s"] = -1;
        } else {
          this["s"] = 0;
        }
        if (0 < _0x40577b) {
          this[0] = _0x40577b;
        } else {
          if (_0x40577b < -1) {
            this[0] = _0x40577b + this["DV"];
          } else {
            this["t"] = 0;
          }
        }
      }, _0x46cc13["prototype"]["fromString"] = function (_0x46fefe, _0x46bd09) {
        var _0x423836;
        if (16 == _0x46bd09) _0x423836 = 4;else if (8 == _0x46bd09) _0x423836 = 3;else if (256 == _0x46bd09) _0x423836 = 8;else if (2 == _0x46bd09) _0x423836 = 1;else if (32 == _0x46bd09) _0x423836 = 5;else {
          if (4 != _0x46bd09) return void this["fromRadix"](_0x46fefe, _0x46bd09);
          _0x423836 = 2;
        }
        this["t"] = 0;
        this["s"] = 0;
        for (var _0x2203c8 = _0x46fefe["length"], _0x397288 = !1, _0x258379 = 0; 0 <= --_0x2203c8;) {
          var _0xa341c7 = 8 == _0x423836 ? 255 & +_0x46fefe[_0x2203c8] : _0x1ea6d1(_0x46fefe, _0x2203c8);
          if (_0xa341c7 < 0) {
            "-" == _0x46fefe["charAt"](_0x2203c8) && (_0x397288 = !0);
          } else {
            _0x397288 = !1, 0 == _0x258379 ? this[this["t"]++] = _0xa341c7 : _0x258379 + _0x423836 > this["DB"] ? (this[this["t"] - 1] |= (_0xa341c7 & (1 << this["DB"] - _0x258379) - 1) << _0x258379, this[this["t"]++] = _0xa341c7 >> this["DB"] - _0x258379) : this[this["t"] - 1] |= _0xa341c7 << _0x258379, (_0x258379 += _0x423836) >= this["DB"] && (_0x258379 -= this["DB"]);
          }
        }
        8 == _0x423836 && 0 != (128 & +_0x46fefe[0]) && (this["s"] = -1, 0 < _0x258379 && (this[this["t"] - 1] |= (1 << this["DB"] - _0x258379) - 1 << _0x258379));
        this["clamp"]();
        _0x397288 && _0x46cc13["ZERO"]["subTo"](this, this);
      }, _0x46cc13["prototype"]["clamp"] = function () {
        for (var _0x46cc13 = this["s"] & this["DM"]; 0 < this["t"] && this[this["t"] - 1] == _0x46cc13;) --this["t"];
      }, _0x46cc13["prototype"]["dlShiftTo"] = function (_0x170a15, _0x16346a) {
        var _0x2078d2;
        for (_0x2078d2 = this["t"] - 1; 0 <= _0x2078d2; --_0x2078d2) _0x16346a[_0x2078d2 + _0x170a15] = this[_0x2078d2];
        for (_0x2078d2 = _0x170a15 - 1; 0 <= _0x2078d2; --_0x2078d2) _0x16346a[_0x2078d2] = 0;
        _0x16346a["t"] = this["t"] + _0x170a15;
        _0x16346a["s"] = this["s"];
      }, _0x46cc13["prototype"]["drShiftTo"] = function (_0x300a63, _0x6b0894) {
        for (var _0x3a54fe = _0x300a63; _0x3a54fe < this["t"]; ++_0x3a54fe) _0x6b0894[_0x3a54fe - _0x300a63] = this[_0x3a54fe];
        _0x6b0894["t"] = Math["max"](this["t"] - _0x300a63, 0);
        _0x6b0894["s"] = this["s"];
      }, _0x46cc13["prototype"]["lShiftTo"] = function (_0x273a7f, _0x593c18) {
        for (var _0x31a918 = _0x273a7f % this["DB"], _0x4f686b = this["DB"] - _0x31a918, _0xd8e230 = (1 << _0x4f686b) - 1, _0x51bb49 = Math["floor"](_0x273a7f / this["DB"]), _0x5b7ef3 = this["s"] << _0x31a918 & this["DM"], _0x1c599f = this["t"] - 1; 0 <= _0x1c599f; --_0x1c599f) {
          _0x593c18[_0x1c599f + _0x51bb49 + 1] = this[_0x1c599f] >> _0x4f686b | _0x5b7ef3;
          _0x5b7ef3 = (this[_0x1c599f] & _0xd8e230) << _0x31a918;
        }
        for (_0x1c599f = _0x51bb49 - 1; 0 <= _0x1c599f; --_0x1c599f) _0x593c18[_0x1c599f] = 0;
        _0x593c18[_0x51bb49] = _0x5b7ef3;
        _0x593c18["t"] = this["t"] + _0x51bb49 + 1;
        _0x593c18["s"] = this["s"];
        _0x593c18["clamp"]();
      }, _0x46cc13["prototype"]["rShiftTo"] = function (_0x3a96f4, _0xd83570) {
        _0xd83570["s"] = this["s"];
        var _0x120f99 = Math["floor"](_0x3a96f4 / this["DB"]);
        if (_0x120f99 >= this["t"]) _0xd83570["t"] = 0;else {
          var _0x332ab2 = _0x3a96f4 % this["DB"],
            _0x486cfd = this["DB"] - _0x332ab2,
            _0x57b419 = (1 << _0x332ab2) - 1;
          _0xd83570[0] = this[_0x120f99] >> _0x332ab2;
          for (var _0x1d86b8 = _0x120f99 + 1; _0x1d86b8 < this["t"]; ++_0x1d86b8) {
            _0xd83570[_0x1d86b8 - _0x120f99 - 1] |= (this[_0x1d86b8] & _0x57b419) << _0x486cfd;
            _0xd83570[_0x1d86b8 - _0x120f99] = this[_0x1d86b8] >> _0x332ab2;
          }
          0 < _0x332ab2 && (_0xd83570[this["t"] - _0x120f99 - 1] |= (this["s"] & _0x57b419) << _0x486cfd);
          _0xd83570["t"] = this["t"] - _0x120f99;
          _0xd83570["clamp"]();
        }
      }, _0x46cc13["prototype"]["subTo"] = function (_0x1bfb46, _0x549664) {
        for (var _0x16e8f1 = 0, _0x377986 = 0, _0x32eb74 = Math["min"](_0x1bfb46["t"], this["t"]); _0x16e8f1 < _0x32eb74;) {
          _0x377986 += this[_0x16e8f1] - _0x1bfb46[_0x16e8f1];
          _0x549664[_0x16e8f1++] = _0x377986 & this["DM"];
          _0x377986 >>= this["DB"];
        }
        if (_0x1bfb46["t"] < this["t"]) {
          for (_0x377986 -= _0x1bfb46["s"]; _0x16e8f1 < this["t"];) {
            _0x377986 += this[_0x16e8f1];
            _0x549664[_0x16e8f1++] = _0x377986 & this["DM"];
            _0x377986 >>= this["DB"];
          }
          _0x377986 += this["s"];
        } else {
          for (_0x377986 += this["s"]; _0x16e8f1 < _0x1bfb46["t"];) {
            _0x377986 -= _0x1bfb46[_0x16e8f1];
            _0x549664[_0x16e8f1++] = _0x377986 & this["DM"];
            _0x377986 >>= this["DB"];
          }
          _0x377986 -= _0x1bfb46["s"];
        }
        if (_0x377986 < 0) {
          _0x549664["s"] = -1;
        } else {
          _0x549664["s"] = 0;
        }
        if (_0x377986 < -1) {
          _0x549664[_0x16e8f1++] = this["DV"] + _0x377986;
        } else {
          0 < _0x377986 && (_0x549664[_0x16e8f1++] = _0x377986);
        }
        _0x549664["t"] = _0x16e8f1;
        _0x549664["clamp"]();
      }, _0x46cc13["prototype"]["multiplyTo"] = function (_0x16123a, _0x362a4d) {
        var _0x5b71c7 = this["abs"](),
          _0x2ef1c3 = _0x16123a["abs"](),
          _0x5da087 = _0x5b71c7["t"];
        for (_0x362a4d["t"] = _0x5da087 + _0x2ef1c3["t"]; 0 <= --_0x5da087;) _0x362a4d[_0x5da087] = 0;
        for (_0x5da087 = 0; _0x5da087 < _0x2ef1c3["t"]; ++_0x5da087) _0x362a4d[_0x5da087 + _0x5b71c7["t"]] = _0x5b71c7["am"](0, _0x2ef1c3[_0x5da087], _0x362a4d, _0x5da087, 0, _0x5b71c7["t"]);
        _0x362a4d["s"] = 0;
        _0x362a4d["clamp"]();
        this["s"] != _0x16123a["s"] && _0x46cc13["ZERO"]["subTo"](_0x362a4d, _0x362a4d);
      }, _0x46cc13["prototype"]["squareTo"] = function (_0x5a401f) {
        for (var _0x3d477b = this["abs"](), _0x23fbd0 = _0x5a401f["t"] = 2 * _0x3d477b["t"]; 0 <= --_0x23fbd0;) _0x5a401f[_0x23fbd0] = 0;
        for (_0x23fbd0 = 0; _0x23fbd0 < _0x3d477b["t"] - 1; ++_0x23fbd0) {
          var _0x57617a = _0x3d477b["am"](_0x23fbd0, _0x3d477b[_0x23fbd0], _0x5a401f, 2 * _0x23fbd0, 0, 1);
          (_0x5a401f[_0x23fbd0 + _0x3d477b["t"]] += _0x3d477b["am"](_0x23fbd0 + 1, 2 * _0x3d477b[_0x23fbd0], _0x5a401f, 2 * _0x23fbd0 + 1, _0x57617a, _0x3d477b["t"] - _0x23fbd0 - 1)) >= _0x3d477b["DV"] && (_0x5a401f[_0x23fbd0 + _0x3d477b["t"]] -= _0x3d477b["DV"], _0x5a401f[_0x23fbd0 + _0x3d477b["t"] + 1] = 1);
        }
        0 < _0x5a401f["t"] && (_0x5a401f[_0x5a401f["t"] - 1] += _0x3d477b["am"](_0x23fbd0, _0x3d477b[_0x23fbd0], _0x5a401f, 2 * _0x23fbd0, 0, 1));
        _0x5a401f["s"] = 0;
        _0x5a401f["clamp"]();
      }, _0x46cc13["prototype"]["divRemTo"] = function (_0x41ab8c, _0x2824dd, _0x4d1123) {
        var _0x7304cf = _0x41ab8c["abs"]();
        if (!(_0x7304cf["t"] <= 0)) {
          var _0x439dcf = this["abs"]();
          if (_0x439dcf["t"] < _0x7304cf["t"]) return null != _0x2824dd && _0x2824dd["fromInt"](0), void (null != _0x4d1123 && this["copyTo"](_0x4d1123));
          null == _0x4d1123 && (_0x4d1123 = _0x425021());
          var _0x56cb54 = _0x425021(),
            _0x2d36e7 = this["s"],
            _0x387fdd = _0x41ab8c["s"],
            _0x13d1f2 = this["DB"] - _0x312495(_0x7304cf[_0x7304cf["t"] - 1]);
          if (0 < _0x13d1f2) {
            _0x7304cf["lShiftTo"](_0x13d1f2, _0x56cb54), _0x439dcf["lShiftTo"](_0x13d1f2, _0x4d1123);
          } else {
            _0x7304cf["copyTo"](_0x56cb54), _0x439dcf["copyTo"](_0x4d1123);
          }
          var _0x391c4a = _0x56cb54["t"],
            _0x543a0b = _0x56cb54[_0x391c4a - 1];
          if (0 != _0x543a0b) {
            var _0x2ae540 = _0x543a0b * (1 << this["F1"]) + (1 < _0x391c4a ? _0x56cb54[_0x391c4a - 2] >> this["F2"] : 0),
              _0x469d6b = this["FV"] / _0x2ae540,
              _0x1bb9ba = (1 << this["F1"]) / _0x2ae540,
              _0x5f4fc5 = 1 << this["F2"],
              _0x56f799 = _0x4d1123["t"],
              _0x1e8018 = _0x56f799 - _0x391c4a,
              _0x4d3acc = null == _0x2824dd ? _0x425021() : _0x2824dd;
            for (_0x56cb54["dlShiftTo"](_0x1e8018, _0x4d3acc), 0 <= _0x4d1123["compareTo"](_0x4d3acc) && (_0x4d1123[_0x4d1123["t"]++] = 1, _0x4d1123["subTo"](_0x4d3acc, _0x4d1123)), _0x46cc13["ONE"]["dlShiftTo"](_0x391c4a, _0x4d3acc), _0x4d3acc["subTo"](_0x56cb54, _0x56cb54); _0x56cb54["t"] < _0x391c4a;) _0x56cb54[_0x56cb54["t"]++] = 0;
            for (; 0 <= --_0x1e8018;) {
              var _0x161488 = _0x4d1123[--_0x56f799] == _0x543a0b ? this["DM"] : Math["floor"](_0x4d1123[_0x56f799] * _0x469d6b + (_0x4d1123[_0x56f799 - 1] + _0x5f4fc5) * _0x1bb9ba);
              if ((_0x4d1123[_0x56f799] += _0x56cb54["am"](0, _0x161488, _0x4d1123, _0x1e8018, 0, _0x391c4a)) < _0x161488) for (_0x56cb54["dlShiftTo"](_0x1e8018, _0x4d3acc), _0x4d1123["subTo"](_0x4d3acc, _0x4d1123); _0x4d1123[_0x56f799] < --_0x161488;) _0x4d1123["subTo"](_0x4d3acc, _0x4d1123);
            }
            null != _0x2824dd && (_0x4d1123["drShiftTo"](_0x391c4a, _0x2824dd), _0x2d36e7 != _0x387fdd && _0x46cc13["ZERO"]["subTo"](_0x2824dd, _0x2824dd));
            _0x4d1123["t"] = _0x391c4a;
            _0x4d1123["clamp"]();
            0 < _0x13d1f2 && _0x4d1123["rShiftTo"](_0x13d1f2, _0x4d1123);
            _0x2d36e7 < 0 && _0x46cc13["ZERO"]["subTo"](_0x4d1123, _0x4d1123);
          }
        }
      }, _0x46cc13["prototype"]["invDigit"] = function () {
        if (this["t"] < 1) return 0;
        var _0x46cc13 = this[0];
        if (0 == (1 & _0x46cc13)) return 0;
        var _0x46a795 = 3 & _0x46cc13;
        return 0 < (_0x46a795 = (_0x46a795 = (_0x46a795 = (_0x46a795 = _0x46a795 * (2 - (15 & _0x46cc13) * _0x46a795) & 15) * (2 - (255 & _0x46cc13) * _0x46a795) & 255) * (2 - ((65535 & _0x46cc13) * _0x46a795 & 65535)) & 65535) * (2 - _0x46cc13 * _0x46a795 % this["DV"]) % this["DV"]) ? this["DV"] - _0x46a795 : -_0x46a795;
      }, _0x46cc13["prototype"]["isEven"] = function () {
        return 0 == (0 < this["t"] ? 1 & this[0] : this["s"]);
      }, _0x46cc13["prototype"]["exp"] = function (_0x4a65ad, _0x212c70) {
        if (4294967295 < _0x4a65ad || _0x4a65ad < 1) return _0x46cc13["ONE"];
        var _0x34ca43 = _0x425021(),
          _0x43b99d = _0x425021(),
          _0x5e3d8c = _0x212c70["convert"](this),
          _0x23c43d = _0x312495(_0x4a65ad) - 1;
        for (_0x5e3d8c["copyTo"](_0x34ca43); 0 <= --_0x23c43d;) if (_0x212c70["sqrTo"](_0x34ca43, _0x43b99d), 0 < (_0x4a65ad & 1 << _0x23c43d)) _0x212c70["mulTo"](_0x43b99d, _0x5e3d8c, _0x34ca43);else {
          var _0x2a54ce = _0x34ca43;
          _0x34ca43 = _0x43b99d;
          _0x43b99d = _0x2a54ce;
        }
        return _0x212c70["revert"](_0x34ca43);
      }, _0x46cc13["prototype"]["chunkSize"] = function (_0x41f3a2) {
        return Math["floor"](Math["LN2"] * this["DB"] / Math["log"](_0x41f3a2));
      }, _0x46cc13["prototype"]["toRadix"] = function (_0x57ab55) {
        if (null == _0x57ab55 && (_0x57ab55 = 10), 0 == this["signum"]() || _0x57ab55 < 2 || 36 < _0x57ab55) return "0";
        var _0x4f1caa = this["chunkSize"](_0x57ab55),
          _0x113f29 = Math["pow"](_0x57ab55, _0x4f1caa),
          _0x3007b3 = _0x46df6e(_0x113f29),
          _0x506024 = _0x425021(),
          _0x5bfbe2 = _0x425021(),
          _0x27f471 = "";
        for (this["divRemTo"](_0x3007b3, _0x506024, _0x5bfbe2); 0 < _0x506024["signum"]();) {
          _0x27f471 = (_0x113f29 + _0x5bfbe2["intValue"]())["toString"](_0x57ab55)["substr"](1) + _0x27f471;
          _0x506024["divRemTo"](_0x3007b3, _0x506024, _0x5bfbe2);
        }
        return _0x5bfbe2["intValue"]()["toString"](_0x57ab55) + _0x27f471;
      }, _0x46cc13["prototype"]["fromRadix"] = function (_0xab2623, _0x3c412d) {
        this["fromInt"](0);
        null == _0x3c412d && (_0x3c412d = 10);
        for (var _0x318346 = this["chunkSize"](_0x3c412d), _0x2a6a20 = Math["pow"](_0x3c412d, _0x318346), _0x10032d = !1, _0x3f84db = 0, _0xf95517 = 0, _0x5bda4b = 0; _0x5bda4b < _0xab2623["length"]; ++_0x5bda4b) {
          var _0x431258 = _0x1ea6d1(_0xab2623, _0x5bda4b);
          if (_0x431258 < 0) {
            "-" == _0xab2623["charAt"](_0x5bda4b) && 0 == this["signum"]() && (_0x10032d = !0);
          } else {
            _0xf95517 = _0x3c412d * _0xf95517 + _0x431258, ++_0x3f84db >= _0x318346 && (this["dMultiply"](_0x2a6a20), this["dAddOffset"](_0xf95517, 0), _0xf95517 = _0x3f84db = 0);
          }
        }
        0 < _0x3f84db && (this["dMultiply"](Math["pow"](_0x3c412d, _0x3f84db)), this["dAddOffset"](_0xf95517, 0));
        _0x10032d && _0x46cc13["ZERO"]["subTo"](this, this);
      }, _0x46cc13["prototype"]["fromNumber"] = function (_0x1b6a1b, _0x2799da, _0x3fea16) {
        if ("number" == typeof _0x2799da) {
          if (_0x1b6a1b < 2) this["fromInt"](1);else for (this["fromNumber"](_0x1b6a1b, _0x3fea16), this["testBit"](_0x1b6a1b - 1) || this["bitwiseTo"](_0x46cc13["ONE"]["shiftLeft"](_0x1b6a1b - 1), _0xfb232b, this), this["isEven"]() && this["dAddOffset"](1, 0); !this["isProbablePrime"](_0x2799da);) {
            this["dAddOffset"](2, 0);
            this["bitLength"]() > _0x1b6a1b && this["subTo"](_0x46cc13["ONE"]["shiftLeft"](_0x1b6a1b - 1), this);
          }
        } else {
          var _0xaccbba = [],
            _0x4bd77e = 7 & _0x1b6a1b;
          _0xaccbba["length"] = 1 + (_0x1b6a1b >> 3);
          _0x2799da["nextBytes"](_0xaccbba);
          if (0 < _0x4bd77e) {
            _0xaccbba[0] &= (1 << _0x4bd77e) - 1;
          } else {
            _0xaccbba[0] = 0;
          }
          this["fromString"](_0xaccbba, 256);
        }
      }, _0x46cc13["prototype"]["bitwiseTo"] = function (_0x5868f5, _0x57e124, _0x5f05b7) {
        var _0x13ecef,
          _0x431f37,
          _0x212347 = Math["min"](_0x5868f5["t"], this["t"]);
        for (_0x13ecef = 0; _0x13ecef < _0x212347; ++_0x13ecef) _0x5f05b7[_0x13ecef] = _0x57e124(this[_0x13ecef], _0x5868f5[_0x13ecef]);
        if (_0x5868f5["t"] < this["t"]) {
          for (_0x431f37 = _0x5868f5["s"] & this["DM"], _0x13ecef = _0x212347; _0x13ecef < this["t"]; ++_0x13ecef) _0x5f05b7[_0x13ecef] = _0x57e124(this[_0x13ecef], _0x431f37);
          _0x5f05b7["t"] = this["t"];
        } else {
          for (_0x431f37 = this["s"] & this["DM"], _0x13ecef = _0x212347; _0x13ecef < _0x5868f5["t"]; ++_0x13ecef) _0x5f05b7[_0x13ecef] = _0x57e124(_0x431f37, _0x5868f5[_0x13ecef]);
          _0x5f05b7["t"] = _0x5868f5["t"];
        }
        _0x5f05b7["s"] = _0x57e124(this["s"], _0x5868f5["s"]);
        _0x5f05b7["clamp"]();
      }, _0x46cc13["prototype"]["changeBit"] = function (_0x387312, _0xc77ee7) {
        var _0x560ddf = _0x46cc13["ONE"]["shiftLeft"](_0x387312);
        return this["bitwiseTo"](_0x560ddf, _0xc77ee7, _0x560ddf), _0x560ddf;
      }, _0x46cc13["prototype"]["addTo"] = function (_0x432801, _0x4850ea) {
        for (var _0x31b1f0 = 0, _0x30d985 = 0, _0x1b15f9 = Math["min"](_0x432801["t"], this["t"]); _0x31b1f0 < _0x1b15f9;) {
          _0x30d985 += this[_0x31b1f0] + _0x432801[_0x31b1f0];
          _0x4850ea[_0x31b1f0++] = _0x30d985 & this["DM"];
          _0x30d985 >>= this["DB"];
        }
        if (_0x432801["t"] < this["t"]) {
          for (_0x30d985 += _0x432801["s"]; _0x31b1f0 < this["t"];) {
            _0x30d985 += this[_0x31b1f0];
            _0x4850ea[_0x31b1f0++] = _0x30d985 & this["DM"];
            _0x30d985 >>= this["DB"];
          }
          _0x30d985 += this["s"];
        } else {
          for (_0x30d985 += this["s"]; _0x31b1f0 < _0x432801["t"];) {
            _0x30d985 += _0x432801[_0x31b1f0];
            _0x4850ea[_0x31b1f0++] = _0x30d985 & this["DM"];
            _0x30d985 >>= this["DB"];
          }
          _0x30d985 += _0x432801["s"];
        }
        if (_0x30d985 < 0) {
          _0x4850ea["s"] = -1;
        } else {
          _0x4850ea["s"] = 0;
        }
        if (0 < _0x30d985) {
          _0x4850ea[_0x31b1f0++] = _0x30d985;
        } else {
          _0x30d985 < -1 && (_0x4850ea[_0x31b1f0++] = this["DV"] + _0x30d985);
        }
        _0x4850ea["t"] = _0x31b1f0;
        _0x4850ea["clamp"]();
      }, _0x46cc13["prototype"]["dMultiply"] = function (_0x2d7d07) {
        this[this["t"]] = this["am"](0, _0x2d7d07 - 1, this, 0, 0, this["t"]);
        ++this["t"];
        this["clamp"]();
      }, _0x46cc13["prototype"]["dAddOffset"] = function (_0x4ef00f, _0x13e7b7) {
        if (0 != _0x4ef00f) {
          for (; this["t"] <= _0x13e7b7;) this[this["t"]++] = 0;
          for (this[_0x13e7b7] += _0x4ef00f; this[_0x13e7b7] >= this["DV"];) {
            this[_0x13e7b7] -= this["DV"];
            ++_0x13e7b7 >= this["t"] && (this[this["t"]++] = 0);
            ++this[_0x13e7b7];
          }
        }
      }, _0x46cc13["prototype"]["multiplyLowerTo"] = function (_0x10db58, _0x3ae37d, _0x4f92b3) {
        var _0x5d8c8a = Math["min"](this["t"] + _0x10db58["t"], _0x3ae37d);
        for (_0x4f92b3["s"] = 0, _0x4f92b3["t"] = _0x5d8c8a; 0 < _0x5d8c8a;) _0x4f92b3[--_0x5d8c8a] = 0;
        for (var _0x5f2c8d = _0x4f92b3["t"] - this["t"]; _0x5d8c8a < _0x5f2c8d; ++_0x5d8c8a) _0x4f92b3[_0x5d8c8a + this["t"]] = this["am"](0, _0x10db58[_0x5d8c8a], _0x4f92b3, _0x5d8c8a, 0, this["t"]);
        for (_0x5f2c8d = Math["min"](_0x10db58["t"], _0x3ae37d); _0x5d8c8a < _0x5f2c8d; ++_0x5d8c8a) this["am"](0, _0x10db58[_0x5d8c8a], _0x4f92b3, _0x5d8c8a, 0, _0x3ae37d - _0x5d8c8a);
        _0x4f92b3["clamp"]();
      }, _0x46cc13["prototype"]["multiplyUpperTo"] = function (_0x160c85, _0x189bce, _0x3a2c9c) {
        var _0x51cb9d = _0x3a2c9c["t"] = this["t"] + _0x160c85["t"] - --_0x189bce;
        for (_0x3a2c9c["s"] = 0; 0 <= --_0x51cb9d;) _0x3a2c9c[_0x51cb9d] = 0;
        for (_0x51cb9d = Math["max"](_0x189bce - this["t"], 0); _0x51cb9d < _0x160c85["t"]; ++_0x51cb9d) _0x3a2c9c[this["t"] + _0x51cb9d - _0x189bce] = this["am"](_0x189bce - _0x51cb9d, _0x160c85[_0x51cb9d], _0x3a2c9c, 0, 0, this["t"] + _0x51cb9d - _0x189bce);
        _0x3a2c9c["clamp"]();
        _0x3a2c9c["drShiftTo"](1, _0x3a2c9c);
      }, _0x46cc13["prototype"]["modInt"] = function (_0x1647ad) {
        if (_0x1647ad <= 0) return 0;
        var _0x453e4c = this["DV"] % _0x1647ad,
          _0x5dacdb = this["s"] < 0 ? _0x1647ad - 1 : 0;
        if (0 < this["t"]) if (0 == _0x453e4c) _0x5dacdb = this[0] % _0x1647ad;else for (var _0x1b9d78 = this["t"] - 1; 0 <= _0x1b9d78; --_0x1b9d78) _0x5dacdb = (_0x453e4c * _0x5dacdb + this[_0x1b9d78]) % _0x1647ad;
        return _0x5dacdb;
      }, _0x46cc13["prototype"]["millerRabin"] = function (_0x5bea7e) {
        var _0x419cf9 = this["subtract"](_0x46cc13["ONE"]),
          _0x4f6306 = _0x419cf9["getLowestSetBit"]();
        if (_0x4f6306 <= 0) return !1;
        var _0x5c625f = _0x419cf9["shiftRight"](_0x4f6306);
        _0x16c700["length"] < (_0x5bea7e = _0x5bea7e + 1 >> 1) && (_0x5bea7e = _0x16c700["length"]);
        for (var _0x3a0263 = _0x425021(), _0x58e43d = 0; _0x58e43d < _0x5bea7e; ++_0x58e43d) {
          var _0x1dfb1a = _0x3a0263["modPow"](_0x5c625f, this);
          if (0 != _0x1dfb1a["compareTo"](_0x46cc13["ONE"]) && 0 != _0x1dfb1a["compareTo"](_0x419cf9)) {
            for (var _0x5a9e49 = 1; _0x5a9e49++ < _0x4f6306 && 0 != _0x1dfb1a["compareTo"](_0x419cf9);) if (0 == (_0x1dfb1a = _0x1dfb1a["modPowInt"](2, this))["compareTo"](_0x46cc13["ONE"])) return !1;
            if (0 != _0x1dfb1a["compareTo"](_0x419cf9)) return !1;
          }
        }
        return !0;
      }, _0x46cc13["prototype"]["square"] = function () {
        var _0x46cc13 = _0x425021();
        return this["squareTo"](_0x46cc13), _0x46cc13;
      }, _0x46cc13["prototype"]["gcda"] = function (_0x5d3e11, _0x208d32) {
        var _0x563b22 = this["s"] < 0 ? this["negate"]() : this["clone"](),
          _0x5b7311 = _0x5d3e11["s"] < 0 ? _0x5d3e11["negate"]() : _0x5d3e11["clone"]();
        if (_0x563b22["compareTo"](_0x5b7311) < 0) {
          var _0x926c66 = _0x563b22;
          _0x563b22 = _0x5b7311;
          _0x5b7311 = _0x926c66;
        }
        var _0xca6de8 = _0x563b22["getLowestSetBit"](),
          _0x1d7bf6 = _0x5b7311["getLowestSetBit"]();
        if (_0x1d7bf6 < 0) _0x208d32(_0x563b22);else {
          _0xca6de8 < _0x1d7bf6 && (_0x1d7bf6 = _0xca6de8);
          0 < _0x1d7bf6 && (_0x563b22["rShiftTo"](_0x1d7bf6, _0x563b22), _0x5b7311["rShiftTo"](_0x1d7bf6, _0x5b7311));
          var _0xa4350a = function () {
            0 < (_0xca6de8 = _0x563b22["getLowestSetBit"]()) && _0x563b22["rShiftTo"](_0xca6de8, _0x563b22);
            0 < (_0xca6de8 = _0x5b7311["getLowestSetBit"]()) && _0x5b7311["rShiftTo"](_0xca6de8, _0x5b7311);
            if (0 <= _0x563b22["compareTo"](_0x5b7311)) {
              _0x563b22["subTo"](_0x5b7311, _0x563b22), _0x563b22["rShiftTo"](1, _0x563b22);
            } else {
              _0x5b7311["subTo"](_0x563b22, _0x5b7311), _0x5b7311["rShiftTo"](1, _0x5b7311);
            }
            if (0 < _0x563b22["signum"]()) {
              setTimeout(_0xa4350a, 0);
            } else {
              0 < _0x1d7bf6 && _0x5b7311["lShiftTo"](_0x1d7bf6, _0x5b7311), setTimeout(function () {
                _0x208d32(_0x5b7311);
              }, 0);
            }
          };
          setTimeout(_0xa4350a, 10);
        }
      }, _0x46cc13["prototype"]["fromNumberAsync"] = function (_0x15b60c, _0x1de11d, _0x39e9ee, _0x5cb406) {
        if ("number" == typeof _0x1de11d) {
          if (_0x15b60c < 2) this["fromInt"](1);else {
            this["fromNumber"](_0x15b60c, _0x39e9ee);
            this["testBit"](_0x15b60c - 1) || this["bitwiseTo"](_0x46cc13["ONE"]["shiftLeft"](_0x15b60c - 1), _0xfb232b, this);
            this["isEven"]() && this["dAddOffset"](1, 0);
            var _0xfc69a0 = this,
              _0x4c3b48 = function () {
                _0xfc69a0["dAddOffset"](2, 0);
                _0xfc69a0["bitLength"]() > _0x15b60c && _0xfc69a0["subTo"](_0x46cc13["ONE"]["shiftLeft"](_0x15b60c - 1), _0xfc69a0);
                if (_0xfc69a0["isProbablePrime"](_0x1de11d)) {
                  setTimeout(function () {
                    _0x5cb406();
                  }, 0);
                } else {
                  setTimeout(_0x4c3b48, 0);
                }
              };
            setTimeout(_0x4c3b48, 0);
          }
        } else {
          var _0x304322 = [],
            _0x5e7edd = 7 & _0x15b60c;
          _0x304322["length"] = 1 + (_0x15b60c >> 3);
          _0x1de11d["nextBytes"](_0x304322);
          if (0 < _0x5e7edd) {
            _0x304322[0] &= (1 << _0x5e7edd) - 1;
          } else {
            _0x304322[0] = 0;
          }
          this["fromString"](_0x304322, 256);
        }
      }, _0x46cc13;
    }(),
    _0x24d53a = function () {
      function _0x7a0cac() {}
      return _0x7a0cac["prototype"]["convert"] = function (_0x41ce77) {
        return _0x41ce77;
      }, _0x7a0cac["prototype"]["revert"] = function (_0x4b1d70) {
        return _0x4b1d70;
      }, _0x7a0cac["prototype"]["mulTo"] = function (_0x50037c, _0x59c94d, _0x11277b) {
        _0x50037c["multiplyTo"](_0x59c94d, _0x11277b);
      }, _0x7a0cac["prototype"]["sqrTo"] = function (_0x42cc41, _0x2ef799) {
        _0x42cc41["squareTo"](_0x2ef799);
      }, _0x7a0cac;
    }(),
    _0x3d96c6 = function () {
      function _0xc1f893(_0x3f080a) {
        this["m"] = _0x3f080a;
      }
      return _0xc1f893["prototype"]["convert"] = function (_0xb45a47) {
        return _0xb45a47["s"] < 0 || 0 <= _0xb45a47["compareTo"](this["m"]) ? _0xb45a47["mod"](this["m"]) : _0xb45a47;
      }, _0xc1f893["prototype"]["revert"] = function (_0x16f707) {
        return _0x16f707;
      }, _0xc1f893["prototype"]["reduce"] = function (_0x319968) {
        _0x319968["divRemTo"](this["m"], null, _0x319968);
      }, _0xc1f893["prototype"]["mulTo"] = function (_0x380f64, _0x1b8469, _0x22a3eb) {
        _0x380f64["multiplyTo"](_0x1b8469, _0x22a3eb);
        this["reduce"](_0x22a3eb);
      }, _0xc1f893["prototype"]["sqrTo"] = function (_0x1827a1, _0x4c3d20) {
        _0x1827a1["squareTo"](_0x4c3d20);
        this["reduce"](_0x4c3d20);
      }, _0xc1f893;
    }(),
    _0x4cb267 = function () {
      function _0xdf43b3(_0x4a9319) {
        this["m"] = _0x4a9319;
        this["mp"] = _0x4a9319["invDigit"]();
        this["mpl"] = 32767 & this["mp"];
        this["mph"] = this["mp"] >> 15;
        this["um"] = (1 << _0x4a9319["DB"] - 15) - 1;
        this["mt2"] = 2 * _0x4a9319["t"];
      }
      return _0xdf43b3["prototype"]["convert"] = function (_0x11c977) {
        var _0x3dc738 = _0x425021();
        return _0x11c977["abs"]()["dlShiftTo"](this["m"]["t"], _0x3dc738), _0x3dc738["divRemTo"](this["m"], null, _0x3dc738), _0x11c977["s"] < 0 && 0 < _0x3dc738["compareTo"](_0x2935af["ZERO"]) && this["m"]["subTo"](_0x3dc738, _0x3dc738), _0x3dc738;
      }, _0xdf43b3["prototype"]["revert"] = function (_0x5202f8) {
        var _0x4263bf = _0x425021();
        return _0x5202f8["copyTo"](_0x4263bf), this["reduce"](_0x4263bf), _0x4263bf;
      }, _0xdf43b3["prototype"]["reduce"] = function (_0x3482a1) {
        for (; _0x3482a1["t"] <= this["mt2"];) _0x3482a1[_0x3482a1["t"]++] = 0;
        for (var _0x204387 = 0; _0x204387 < this["m"]["t"]; ++_0x204387) {
          var _0x303298 = 32767 & _0x3482a1[_0x204387],
            _0x4d9d80 = _0x303298 * this["mpl"] + ((_0x303298 * this["mph"] + (_0x3482a1[_0x204387] >> 15) * this["mpl"] & this["um"]) << 15) & _0x3482a1["DM"];
          for (_0x3482a1[_0x303298 = _0x204387 + this["m"]["t"]] += this["m"]["am"](0, _0x4d9d80, _0x3482a1, _0x204387, 0, this["m"]["t"]); _0x3482a1[_0x303298] >= _0x3482a1["DV"];) {
            _0x3482a1[_0x303298] -= _0x3482a1["DV"];
            _0x3482a1[++_0x303298]++;
          }
        }
        _0x3482a1["clamp"]();
        _0x3482a1["drShiftTo"](this["m"]["t"], _0x3482a1);
        0 <= _0x3482a1["compareTo"](this["m"]) && _0x3482a1["subTo"](this["m"], _0x3482a1);
      }, _0xdf43b3["prototype"]["mulTo"] = function (_0x3ffcce, _0x116ccb, _0x3016b6) {
        _0x3ffcce["multiplyTo"](_0x116ccb, _0x3016b6);
        this["reduce"](_0x3016b6);
      }, _0xdf43b3["prototype"]["sqrTo"] = function (_0x50e4f5, _0x297e38) {
        _0x50e4f5["squareTo"](_0x297e38);
        this["reduce"](_0x297e38);
      }, _0xdf43b3;
    }(),
    _0x51d9ef = function () {
      function _0x100af5(_0x2e4853) {
        this["m"] = _0x2e4853;
        this["r2"] = _0x425021();
        this["q3"] = _0x425021();
        _0x2935af["ONE"]["dlShiftTo"](2 * _0x2e4853["t"], this["r2"]);
        this["mu"] = this["r2"]["divide"](_0x2e4853);
      }
      return _0x100af5["prototype"]["convert"] = function (_0x5e0c07) {
        if (_0x5e0c07["s"] < 0 || _0x5e0c07["t"] > 2 * this["m"]["t"]) return _0x5e0c07["mod"](this["m"]);
        if (_0x5e0c07["compareTo"](this["m"]) < 0) return _0x5e0c07;
        var _0x38c177 = _0x425021();
        return _0x5e0c07["copyTo"](_0x38c177), this["reduce"](_0x38c177), _0x38c177;
      }, _0x100af5["prototype"]["revert"] = function (_0x53035b) {
        return _0x53035b;
      }, _0x100af5["prototype"]["reduce"] = function (_0x5f39d8) {
        for (_0x5f39d8["drShiftTo"](this["m"]["t"] - 1, this["r2"]), _0x5f39d8["t"] > this["m"]["t"] + 1 && (_0x5f39d8["t"] = this["m"]["t"] + 1, _0x5f39d8["clamp"]()), this["mu"]["multiplyUpperTo"](this["r2"], this["m"]["t"] + 1, this["q3"]), this["m"]["multiplyLowerTo"](this["q3"], this["m"]["t"] + 1, this["r2"]); _0x5f39d8["compareTo"](this["r2"]) < 0;) _0x5f39d8["dAddOffset"](1, this["m"]["t"] + 1);
        for (_0x5f39d8["subTo"](this["r2"], _0x5f39d8); 0 <= _0x5f39d8["compareTo"](this["m"]);) _0x5f39d8["subTo"](this["m"], _0x5f39d8);
      }, _0x100af5["prototype"]["mulTo"] = function (_0x720735, _0x5b47e6, _0x41d9c1) {
        _0x720735["multiplyTo"](_0x5b47e6, _0x41d9c1);
        this["reduce"](_0x41d9c1);
      }, _0x100af5["prototype"]["sqrTo"] = function (_0x464b92, _0x42a884) {
        _0x464b92["squareTo"](_0x42a884);
        this["reduce"](_0x42a884);
      }, _0x100af5;
    }();
  function _0x425021() {
    return new _0x2935af(null);
  }
  function _0x5baf06(_0xe04d3b, _0x372989) {
    return new _0x2935af(_0xe04d3b, _0x372989);
  }
  if ("Microsoft Internet Explorer" == navigator["appName"]) {
    _0x2935af["prototype"]["am"] = function (_0x1af29b, _0x16b8da, _0x597a23, _0x401af9, _0x55ee66, _0x516c12) {
      for (var _0x36093e = 32767 & _0x16b8da, _0x259618 = _0x16b8da >> 15; 0 <= --_0x516c12;) {
        var _0x57bb8f = 32767 & this[_0x1af29b],
          _0x5ae27a = this[_0x1af29b++] >> 15,
          _0x387d95 = _0x259618 * _0x57bb8f + _0x5ae27a * _0x36093e;
        _0x55ee66 = ((_0x57bb8f = _0x36093e * _0x57bb8f + ((32767 & _0x387d95) << 15) + _0x597a23[_0x401af9] + (1073741823 & _0x55ee66)) >>> 30) + (_0x387d95 >>> 15) + _0x259618 * _0x5ae27a + (_0x55ee66 >>> 30);
        _0x597a23[_0x401af9++] = 1073741823 & _0x57bb8f;
      }
      return _0x55ee66;
    }, _0x10b29a = 30;
  } else {
    if ("Netscape" != navigator["appName"]) {
      _0x2935af["prototype"]["am"] = function (_0x4c4e6e, _0x5ead7c, _0x337aa0, _0x106255, _0x34e937, _0x2d2a12) {
        for (; 0 <= --_0x2d2a12;) {
          var _0x232504 = _0x5ead7c * this[_0x4c4e6e++] + _0x337aa0[_0x106255] + _0x34e937;
          _0x34e937 = Math["floor"](_0x232504 / 67108864);
          _0x337aa0[_0x106255++] = 67108863 & _0x232504;
        }
        return _0x34e937;
      }, _0x10b29a = 26;
    } else {
      _0x2935af["prototype"]["am"] = function (_0x3c1afb, _0x18bfac, _0x136996, _0x327041, _0x13dbc4, _0x1a1704) {
        for (var _0x5c74f1 = 16383 & _0x18bfac, _0x24929e = _0x18bfac >> 14; 0 <= --_0x1a1704;) {
          var _0x1dec8c = 16383 & this[_0x3c1afb],
            _0x2a0d28 = this[_0x3c1afb++] >> 14,
            _0x380ee7 = _0x24929e * _0x1dec8c + _0x2a0d28 * _0x5c74f1;
          _0x13dbc4 = ((_0x1dec8c = _0x5c74f1 * _0x1dec8c + ((16383 & _0x380ee7) << 14) + _0x136996[_0x327041] + _0x13dbc4) >> 28) + (_0x380ee7 >> 14) + _0x24929e * _0x2a0d28;
          _0x136996[_0x327041++] = 268435455 & _0x1dec8c;
        }
        return _0x13dbc4;
      }, _0x10b29a = 28;
    }
  }
  _0x2935af["prototype"]["DB"] = _0x10b29a;
  _0x2935af["prototype"]["DM"] = (1 << _0x10b29a) - 1;
  _0x2935af["prototype"]["DV"] = 1 << _0x10b29a;
  _0x2935af["prototype"]["FV"] = Math["pow"](2, 52);
  _0x2935af["prototype"]["F1"] = 52 - _0x10b29a;
  _0x2935af["prototype"]["F2"] = 2 * _0x10b29a - 52;
  var _0x563e61,
    _0x1d04ef,
    _0x1c8429 = [];
  for (_0x563e61 = "0"["charCodeAt"](0), _0x1d04ef = 0; _0x1d04ef <= 9; ++_0x1d04ef) _0x1c8429[_0x563e61++] = _0x1d04ef;
  for (_0x563e61 = "a"["charCodeAt"](0), _0x1d04ef = 10; _0x1d04ef < 36; ++_0x1d04ef) _0x1c8429[_0x563e61++] = _0x1d04ef;
  for (_0x563e61 = "A"["charCodeAt"](0), _0x1d04ef = 10; _0x1d04ef < 36; ++_0x1d04ef) _0x1c8429[_0x563e61++] = _0x1d04ef;
  function _0x1ea6d1(_0x1886b1, _0x3dcfad) {
    var _0x9abde0 = _0x1c8429[_0x1886b1["charCodeAt"](_0x3dcfad)];
    return null == _0x9abde0 ? -1 : _0x9abde0;
  }
  function _0x46df6e(_0x48f57e) {
    var _0x8b3fc2 = _0x425021();
    return _0x8b3fc2["fromInt"](_0x48f57e), _0x8b3fc2;
  }
  function _0x312495(_0x30770b) {
    var _0x3e6991,
      _0x4657cc = 1;
    return 0 != (_0x3e6991 = _0x30770b >>> 16) && (_0x30770b = _0x3e6991, _0x4657cc += 16), 0 != (_0x3e6991 = _0x30770b >> 8) && (_0x30770b = _0x3e6991, _0x4657cc += 8), 0 != (_0x3e6991 = _0x30770b >> 4) && (_0x30770b = _0x3e6991, _0x4657cc += 4), 0 != (_0x3e6991 = _0x30770b >> 2) && (_0x30770b = _0x3e6991, _0x4657cc += 2), 0 != (_0x3e6991 = _0x30770b >> 1) && (_0x30770b = _0x3e6991, _0x4657cc += 1), _0x4657cc;
  }
  _0x2935af["ZERO"] = _0x46df6e(0);
  _0x2935af["ONE"] = _0x46df6e(1);
  var _0x53c881,
    _0x198bd8,
    _0x19fde7 = function () {
      function _0x7d14e5() {
        this["i"] = 0;
        this["j"] = 0;
        this["S"] = [];
      }
      return _0x7d14e5["prototype"]["init"] = function (_0xab856e) {
        var _0x2f4340, _0x2e1107, _0xb75fb9;
        for (_0x2f4340 = 0; _0x2f4340 < 256; ++_0x2f4340) this["S"][_0x2f4340] = _0x2f4340;
        for (_0x2f4340 = _0x2e1107 = 0; _0x2f4340 < 256; ++_0x2f4340) {
          _0xb75fb9 = this["S"][_0x2f4340];
          this["S"][_0x2f4340] = this["S"][_0x2e1107 = _0x2e1107 + this["S"][_0x2f4340] + _0xab856e[_0x2f4340 % _0xab856e["length"]] & 255];
          this["S"][_0x2e1107] = _0xb75fb9;
        }
        this["i"] = 0;
        this["j"] = 0;
      }, _0x7d14e5["prototype"]["next"] = function () {
        var _0x7d14e5;
        return this["i"] = this["i"] + 1 & 255, this["j"] = this["j"] + this["S"][this["i"]] & 255, _0x7d14e5 = this["S"][this["i"]], this["S"][this["i"]] = this["S"][this["j"]], this["S"][this["j"]] = _0x7d14e5, this["S"][_0x7d14e5 + this["S"][this["i"]] & 255];
      }, _0x7d14e5;
    }(),
    _0xd1fcb7 = 256,
    _0x28fced = null;
  if (null == _0x28fced) {
    _0x28fced = [];
    var _0x234805 = void (_0x198bd8 = 0);
    var _0xde5242 = new Uint32Array(256);
    //if (window["crypto"] && window["crypto"]["getRandomValues"]) {} else {
    //  global = new Array();
    //  window = new Array();
    //}
  }
  function _0x50b972() {
    if (null == _0x53c881) {
      for (_0x53c881 = new _0x19fde7(); _0x198bd8 < _0xd1fcb7;) {
        var _0x20544c = Math["floor"](65536);
        _0x28fced[_0x198bd8++] = 255 & _0x20544c;
      }
      for (_0x53c881["init"](_0x28fced), _0x198bd8 = 0; _0x198bd8 < _0x28fced["length"]; ++_0x198bd8) _0x28fced[_0x198bd8] = 0;
      _0x198bd8 = 0;
    }
    return _0x53c881["next"]();
  }
  var _0x1c692d = function () {
      function _0x3f7749() {}
      return _0x3f7749["prototype"]["nextBytes"] = function (_0x218966) {
        for (var _0x52ef1d = 0; _0x52ef1d < _0x218966["length"]; ++_0x52ef1d) _0x218966[_0x52ef1d] = _0x50b972();
      }, _0x3f7749;
    }(),
    _0x56aa3c = function () {
      function _0x19243d() {
        this["n"] = null;
        this["e"] = 0;
        this["d"] = null;
        this["p"] = null;
        this["q"] = null;
        this["dmp1"] = null;
        this["dmq1"] = null;
        this["coeff"] = null;
      }
      return _0x19243d["prototype"]["doPublic"] = function (_0x58c5a3) {
        return _0x58c5a3["modPowInt"](this["e"], this["n"]);
      }, _0x19243d["prototype"]["doPrivate"] = function (_0x126700) {
        if (null == this["p"] || null == this["q"]) return _0x126700["modPow"](this["d"], this["n"]);
        for (var _0x3c2a80 = _0x126700["mod"](this["p"])["modPow"](this["dmp1"], this["p"]), _0x4a698e = _0x126700["mod"](this["q"])["modPow"](this["dmq1"], this["q"]); _0x3c2a80["compareTo"](_0x4a698e) < 0;) _0x3c2a80 = _0x3c2a80["add"](this["p"]);
        return _0x3c2a80["subtract"](_0x4a698e)["multiply"](this["coeff"])["mod"](this["p"])["multiply"](this["q"])["add"](_0x4a698e);
      }, _0x19243d["prototype"]["setPublic"] = function (_0x4cbfaf, _0x576a9c) {
        if (null != _0x4cbfaf && null != _0x576a9c && 0 < _0x4cbfaf["length"] && 0 < _0x576a9c["length"]) {
          this["n"] = _0x5baf06(_0x4cbfaf, 16), this["e"] = parseInt(_0x576a9c, 16);
        } else {
          console["error"]("Invalid RSA public key");
        }
      }, _0x19243d["prototype"]["encrypt"] = function (_0x35dde9) {
        var _0x1971c2 = function (_0x53504c, _0x520615) {
          if (_0x520615 < _0x53504c["length"] + 11) return console["error"]("Message too long for RSA"), null;
          for (var _0x395439 = [], _0x37e200 = _0x53504c["length"] - 1; 0 <= _0x37e200 && 0 < _0x520615;) {
            var _0x27c7a6 = _0x53504c["charCodeAt"](_0x37e200--);
            if (_0x27c7a6 < 128) {
              _0x395439[--_0x520615] = _0x27c7a6;
            } else {
              if (127 < _0x27c7a6 && _0x27c7a6 < 2048) {
                _0x395439[--_0x520615] = 63 & _0x27c7a6 | 128, _0x395439[--_0x520615] = _0x27c7a6 >> 6 | 192;
              } else {
                _0x395439[--_0x520615] = 63 & _0x27c7a6 | 128, _0x395439[--_0x520615] = _0x27c7a6 >> 6 & 63 | 128, _0x395439[--_0x520615] = _0x27c7a6 >> 12 | 224;
              }
            }
          }
          _0x395439[--_0x520615] = 0;
          for (var _0x5499c2 = new _0x1c692d(), _0x1e4d4c = []; 2 < _0x520615;) {
            for (_0x1e4d4c[0] = 0; 0 == _0x1e4d4c[0];) _0x5499c2["nextBytes"](_0x1e4d4c);
            _0x395439[--_0x520615] = _0x1e4d4c[0];
          }
          return _0x395439[--_0x520615] = 2, _0x395439[--_0x520615] = 0, new _0x2935af(_0x395439);
        }(_0x35dde9, this["n"]["bitLength"]() + 7 >> 3);
        if (null == _0x1971c2) return null;
        var _0x173b94 = this["doPublic"](_0x1971c2);
        if (null == _0x173b94) return null;
        var _0x423d43 = _0x173b94["toString"](16);
        return 0 == (1 & _0x423d43["length"]) ? _0x423d43 : "0" + _0x423d43;
      }, _0x19243d["prototype"]["setPrivate"] = function (_0x469101, _0x203293, _0x12e10b) {
        if (null != _0x469101 && null != _0x203293 && 0 < _0x469101["length"] && 0 < _0x203293["length"]) {
          this["n"] = _0x5baf06(_0x469101, 16), this["e"] = parseInt(_0x203293, 16), this["d"] = _0x5baf06(_0x12e10b, 16);
        } else {
          console["error"]("Invalid RSA private key");
        }
      }, _0x19243d["prototype"]["setPrivateEx"] = function (_0x41060b, _0x211748, _0x387c5c, _0x369faf, _0x41a59f, _0x1457b9, _0x38e7ba, _0x1a1f21) {
        if (null != _0x41060b && null != _0x211748 && 0 < _0x41060b["length"] && 0 < _0x211748["length"]) {
          this["n"] = _0x5baf06(_0x41060b, 16), this["e"] = parseInt(_0x211748, 16), this["d"] = _0x5baf06(_0x387c5c, 16), this["p"] = _0x5baf06(_0x369faf, 16), this["q"] = _0x5baf06(_0x41a59f, 16), this["dmp1"] = _0x5baf06(_0x1457b9, 16), this["dmq1"] = _0x5baf06(_0x38e7ba, 16), this["coeff"] = _0x5baf06(_0x1a1f21, 16);
        } else {
          console["error"]("Invalid RSA private key");
        }
      }, _0x19243d["prototype"]["generate"] = function (_0x54cded, _0x252e2c) {
        var _0x4d5ffc = new _0x1c692d(),
          _0x196d88 = _0x54cded >> 1;
        this["e"] = parseInt(_0x252e2c, 16);
        for (var _0x504a1c = new _0x2935af(_0x252e2c, 16);;) {
          for (; this["p"] = new _0x2935af(_0x54cded - _0x196d88, 1, _0x4d5ffc), 0 != this["p"]["subtract"](_0x2935af["ONE"])["gcd"](_0x504a1c)["compareTo"](_0x2935af["ONE"]) || !this["p"]["isProbablePrime"](10););
          for (; this["q"] = new _0x2935af(_0x196d88, 1, _0x4d5ffc), 0 != this["q"]["subtract"](_0x2935af["ONE"])["gcd"](_0x504a1c)["compareTo"](_0x2935af["ONE"]) || !this["q"]["isProbablePrime"](10););
          if (this["p"]["compareTo"](this["q"]) <= 0) {
            var _0x2c514a = this["p"];
            this["p"] = this["q"];
            this["q"] = _0x2c514a;
          }
          var _0x5835c5 = this["p"]["subtract"](_0x2935af["ONE"]),
            _0xcff2db = this["q"]["subtract"](_0x2935af["ONE"]),
            _0x38f7ca = _0x5835c5["multiply"](_0xcff2db);
          if (0 == _0x38f7ca["gcd"](_0x504a1c)["compareTo"](_0x2935af["ONE"])) {
            this["n"] = this["p"]["multiply"](this["q"]);
            this["d"] = _0x504a1c["modInverse"](_0x38f7ca);
            this["dmp1"] = this["d"]["mod"](_0x5835c5);
            this["dmq1"] = this["d"]["mod"](_0xcff2db);
            this["coeff"] = this["q"]["modInverse"](this["p"]);
            break;
          }
        }
      }, _0x19243d["prototype"]["decrypt"] = function (_0x36f0fa) {
        var _0x481c7d = _0x5baf06(_0x36f0fa, 16),
          _0x3758ed = this["doPrivate"](_0x481c7d);
        return null == _0x3758ed ? null : function (_0x1a083f, _0x497a1f) {
          for (var _0x459609 = _0x1a083f["toByteArray"](), _0x467192 = 0; _0x467192 < _0x459609["length"] && 0 == _0x459609[_0x467192];) ++_0x467192;
          if (_0x459609["length"] - _0x467192 != _0x497a1f - 1 || 2 != _0x459609[_0x467192]) return null;
          for (++_0x467192; 0 != _0x459609[_0x467192];) if (++_0x467192 >= _0x459609["length"]) return null;
          for (var _0x1ac7fe = ""; ++_0x467192 < _0x459609["length"];) {
            var _0x53487d = 255 & _0x459609[_0x467192];
            if (_0x53487d < 128) {
              _0x1ac7fe += String["fromCharCode"](_0x53487d);
            } else {
              if (191 < _0x53487d && _0x53487d < 224) {
                _0x1ac7fe += String["fromCharCode"]((31 & _0x53487d) << 6 | 63 & _0x459609[_0x467192 + 1]), ++_0x467192;
              } else {
                _0x1ac7fe += String["fromCharCode"]((15 & _0x53487d) << 12 | (63 & _0x459609[_0x467192 + 1]) << 6 | 63 & _0x459609[_0x467192 + 2]), _0x467192 += 2;
              }
            }
          }
          return _0x1ac7fe;
        }(_0x3758ed, this["n"]["bitLength"]() + 7 >> 3);
      }, _0x19243d["prototype"]["generateAsync"] = function (_0x4639cc, _0x5e6d24, _0x2ea27d) {
        var _0x3ea2ab = new _0x1c692d(),
          _0x17c5f2 = _0x4639cc >> 1;
        this["e"] = parseInt(_0x5e6d24, 16);
        var _0x5a2115 = new _0x2935af(_0x5e6d24, 16),
          _0x18f76e = this,
          _0x23dc1f = function () {
            var _0x5e6d24 = function () {
                if (_0x18f76e["p"]["compareTo"](_0x18f76e["q"]) <= 0) {
                  var _0x4639cc = _0x18f76e["p"];
                  _0x18f76e["p"] = _0x18f76e["q"];
                  _0x18f76e["q"] = _0x4639cc;
                }
                var _0x5e6d24 = _0x18f76e["p"]["subtract"](_0x2935af["ONE"]),
                  _0x343fe4 = _0x18f76e["q"]["subtract"](_0x2935af["ONE"]),
                  _0x22c084 = _0x5e6d24["multiply"](_0x343fe4);
                if (0 == _0x22c084["gcd"](_0x5a2115)["compareTo"](_0x2935af["ONE"])) {
                  _0x18f76e["n"] = _0x18f76e["p"]["multiply"](_0x18f76e["q"]), _0x18f76e["d"] = _0x5a2115["modInverse"](_0x22c084), _0x18f76e["dmp1"] = _0x18f76e["d"]["mod"](_0x5e6d24), _0x18f76e["dmq1"] = _0x18f76e["d"]["mod"](_0x343fe4), _0x18f76e["coeff"] = _0x18f76e["q"]["modInverse"](_0x18f76e["p"]), setTimeout(function () {
                    _0x2ea27d();
                  }, 0);
                } else {
                  setTimeout(_0x23dc1f, 0);
                }
              },
              _0x3870eb = function () {
                _0x18f76e["q"] = _0x425021();
                _0x18f76e["q"]["fromNumberAsync"](_0x17c5f2, 1, _0x3ea2ab, function () {
                  _0x18f76e["q"]["subtract"](_0x2935af["ONE"])["gcda"](_0x5a2115, function (_0x239128) {
                    if (0 == _0x239128["compareTo"](_0x2935af["ONE"]) && _0x18f76e["q"]["isProbablePrime"](10)) {
                      setTimeout(_0x5e6d24, 0);
                    } else {
                      setTimeout(_0x3870eb, 0);
                    }
                  });
                });
              },
              _0xfc89d3 = function () {
                _0x18f76e["p"] = _0x425021();
                _0x18f76e["p"]["fromNumberAsync"](_0x4639cc - _0x17c5f2, 1, _0x3ea2ab, function () {
                  _0x18f76e["p"]["subtract"](_0x2935af["ONE"])["gcda"](_0x5a2115, function (_0x4c257a) {
                    if (0 == _0x4c257a["compareTo"](_0x2935af["ONE"]) && _0x18f76e["p"]["isProbablePrime"](10)) {
                      setTimeout(_0x3870eb, 0);
                    } else {
                      setTimeout(_0xfc89d3, 0);
                    }
                  });
                });
              };
            setTimeout(_0xfc89d3, 0);
          };
        setTimeout(_0x23dc1f, 0);
      }, _0x19243d["prototype"]["sign"] = function (_0x555d0b, _0x29b8d0, _0x17eec9) {
        var _0x3729c9 = function (_0x3b056e, _0x3f4c23) {
          if (_0x3f4c23 < _0x3b056e["length"] + 22) return console["error"]("Message too long for RSA"), null;
          for (var _0x17eec9 = _0x3f4c23 - _0x3b056e["length"] - 6, _0x20f282 = "", _0x549bce = 0; _0x549bce < _0x17eec9; _0x549bce += 2) _0x20f282 += "ff";
          return _0x5baf06("0001" + _0x20f282 + "00" + _0x3b056e, 16);
        }((_0x50b97f[_0x17eec9] || "") + _0x29b8d0(_0x555d0b)["toString"](), this["n"]["bitLength"]() / 4);
        if (null == _0x3729c9) return null;
        var _0x249548 = this["doPrivate"](_0x3729c9);
        if (null == _0x249548) return null;
        var _0x2c9b88 = _0x249548["toString"](16);
        return 0 == (1 & _0x2c9b88["length"]) ? _0x2c9b88 : "0" + _0x2c9b88;
      }, _0x19243d["prototype"]["verify"] = function (_0xc36bda, _0x2a719f, _0x3ab3b4) {
        var _0x4ddb5d = _0x5baf06(_0x2a719f, 16),
          _0x214cd6 = this["doPublic"](_0x4ddb5d);
        return null == _0x214cd6 ? null : function (_0x43ea64) {
          for (var _0x2a719f in _0x50b97f) if (_0x50b97f["hasOwnProperty"](_0x2a719f)) {
            var _0x3ab3b4 = _0x50b97f[_0x2a719f],
              _0xb76119 = _0x3ab3b4["length"];
            if (_0x43ea64["substr"](0, _0xb76119) == _0x3ab3b4) return _0x43ea64["substr"](_0xb76119);
          }
          return _0x43ea64;
        }(_0x214cd6["toString"](16)["replace"](/^1f+00/, "")) == _0x3ab3b4(_0xc36bda)["toString"]();
      }, _0x19243d;
    }(),
    _0x50b97f = {
      "md2": "3020300c06082a864886f70d020205000410",
      "md5": "3020300c06082a864886f70d020505000410",
      "sha1": "3021300906052b0e03021a05000414",
      "sha224": "302d300d06096086480165030402040500041c",
      "sha256": "3031300d060960864801650304020105000420",
      "sha384": "3041300d060960864801650304020205000430",
      "sha512": "3051300d060960864801650304020305000440",
      "ripemd160": "3021300906052b2403020105000414"
    },
    _0x1bbd98 = {
      "lang": {
        "extend": function (_0x4d3756, _0x571eb4, _0x29e6d6) {
          if (!_0x571eb4 || !_0x4d3756) throw new Error("YAHOO.lang.extend failed, please check that all dependencies are included.");
          var _0x170804 = function () {};
          if (_0x170804["prototype"] = _0x571eb4["prototype"], _0x4d3756["prototype"] = new _0x170804(), (_0x4d3756["prototype"]["constructor"] = _0x4d3756)["superclass"] = _0x571eb4["prototype"], _0x571eb4["prototype"]["constructor"] == Object["prototype"]["constructor"] && (_0x571eb4["prototype"]["constructor"] = _0x571eb4), _0x29e6d6) {
            var _0x3693d9;
            for (_0x3693d9 in _0x29e6d6) _0x4d3756["prototype"][_0x3693d9] = _0x29e6d6[_0x3693d9];
            var _0xcfffe5 = function () {},
              _0x129a8a = ["toString", "valueOf"];
            try {
              /MSIE/["test"](navigator["uA"]) && (_0xcfffe5 = function (_0x22041f, _0x2959db) {
                for (_0x3693d9 = 0; _0x3693d9 < _0x129a8a["length"]; _0x3693d9 += 1) {
                  var _0x29e6d6 = _0x129a8a[_0x3693d9],
                    _0x42e818 = _0x2959db[_0x29e6d6];
                  "function" == typeof _0x42e818 && _0x42e818 != Object["prototype"][_0x29e6d6] && (_0x22041f[_0x29e6d6] = _0x42e818);
                }
              });
            } catch (_0xb09e15) {}
            _0xcfffe5(_0x4d3756["prototype"], _0x29e6d6);
          }
        }
      }
    };
  var _0x2f02a5 = {
    "asn1": {}
  };
  void 0 !== _0x2f02a5["asn1"] && _0x2f02a5["asn1"];
  _0x2f02a5["asn1"]["ASN1Util"] = new function () {
    this["integerToByteHex"] = function (_0x383c06) {
      var _0x1f54af = _0x383c06["toString"](16);
      return _0x1f54af["length"] % 2 == 1 && (_0x1f54af = "0" + _0x1f54af), _0x1f54af;
    };
    this["bigIntToMinTwosComplementsHex"] = function (_0xe3f045) {
      var _0x252856 = _0xe3f045["toString"](16);
      if ("-" != _0x252856["substr"](0, 1)) {
        if (_0x252856["length"] % 2 == 1) {
          _0x252856 = "0" + _0x252856;
        } else {
          _0x252856["match"](/^[0-7]/) || (_0x252856 = "00" + _0x252856);
        }
      } else {
        var _0x10e08d = _0x252856["substr"](1)["length"];
        if (_0x10e08d % 2 == 1) {
          _0x10e08d += 1;
        } else {
          _0x252856["match"](/^[0-7]/) || (_0x10e08d += 2);
        }
        for (var _0x2a6131 = "", _0x4d211b = 0; _0x4d211b < _0x10e08d; _0x4d211b++) _0x2a6131 += "f";
        _0x252856 = new _0x2935af(_0x2a6131, 16)["xor"](_0xe3f045)["add"](_0x2935af["ONE"])["toString"](16)["replace"](/^-/, "");
      }
      return _0x252856;
    };
    this["getPEMStringFromHex"] = function (_0x54a841, _0x26cd90) {
      return hextopem(_0x54a841, _0x26cd90);
    };
    this["newObject"] = function (_0x259224) {
      var _0x1214de = _0x2f02a5["asn1"],
        _0x14f946 = _0x1214de["DERBoolean"],
        _0x53b6a1 = _0x1214de["DERInteger"],
        _0x2da46f = _0x1214de["DERBitString"],
        _0x20daf9 = _0x1214de["DEROctetString"],
        _0x262814 = _0x1214de["DERNull"],
        _0x348e5d = _0x1214de["DERObjectIdentifier"],
        _0x51bc19 = _0x1214de["DEREnumerated"],
        _0x4c0132 = _0x1214de["DERUTF8String"],
        _0xdf9f07 = _0x1214de["DERNumericString"],
        _0x4caa63 = _0x1214de["DERPrintableString"],
        _0x2e7100 = _0x1214de["DERTeletexString"],
        _0x5aca82 = _0x1214de["DERIA5String"],
        _0x3a3cf2 = _0x1214de["DERUTCTime"],
        _0x5ca427 = _0x1214de["DERGeneralizedTime"],
        _0x1f3b2e = _0x1214de["DERSequence"],
        _0x481792 = _0x1214de["DERSet"],
        _0x4c37c7 = _0x1214de["DERTaggedObject"],
        _0x5e17db = _0x1214de["ASN1Util"]["newObject"],
        _0x2f7fd0 = Object["keys"](_0x259224);
      if (1 != _0x2f7fd0["length"]) throw "key of param shall be only one.";
      var _0x2c13e9 = _0x2f7fd0[0];
      if (-1 == ":bool:int:bitstr:octstr:null:oid:enum:utf8str:numstr:prnstr:telstr:ia5str:utctime:gentime:seq:set:tag:"["indexOf"](":" + _0x2c13e9 + ":")) throw "undefined key: " + _0x2c13e9;
      if ("bool" == _0x2c13e9) return new _0x14f946(_0x259224[_0x2c13e9]);
      if ("int" == _0x2c13e9) return new _0x53b6a1(_0x259224[_0x2c13e9]);
      if ("bitstr" == _0x2c13e9) return new _0x2da46f(_0x259224[_0x2c13e9]);
      if ("octstr" == _0x2c13e9) return new _0x20daf9(_0x259224[_0x2c13e9]);
      if ("null" == _0x2c13e9) return new _0x262814(_0x259224[_0x2c13e9]);
      if ("oid" == _0x2c13e9) return new _0x348e5d(_0x259224[_0x2c13e9]);
      if ("enum" == _0x2c13e9) return new _0x51bc19(_0x259224[_0x2c13e9]);
      if ("utf8str" == _0x2c13e9) return new _0x4c0132(_0x259224[_0x2c13e9]);
      if ("numstr" == _0x2c13e9) return new _0xdf9f07(_0x259224[_0x2c13e9]);
      if ("prnstr" == _0x2c13e9) return new _0x4caa63(_0x259224[_0x2c13e9]);
      if ("telstr" == _0x2c13e9) return new _0x2e7100(_0x259224[_0x2c13e9]);
      if ("ia5str" == _0x2c13e9) return new _0x5aca82(_0x259224[_0x2c13e9]);
      if ("utctime" == _0x2c13e9) return new _0x3a3cf2(_0x259224[_0x2c13e9]);
      if ("gentime" == _0x2c13e9) return new _0x5ca427(_0x259224[_0x2c13e9]);
      if ("seq" == _0x2c13e9) {
        for (var _0x6b7664 = _0x259224[_0x2c13e9], _0x14ff88 = [], _0x286e3d = 0; _0x286e3d < _0x6b7664["length"]; _0x286e3d++) {
          var _0xe9c724 = _0x5e17db(_0x6b7664[_0x286e3d]);
          _0x14ff88["push"](_0xe9c724);
        }
        return new _0x1f3b2e({
          "array": _0x14ff88
        });
      }
      if ("set" == _0x2c13e9) {
        for (_0x6b7664 = _0x259224[_0x2c13e9], _0x14ff88 = [], _0x286e3d = 0; _0x286e3d < _0x6b7664["length"]; _0x286e3d++) {
          _0xe9c724 = _0x5e17db(_0x6b7664[_0x286e3d]);
          _0x14ff88["push"](_0xe9c724);
        }
        return new _0x481792({
          "array": _0x14ff88
        });
      }
      if ("tag" == _0x2c13e9) {
        var _0x2aa86a = _0x259224[_0x2c13e9];
        if ("[object Array]" === Object["prototype"]["toString"]["call"](_0x2aa86a) && 3 == _0x2aa86a["length"]) {
          var _0x335059 = _0x5e17db(_0x2aa86a[2]);
          return new _0x4c37c7({
            "tag": _0x2aa86a[0],
            "explicit": _0x2aa86a[1],
            "obj": _0x335059
          });
        }
        var _0x59154d = {};
        if (void 0 !== _0x2aa86a["explicit"] && (_0x59154d["explicit"] = _0x2aa86a["explicit"]), void 0 !== _0x2aa86a["tag"] && (_0x59154d["tag"] = _0x2aa86a["tag"]), void 0 === _0x2aa86a["obj"]) throw "obj shall be specified for 'tag'.";
        return _0x59154d["obj"] = _0x5e17db(_0x2aa86a["obj"]), new _0x4c37c7(_0x59154d);
      }
    };
    this["jsonToASN1HEX"] = function (_0x114a10) {
      return this["newObject"](_0x114a10)["getEncodedHex"]();
    };
  }();
  _0x2f02a5["asn1"]["ASN1Util"]["oidHexToInt"] = function (_0x106182) {
    for (var _0x58b2b3 = "", _0xe18f61 = parseInt(_0x106182["substr"](0, 2), 16), _0x3c33d9 = (_0x58b2b3 = Math["floor"](_0xe18f61 / 40) + "." + _0xe18f61 % 40, ""), _0x5b109b = 2; _0x5b109b < _0x106182["length"]; _0x5b109b += 2) {
      var _0x90a44a = ("00000000" + parseInt(_0x106182["substr"](_0x5b109b, 2), 16)["toString"](2))["slice"](-8);
      _0x3c33d9 += _0x90a44a["substr"](1, 7);
      "0" == _0x90a44a["substr"](0, 1) && (_0x58b2b3 = _0x58b2b3 + "." + new _0x2935af(_0x3c33d9, 2)["toString"](10), _0x3c33d9 = "");
    }
    return _0x58b2b3;
  };
  _0x2f02a5["asn1"]["ASN1Util"]["oidIntToHex"] = function (_0x2f3bd9) {
    var _0x26d55e = function (_0x2c5e07) {
        var _0x352e24 = _0x2c5e07["toString"](16);
        return 1 == _0x352e24["length"] && (_0x352e24 = "0" + _0x352e24), _0x352e24;
      },
      _0x425a60 = function (_0x50a16b) {
        var _0x45c230 = "",
          _0x3505ab = new _0x2935af(_0x50a16b, 10)["toString"](2),
          _0x188633 = 7 - _0x3505ab["length"] % 7;
        7 == _0x188633 && (_0x188633 = 0);
        for (var _0x22ef93 = "", _0x53f4f8 = 0; _0x53f4f8 < _0x188633; _0x53f4f8++) _0x22ef93 += "0";
        for (_0x3505ab = _0x22ef93 + _0x3505ab, _0x53f4f8 = 0; _0x53f4f8 < _0x3505ab["length"] - 1; _0x53f4f8 += 7) {
          var _0x38fdfe = _0x3505ab["substr"](_0x53f4f8, 7);
          _0x53f4f8 != _0x3505ab["length"] - 7 && (_0x38fdfe = "1" + _0x38fdfe);
          _0x45c230 += _0x26d55e(parseInt(_0x38fdfe, 2));
        }
        return _0x45c230;
      };
    if (!_0x2f3bd9["match"](/^[0-9.]+$/)) throw "malformed oid string: " + _0x2f3bd9;
    var _0x3a63ad = "",
      _0x490035 = _0x2f3bd9["split"]("."),
      _0x2a5dc2 = 40 * parseInt(_0x490035[0]) + parseInt(_0x490035[1]);
    _0x3a63ad += _0x26d55e(_0x2a5dc2);
    _0x490035["splice"](0, 2);
    for (var _0x511933 = 0; _0x511933 < _0x490035["length"]; _0x511933++) _0x3a63ad += _0x425a60(_0x490035[_0x511933]);
    return _0x3a63ad;
  };
  _0x2f02a5["asn1"]["ASN1Object"] = function () {
    this["getLengthHexFromValue"] = function () {
      if (void 0 === this["hV"] || null == this["hV"]) throw "this.hV is null or undefined.";
      if (this["hV"]["length"] % 2 == 1) throw "value hex must be even length: n=" + ""["length"] + ",v=" + this["hV"];
      var _0x20544c = this["hV"]["length"] / 2,
        _0x10182d = _0x20544c["toString"](16);
      if (_0x10182d["length"] % 2 == 1 && (_0x10182d = "0" + _0x10182d), _0x20544c < 128) return _0x10182d;
      var _0x47c599 = _0x10182d["length"] / 2;
      if (15 < _0x47c599) throw "ASN.1 length too long to represent by 8x: n = " + _0x20544c["toString"](16);
      return (128 + _0x47c599)["toString"](16) + _0x10182d;
    };
    this["getEncodedHex"] = function () {
      return (null == this["hTLV"] || this["isModified"]) && (this["hV"] = this["getFreshValueHex"](), this["hL"] = this["getLengthHexFromValue"](), this["hTLV"] = this["hT"] + this["hL"] + this["hV"], this["isModified"] = !1), this["hTLV"];
    };
    this["getValueHex"] = function () {
      return this["getEncodedHex"](), this["hV"];
    };
    this["getFreshValueHex"] = function () {
      return "";
    };
  };
  _0x2f02a5["asn1"]["DERAbstractString"] = function (_0x545af5) {
    _0x2f02a5["asn1"]["DERAbstractString"]["superclass"]["constructor"]["call"](this);
    this["getString"] = function () {
      return this["s"];
    };
    this["setString"] = function (_0x523241) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["s"] = _0x523241;
      this["hV"] = stohex(this["s"]);
    };
    this["setStringHex"] = function (_0x4b7ad1) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["s"] = null;
      this["hV"] = _0x4b7ad1;
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
    void 0 !== _0x545af5 && ("string" == typeof _0x545af5 ? this["setString"](_0x545af5) : void 0 !== _0x545af5["str"] ? this["setString"](_0x545af5["str"]) : void 0 !== _0x545af5["hex"] && this["setStringHex"](_0x545af5["hex"]));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERAbstractString"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERAbstractTime"] = function (_0x3e485a) {
    _0x2f02a5["asn1"]["DERAbstractTime"]["superclass"]["constructor"]["call"](this);
    this["localDateToUTC"] = function (_0x2349ab) {
      return utc = _0x2349ab["getTime"]() + 60000 * _0x2349ab["getTimezoneOffset"](), new Date(utc);
    };
    this["formatDate"] = function (_0x4de640, _0x106cbb, _0x54ca82) {
      var _0x2a9c6e = this["zeroPadding"],
        _0x5cba9a = this["localDateToUTC"](_0x4de640),
        _0xda3fe7 = String(_0x5cba9a["getFullYear"]());
      "utc" == _0x106cbb && (_0xda3fe7 = _0xda3fe7["substr"](2, 2));
      var _0x181800 = _0xda3fe7 + _0x2a9c6e(String(_0x5cba9a["getMonth"]() + 1), 2) + _0x2a9c6e(String(_0x5cba9a["getDate"]()), 2) + _0x2a9c6e(String(_0x5cba9a["getHours"]()), 2) + _0x2a9c6e(String(_0x5cba9a["getMinutes"]()), 2) + _0x2a9c6e(String(_0x5cba9a["getSeconds"]()), 2);
      if (!0 === _0x54ca82) {
        var _0x53fec8 = _0x5cba9a["getMilliseconds"]();
        if (0 != _0x53fec8) {
          var _0x3a9091 = _0x2a9c6e(String(_0x53fec8), 3);
          _0x181800 = _0x181800 + "." + (_0x3a9091 = _0x3a9091["replace"](/[0]+$/, ""));
        }
      }
      return _0x181800 + "Z";
    };
    this["zeroPadding"] = function (_0x568a80, _0x3b884c) {
      return _0x568a80["length"] >= _0x3b884c ? _0x568a80 : new Array(_0x3b884c - _0x568a80["length"] + 1)["join"]("0") + _0x568a80;
    };
    this["getString"] = function () {
      return this["s"];
    };
    this["setString"] = function (_0x4d7947) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["s"] = _0x4d7947;
      this["hV"] = stohex(_0x4d7947);
    };
    this["setByDateValue"] = function (_0x4763a7, _0x4ee5a5, _0x4b49e3, _0x4751e7, _0x442a39, _0x11000f) {
      var _0x35b914 = new Date(Date["UTC"](_0x4763a7, _0x4ee5a5 - 1, _0x4b49e3, _0x4751e7, _0x442a39, _0x11000f, 0));
      this["setByDate"](_0x35b914);
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERAbstractTime"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERAbstractStructured"] = function (_0x67861f) {
    _0x2f02a5["asn1"]["DERAbstractString"]["superclass"]["constructor"]["call"](this);
    this["setByASN1ObjectArray"] = function (_0x22cc43) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["asn1Array"] = _0x22cc43;
    };
    this["appendASN1Object"] = function (_0xe45ec3) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["asn1Array"]["push"](_0xe45ec3);
    };
    this["asn1Array"] = new Array();
    void 0 !== _0x67861f && void 0 !== _0x67861f["array"] && (this["asn1Array"] = _0x67861f["array"]);
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERAbstractStructured"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERBoolean"] = function () {
    _0x2f02a5["asn1"]["DERBoolean"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "01";
    this["hTLV"] = "0101ff";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERBoolean"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERInteger"] = function (_0x308af7) {
    _0x2f02a5["asn1"]["DERInteger"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "02";
    this["setByBigInteger"] = function (_0x271a44) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["hV"] = _0x2f02a5["asn1"]["ASN1Util"]["bigIntToMinTwosComplementsHex"](_0x271a44);
    };
    this["setByInteger"] = function (_0x3a56cd) {
      var _0x38e81a = new _0x2935af(String(_0x3a56cd), 10);
      this["setByBigInteger"](_0x38e81a);
    };
    this["setValueHex"] = function (_0x11c150) {
      this["hV"] = _0x11c150;
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
    void 0 !== _0x308af7 && (void 0 !== _0x308af7["bigint"] ? this["setByBigInteger"](_0x308af7["bigint"]) : void 0 !== _0x308af7["int"] ? this["setByInteger"](_0x308af7["int"]) : "number" == typeof _0x308af7 ? this["setByInteger"](_0x308af7) : void 0 !== _0x308af7["hex"] && this["setValueHex"](_0x308af7["hex"]));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERInteger"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERBitString"] = function (_0x14b5e6) {
    if (void 0 !== _0x14b5e6 && void 0 !== _0x14b5e6["obj"]) {
      var _0x42353f = _0x2f02a5["asn1"]["ASN1Util"]["newObject"](_0x14b5e6["obj"]);
      _0x14b5e6["hex"] = "00" + _0x42353f["getEncodedHex"]();
    }
    _0x2f02a5["asn1"]["DERBitString"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "03";
    this["setHexValueIncludingUnusedBits"] = function (_0x349c4d) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["hV"] = _0x349c4d;
    };
    this["setUnusedBitsAndHexValue"] = function (_0x4ef8fc, _0x444664) {
      if (_0x4ef8fc < 0 || 7 < _0x4ef8fc) throw "unused bits shall be from 0 to 7: u = " + _0x4ef8fc;
      var _0x3fb86b = "0" + _0x4ef8fc;
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["hV"] = _0x3fb86b + _0x444664;
    };
    this["setByBinaryString"] = function (_0x1c0556) {
      var _0x483edc = 8 - (_0x1c0556 = _0x1c0556["replace"](/0+$/, ""))["length"] % 8;
      8 == _0x483edc && (_0x483edc = 0);
      for (var _0x2e0909 = 0; _0x2e0909 <= _0x483edc; _0x2e0909++) _0x1c0556 += "0";
      var _0x56ee31 = "";
      for (_0x2e0909 = 0; _0x2e0909 < _0x1c0556["length"] - 1; _0x2e0909 += 8) {
        var _0x23450d = _0x1c0556["substr"](_0x2e0909, 8),
          _0x265248 = parseInt(_0x23450d, 2)["toString"](16);
        1 == _0x265248["length"] && (_0x265248 = "0" + _0x265248);
        _0x56ee31 += _0x265248;
      }
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["hV"] = "0" + _0x483edc + _0x56ee31;
    };
    this["setByBooleanArray"] = function (_0x431fb6) {
      for (var _0x2e32c5 = "", _0x2d85d7 = 0; _0x2d85d7 < _0x431fb6["length"]; _0x2d85d7++) _0x2e32c5 += 1 == _0x431fb6[_0x2d85d7] ? "1" : "0";
      this["setByBinaryString"](_0x2e32c5);
    };
    this["newFalseArray"] = function (_0x344f8f) {
      for (var _0x540ed4 = new Array(_0x344f8f), _0x50e039 = 0; _0x50e039 < _0x344f8f; _0x50e039++) _0x540ed4[_0x50e039] = !1;
      return _0x540ed4;
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
    void 0 !== _0x14b5e6 && ("string" == typeof _0x14b5e6 && _0x14b5e6["toLowerCase"]()["match"](/^[0-9a-f]+$/) ? this["setHexValueIncludingUnusedBits"](_0x14b5e6) : void 0 !== _0x14b5e6["hex"] ? this["setHexValueIncludingUnusedBits"](_0x14b5e6["hex"]) : void 0 !== _0x14b5e6["bin"] ? this["setByBinaryString"](_0x14b5e6["bin"]) : void 0 !== _0x14b5e6["array"] && this["setByBooleanArray"](_0x14b5e6["array"]));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERBitString"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DEROctetString"] = function (_0x443077) {
    if (void 0 !== _0x443077 && void 0 !== _0x443077["obj"]) {
      var _0xe36981 = _0x2f02a5["asn1"]["ASN1Util"]["newObject"](_0x443077["obj"]);
      _0x443077["hex"] = _0xe36981["getEncodedHex"]();
    }
    _0x2f02a5["asn1"]["DEROctetString"]["superclass"]["constructor"]["call"](this, _0x443077);
    this["hT"] = "04";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DEROctetString"], _0x2f02a5["asn1"]["DERAbstractString"]);
  _0x2f02a5["asn1"]["DERNull"] = function () {
    _0x2f02a5["asn1"]["DERNull"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "05";
    this["hTLV"] = "0500";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERNull"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERObjectIdentifier"] = function (_0x45f0a0) {
    var _0x9f399 = function (_0x2bd6b2) {
        var _0x491b72 = _0x2bd6b2["toString"](16);
        return 1 == _0x491b72["length"] && (_0x491b72 = "0" + _0x491b72), _0x491b72;
      },
      _0x5949c2 = function (_0x4722d4) {
        var _0x1898e = "",
          _0x45a4ff = new _0x2935af(_0x4722d4, 10)["toString"](2),
          _0x24b6ed = 7 - _0x45a4ff["length"] % 7;
        7 == _0x24b6ed && (_0x24b6ed = 0);
        for (var _0x1094f0 = "", _0x5c3350 = 0; _0x5c3350 < _0x24b6ed; _0x5c3350++) _0x1094f0 += "0";
        for (_0x45a4ff = _0x1094f0 + _0x45a4ff, _0x5c3350 = 0; _0x5c3350 < _0x45a4ff["length"] - 1; _0x5c3350 += 7) {
          var _0x520ee5 = _0x45a4ff["substr"](_0x5c3350, 7);
          _0x5c3350 != _0x45a4ff["length"] - 7 && (_0x520ee5 = "1" + _0x520ee5);
          _0x1898e += _0x9f399(parseInt(_0x520ee5, 2));
        }
        return _0x1898e;
      };
    _0x2f02a5["asn1"]["DERObjectIdentifier"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "06";
    this["setValueHex"] = function (_0x26f4c9) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["s"] = null;
      this["hV"] = _0x26f4c9;
    };
    this["setValueOidString"] = function (_0x25cb5a) {
      if (!_0x25cb5a["match"](/^[0-9.]+$/)) throw "malformed oid string: " + _0x25cb5a;
      var _0x2614db = "",
        _0x492f86 = _0x25cb5a["split"]("."),
        _0x424dce = 40 * parseInt(_0x492f86[0]) + parseInt(_0x492f86[1]);
      _0x2614db += _0x9f399(_0x424dce);
      _0x492f86["splice"](0, 2);
      for (var _0x25c285 = 0; _0x25c285 < _0x492f86["length"]; _0x25c285++) _0x2614db += _0x5949c2(_0x492f86[_0x25c285]);
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["s"] = null;
      this["hV"] = _0x2614db;
    };
    this["setValueName"] = function (_0x2eae42) {
      var _0x14d959 = _0x2f02a5["asn1"]["x509"]["OID"]["name2oid"](_0x2eae42);
      if ("" === _0x14d959) throw "DERObjectIdentifier oidName undefined: " + _0x2eae42;
      this["setValueOidString"](_0x14d959);
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
    void 0 !== _0x45f0a0 && ("string" == typeof _0x45f0a0 ? _0x45f0a0["match"](/^[0-2].[0-9.]+$/) ? this["setValueOidString"](_0x45f0a0) : this["setValueName"](_0x45f0a0) : void 0 !== _0x45f0a0["oid"] ? this["setValueOidString"](_0x45f0a0["oid"]) : void 0 !== _0x45f0a0["hex"] ? this["setValueHex"](_0x45f0a0["hex"]) : void 0 !== _0x45f0a0["name"] && this["setValueName"](_0x45f0a0["name"]));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERObjectIdentifier"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DEREnumerated"] = function (_0x37e6d7) {
    _0x2f02a5["asn1"]["DEREnumerated"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "0a";
    this["setByBigInteger"] = function (_0x31f2a7) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["hV"] = _0x2f02a5["asn1"]["ASN1Util"]["bigIntToMinTwosComplementsHex"](_0x31f2a7);
    };
    this["setByInteger"] = function (_0x3674ed) {
      var _0x4a14b7 = new _0x2935af(String(_0x3674ed), 10);
      this["setByBigInteger"](_0x4a14b7);
    };
    this["setValueHex"] = function (_0x3da022) {
      this["hV"] = _0x3da022;
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
    void 0 !== _0x37e6d7 && (void 0 !== _0x37e6d7["int"] ? this["setByInteger"](_0x37e6d7["int"]) : "number" == typeof _0x37e6d7 ? this["setByInteger"](_0x37e6d7) : void 0 !== _0x37e6d7["hex"] && this["setValueHex"](_0x37e6d7["hex"]));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DEREnumerated"], _0x2f02a5["asn1"]["ASN1Object"]);
  _0x2f02a5["asn1"]["DERUTF8String"] = function (_0x367654) {
    _0x2f02a5["asn1"]["DERUTF8String"]["superclass"]["constructor"]["call"](this, _0x367654);
    this["hT"] = "0c";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERUTF8String"], _0x2f02a5["asn1"]["DERAbstractString"]);
  _0x2f02a5["asn1"]["DERNumericString"] = function (_0x102996) {
    _0x2f02a5["asn1"]["DERNumericString"]["superclass"]["constructor"]["call"](this, _0x102996);
    this["hT"] = "12";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERNumericString"], _0x2f02a5["asn1"]["DERAbstractString"]);
  _0x2f02a5["asn1"]["DERPrintableString"] = function (_0x48a4e8) {
    _0x2f02a5["asn1"]["DERPrintableString"]["superclass"]["constructor"]["call"](this, _0x48a4e8);
    this["hT"] = "13";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERPrintableString"], _0x2f02a5["asn1"]["DERAbstractString"]);
  _0x2f02a5["asn1"]["DERTeletexString"] = function (_0x51e1c4) {
    _0x2f02a5["asn1"]["DERTeletexString"]["superclass"]["constructor"]["call"](this, _0x51e1c4);
    this["hT"] = "14";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERTeletexString"], _0x2f02a5["asn1"]["DERAbstractString"]);
  _0x2f02a5["asn1"]["DERIA5String"] = function (_0x41620d) {
    _0x2f02a5["asn1"]["DERIA5String"]["superclass"]["constructor"]["call"](this, _0x41620d);
    this["hT"] = "16";
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERIA5String"], _0x2f02a5["asn1"]["DERAbstractString"]);
  _0x2f02a5["asn1"]["DERUTCTime"] = function (_0x2556d3) {
    _0x2f02a5["asn1"]["DERUTCTime"]["superclass"]["constructor"]["call"](this, _0x2556d3);
    this["hT"] = "17";
    this["setByDate"] = function (_0x5b710b) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["date"] = _0x5b710b;
      this["s"] = this["formatDate"](this["date"], "utc");
      this["hV"] = stohex(this["s"]);
    };
    this["getFreshValueHex"] = function () {
      return void 0 === this["date"] && void 0 === this["s"] && (this["date"] = new Date(), this["s"] = this["formatDate"](this["date"], "utc"), this["hV"] = stohex(this["s"])), this["hV"];
    };
    void 0 !== _0x2556d3 && (void 0 !== _0x2556d3["str"] ? this["setString"](_0x2556d3["str"]) : "string" == typeof _0x2556d3 && _0x2556d3["match"](/^[0-9]{12}Z$/) ? this["setString"](_0x2556d3) : void 0 !== _0x2556d3["hex"] ? this["setStringHex"](_0x2556d3["hex"]) : void 0 !== _0x2556d3["date"] && this["setByDate"](_0x2556d3["date"]));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERUTCTime"], _0x2f02a5["asn1"]["DERAbstractTime"]);
  _0x2f02a5["asn1"]["DERGeneralizedTime"] = function (_0x4bf687) {
    _0x2f02a5["asn1"]["DERGeneralizedTime"]["superclass"]["constructor"]["call"](this, _0x4bf687);
    this["hT"] = "18";
    this["withMillis"] = !1;
    this["setByDate"] = function (_0x246a1b) {
      this["hTLV"] = null;
      this["isModified"] = !0;
      this["date"] = _0x246a1b;
      this["s"] = this["formatDate"](this["date"], "gen", this["withMillis"]);
      this["hV"] = stohex(this["s"]);
    };
    this["getFreshValueHex"] = function () {
      return void 0 === this["date"] && void 0 === this["s"] && (this["date"] = new Date(), this["s"] = this["formatDate"](this["date"], "gen", this["withMillis"]), this["hV"] = stohex(this["s"])), this["hV"];
    };
    void 0 !== _0x4bf687 && (void 0 !== _0x4bf687["str"] ? this["setString"](_0x4bf687["str"]) : "string" == typeof _0x4bf687 && _0x4bf687["match"](/^[0-9]{14}Z$/) ? this["setString"](_0x4bf687) : void 0 !== _0x4bf687["hex"] ? this["setStringHex"](_0x4bf687["hex"]) : void 0 !== _0x4bf687["date"] && this["setByDate"](_0x4bf687["date"]), !0 === _0x4bf687["millis"] && (this["withMillis"] = !0));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERGeneralizedTime"], _0x2f02a5["asn1"]["DERAbstractTime"]);
  _0x2f02a5["asn1"]["DERSequence"] = function (_0x126f30) {
    _0x2f02a5["asn1"]["DERSequence"]["superclass"]["constructor"]["call"](this, _0x126f30);
    this["hT"] = "30";
    this["getFreshValueHex"] = function () {
      for (var _0x126f30 = "", _0x184d55 = 0; _0x184d55 < this["asn1Array"]["length"]; _0x184d55++) _0x126f30 += this["asn1Array"][_0x184d55]["getEncodedHex"]();
      return this["hV"] = _0x126f30, this["hV"];
    };
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERSequence"], _0x2f02a5["asn1"]["DERAbstractStructured"]);
  _0x2f02a5["asn1"]["DERSet"] = function (_0x58d390) {
    _0x2f02a5["asn1"]["DERSet"]["superclass"]["constructor"]["call"](this, _0x58d390);
    this["hT"] = "31";
    this["sortFlag"] = !0;
    this["getFreshValueHex"] = function () {
      for (var _0x58d390 = new Array(), _0x4efcb8 = 0; _0x4efcb8 < this["asn1Array"]["length"]; _0x4efcb8++) _0x58d390["push"](this["asn1Array"][_0x4efcb8]["getEncodedHex"]());
      return 1 == this["sortFlag"] && _0x58d390["sort"](), this["hV"] = _0x58d390["join"](""), this["hV"];
    };
    void 0 !== _0x58d390 && void 0 !== _0x58d390["sortflag"] && 0 == _0x58d390["sortflag"] && (this["sortFlag"] = !1);
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERSet"], _0x2f02a5["asn1"]["DERAbstractStructured"]);
  _0x2f02a5["asn1"]["DERTaggedObject"] = function (_0x1351db) {
    _0x2f02a5["asn1"]["DERTaggedObject"]["superclass"]["constructor"]["call"](this);
    this["hT"] = "a0";
    this["hV"] = "";
    this["isExplicit"] = !0;
    this["asn1Object"] = null;
    this["setASN1Object"] = function (_0x244761, _0x5b13dd, _0x433a39) {
      this["hT"] = _0x5b13dd;
      this["isExplicit"] = _0x244761;
      this["asn1Object"] = _0x433a39;
      if (this["isExplicit"]) {
        this["hV"] = this["asn1Object"]["getEncodedHex"](), this["hTLV"] = null, this["isModified"] = !0;
      } else {
        this["hV"] = null, this["hTLV"] = _0x433a39["getEncodedHex"](), this["hTLV"] = this["hTLV"]["replace"](/^../, _0x5b13dd), this["isModified"] = !1;
      }
    };
    this["getFreshValueHex"] = function () {
      return this["hV"];
    };
    void 0 !== _0x1351db && (void 0 !== _0x1351db["tag"] && (this["hT"] = _0x1351db["tag"]), void 0 !== _0x1351db["explicit"] && (this["isExplicit"] = _0x1351db["explicit"]), void 0 !== _0x1351db["obj"] && (this["asn1Object"] = _0x1351db["obj"], this["setASN1Object"](this["isExplicit"], this["hT"], this["asn1Object"])));
  };
  _0x1bbd98["lang"]["extend"](_0x2f02a5["asn1"]["DERTaggedObject"], _0x2f02a5["asn1"]["ASN1Object"]);
  var _0x41a492 = function (_0x33a578) {
      function _0xb3f6eb(_0x24d672) {
        var _0xe964c2 = _0x33a578["call"](this) || this;
        return _0x24d672 && ("string" == typeof _0x24d672 ? _0xe964c2["parseKey"](_0x24d672) : (_0xb3f6eb["hasPrivateKeyProperty"](_0x24d672) || _0xb3f6eb["hasPublicKeyProperty"](_0x24d672)) && _0xe964c2["parsePropertiesFrom"](_0x24d672)), _0xe964c2;
      }
      return function (_0xddeca, _0x357b27) {
        function _0x814063() {
          this["constructor"] = _0xddeca;
        }
        _0x556c8d(_0xddeca, _0x357b27);
        if (null === _0x357b27) {
          _0xddeca["prototype"] = Object["create"](_0x357b27);
        } else {
          _0xddeca["prototype"] = (_0x814063["prototype"] = _0x357b27["prototype"], new _0x814063());
        }
      }(_0xb3f6eb, _0x33a578), _0xb3f6eb["prototype"]["parseKey"] = function (_0x2abee0) {
        try {
          var _0x53ddf0 = 0,
            _0x286e7e = 0,
            _0x39332d = /^\s*(?:[0-9A-Fa-f][0-9A-Fa-f]\s*)+$/["test"](_0x2abee0) ? function (_0x4274ca) {
              var _0xa344d5;
              if (void 0 === _0x465910) {
                var _0x1e7d98 = "0123456789ABCDEF",
                  _0x2d9fcf = " \f\n\r\t \u2028\u2029";
                for (_0x465910 = {}, _0xa344d5 = 0; _0xa344d5 < 16; ++_0xa344d5) _0x465910[_0x1e7d98["charAt"](_0xa344d5)] = _0xa344d5;
                for (_0x1e7d98 = _0x1e7d98["toLowerCase"](), _0xa344d5 = 10; _0xa344d5 < 16; ++_0xa344d5) _0x465910[_0x1e7d98["charAt"](_0xa344d5)] = _0xa344d5;
                for (_0xa344d5 = 0; _0xa344d5 < _0x2d9fcf["length"]; ++_0xa344d5) _0x465910[_0x2d9fcf["charAt"](_0xa344d5)] = -1;
              }
              var _0x3ee697 = [],
                _0x13a5fe = 0,
                _0x111111 = 0;
              for (_0xa344d5 = 0; _0xa344d5 < _0x4274ca["length"]; ++_0xa344d5) {
                var _0x5947c6 = _0x4274ca["charAt"](_0xa344d5);
                if ("=" == _0x5947c6) break;
                if (-1 != (_0x5947c6 = _0x465910[_0x5947c6])) {
                  if (void 0 === _0x5947c6) throw new Error("Illegal character at offset " + _0xa344d5);
                  _0x13a5fe |= _0x5947c6;
                  if (2 <= ++_0x111111) {
                    _0x3ee697[_0x3ee697["length"]] = _0x13a5fe, _0x111111 = _0x13a5fe = 0;
                  } else {
                    _0x13a5fe <<= 4;
                  }
                }
              }
              if (_0x111111) throw new Error("Hex encoding incomplete: 4 bits missing");
              return _0x3ee697;
            }(_0x2abee0) : _0x5a02a1["unarmor"](_0x2abee0),
            _0x264f38 = _0x408860["decode"](_0x39332d);
          if (3 === _0x264f38["sub"]["length"] && (_0x264f38 = _0x264f38["sub"][2]["sub"][0]), 9 === _0x264f38["sub"]["length"]) {
            _0x53ddf0 = _0x264f38["sub"][1]["getHexStringValue"]();
            this["n"] = _0x5baf06(_0x53ddf0, 16);
            _0x286e7e = _0x264f38["sub"][2]["getHexStringValue"]();
            this["e"] = parseInt(_0x286e7e, 16);
            var _0x3efe14 = _0x264f38["sub"][3]["getHexStringValue"]();
            this["d"] = _0x5baf06(_0x3efe14, 16);
            var _0x5a91df = _0x264f38["sub"][4]["getHexStringValue"]();
            this["p"] = _0x5baf06(_0x5a91df, 16);
            var _0x356d93 = _0x264f38["sub"][5]["getHexStringValue"]();
            this["q"] = _0x5baf06(_0x356d93, 16);
            var _0x5486e5 = _0x264f38["sub"][6]["getHexStringValue"]();
            this["dmp1"] = _0x5baf06(_0x5486e5, 16);
            var _0x21f4cd = _0x264f38["sub"][7]["getHexStringValue"]();
            this["dmq1"] = _0x5baf06(_0x21f4cd, 16);
            var _0x2d2630 = _0x264f38["sub"][8]["getHexStringValue"]();
            this["coeff"] = _0x5baf06(_0x2d2630, 16);
          } else {
            if (2 !== _0x264f38["sub"]["length"]) return !1;
            var _0x277925 = _0x264f38["sub"][1]["sub"][0];
            _0x53ddf0 = _0x277925["sub"][0]["getHexStringValue"]();
            this["n"] = _0x5baf06(_0x53ddf0, 16);
            _0x286e7e = _0x277925["sub"][1]["getHexStringValue"]();
            this["e"] = parseInt(_0x286e7e, 16);
          }
          return !0;
        } catch (_0xb7285e) {
          return !1;
        }
      }, _0xb3f6eb["prototype"]["getPrivateBaseKey"] = function () {
        var _0x33a578 = {
          "array": [new _0x2f02a5["asn1"]["DERInteger"]({
            "int": 0
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["n"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "int": this["e"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["d"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["p"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["q"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["dmp1"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["dmq1"]
          }), new _0x2f02a5["asn1"]["DERInteger"]({
            "bigint": this["coeff"]
          })]
        };
        return new _0x2f02a5["asn1"]["DERSequence"](_0x33a578)["getEncodedHex"]();
      }, _0xb3f6eb["prototype"]["getPrivateBaseKeyB64"] = function () {
        return _0x125db2(this["getPrivateBaseKey"]());
      }, _0xb3f6eb["prototype"]["getPublicBaseKey"] = function () {
        var _0x33a578 = new _0x2f02a5["asn1"]["DERSequence"]({
            "array": [new _0x2f02a5["asn1"]["DERObjectIdentifier"]({
              "oid": "1.2.840.113549.1.1.1"
            }), new _0x2f02a5["asn1"]["DERNull"]()]
          }),
          _0x48af93 = new _0x2f02a5["asn1"]["DERSequence"]({
            "array": [new _0x2f02a5["asn1"]["DERInteger"]({
              "bigint": this["n"]
            }), new _0x2f02a5["asn1"]["DERInteger"]({
              "int": this["e"]
            })]
          }),
          _0x25edbb = new _0x2f02a5["asn1"]["DERBitString"]({
            "hex": "00" + _0x48af93["getEncodedHex"]()
          });
        return new _0x2f02a5["asn1"]["DERSequence"]({
          "array": [_0x33a578, _0x25edbb]
        })["getEncodedHex"]();
      }, _0xb3f6eb["prototype"]["getPublicBaseKeyB64"] = function () {
        return _0x125db2(this["getPublicBaseKey"]());
      }, _0xb3f6eb["wordwrap"] = function (_0x469c11, _0x2f5bfa) {
        if (!_0x469c11) return _0x469c11;
        var _0x37b50e = "(.{1," + (_0x2f5bfa = _0x2f5bfa || 64) + "})( +|$\n?)|(.{1," + _0x2f5bfa + "})";
        return _0x469c11["match"](RegExp(_0x37b50e, "g"))["join"]("\n");
      }, _0xb3f6eb["prototype"]["getPrivateKey"] = function () {
        var _0x33a578 = "-----BEGIN RSA PRIVATE KEY-----\n";
        return (_0x33a578 += _0xb3f6eb["wordwrap"](this["getPrivateBaseKeyB64"]()) + "\n") + "-----END RSA PRIVATE KEY-----";
      }, _0xb3f6eb["prototype"]["getPublicKey"] = function () {
        var _0x33a578 = "-----BEGIN PUBLIC KEY-----\n";
        return (_0x33a578 += _0xb3f6eb["wordwrap"](this["getPublicBaseKeyB64"]()) + "\n") + "-----END PUBLIC KEY-----";
      }, _0xb3f6eb["hasPublicKeyProperty"] = function (_0x32bb2b) {
        return (_0x32bb2b = _0x32bb2b || {})["hasOwnProperty"]("n") && _0x32bb2b["hasOwnProperty"]("e");
      }, _0xb3f6eb["hasPrivateKeyProperty"] = function (_0x101d27) {
        return (_0x101d27 = _0x101d27 || {})["hasOwnProperty"]("n") && _0x101d27["hasOwnProperty"]("e") && _0x101d27["hasOwnProperty"]("d") && _0x101d27["hasOwnProperty"]("p") && _0x101d27["hasOwnProperty"]("q") && _0x101d27["hasOwnProperty"]("dmp1") && _0x101d27["hasOwnProperty"]("dmq1") && _0x101d27["hasOwnProperty"]("coeff");
      }, _0xb3f6eb["prototype"]["parsePropertiesFrom"] = function (_0x19affd) {
        this["n"] = _0x19affd["n"];
        this["e"] = _0x19affd["e"];
        _0x19affd["hasOwnProperty"]("d") && (this["d"] = _0x19affd["d"], this["p"] = _0x19affd["p"], this["q"] = _0x19affd["q"], this["dmp1"] = _0x19affd["dmp1"], this["dmq1"] = _0x19affd["dmq1"], this["coeff"] = _0x19affd["coeff"]);
      }, _0xb3f6eb;
    }(_0x56aa3c),
    _0x4b4d2c = function () {
      function _0x2f9c92(_0x19a9fb) {
        _0x19a9fb = _0x19a9fb || {};
        this["default_key_size"] = parseInt(_0x19a9fb["default_key_size"], 10) || 1024;
        this["default_public_exponent"] = _0x19a9fb["default_public_exponent"] || "010001";
        this["log"] = _0x19a9fb["log"] || !1;
        this["key"] = null;
      }
      return _0x2f9c92["prototype"]["setKey"] = function (_0x1faf93) {
        this["log"] && this["key"] && console["warn"]("A key was already set, overriding existing.");
        this["key"] = new _0x41a492(_0x1faf93);
      }, _0x2f9c92["prototype"]["setPrivateKey"] = function (_0x40b233) {
        this["setKey"](_0x40b233);
      }, _0x2f9c92["prototype"]["setPublicKey"] = function (_0x3346c5) {
        this["setKey"](_0x3346c5);
      }, _0x2f9c92["prototype"]["decrypt"] = function (_0x5bbfaf) {
        try {
          return this["getKey"]()["decrypt"](_0x5c2128(_0x5bbfaf));
        } catch (_0x47bbe7) {
          return !1;
        }
      }, _0x2f9c92["prototype"]["encrypt"] = function (_0x4c98fe) {
        try {
          return _0x125db2(this["getKey"]()["encrypt"](_0x4c98fe));
        } catch (_0x206989) {
          return !1;
        }
      }, _0x2f9c92["prototype"]["sign"] = function (_0x42cd43, _0x32d28f, _0x2a76ad) {
        try {
          return _0x125db2(this["getKey"]()["sign"](_0x42cd43, _0x32d28f, _0x2a76ad));
        } catch (_0x40c544) {
          return !1;
        }
      }, _0x2f9c92["prototype"]["verify"] = function (_0x4d5b24, _0x78dca0, _0x9fcf29) {
        try {
          return this["getKey"]()["verify"](_0x4d5b24, _0x5c2128(_0x78dca0), _0x9fcf29);
        } catch (_0x80a5ee) {
          return !1;
        }
      }, _0x2f9c92["prototype"]["getKey"] = function (_0x3bdda3) {
        if (!this["key"]) {
          if (this["key"] = new _0x41a492(), _0x3bdda3 && "[object Function]" === {}["toString"]["call"](_0x3bdda3)) return void this["key"]["generateAsync"](this["default_key_size"], this["default_public_exponent"], _0x3bdda3);
          this["key"]["generate"](this["default_key_size"], this["default_public_exponent"]);
        }
        return this["key"];
      }, _0x2f9c92["prototype"]["getPrivateKey"] = function () {
        return this["getKey"]()["getPrivateKey"]();
      }, _0x2f9c92["prototype"]["getPrivateKeyB64"] = function () {
        return this["getKey"]()["getPrivateBaseKeyB64"]();
      }, _0x2f9c92["prototype"]["getPublicKey"] = function () {
        return this["getKey"]()["getPublicKey"]();
      }, _0x2f9c92["prototype"]["getPublicKeyB64"] = function () {
        return this["getKey"]()["getPublicBaseKeyB64"]();
      }, _0x2f9c92["version"] = "3.0.0-rc.1", _0x2f9c92;
    }();
  window["JSEncrypt"] = _0x4b4d2c;
  _0x20544c["JSEncrypt"] = _0x4b4d2c;
  _0x20544c["default"] = _0x4b4d2c;
  Object["defineProperty"](_0x20544c, "__esModule", {
    "value": !0
  });
});
function _0x4f6d79(_0x50f9fa) {
  const _0x506402 = "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5GVku07yXCndaMS1evPIPyWwhbdWMVRqL4qg4OsKbzyTGmV4YkG8H0hwwrFLuPhqC5tL136aaizuL/lN5DRRbePct6syILOLLCBJ5J5rQyGr00l1zQvdNKYp4tT5EFlqw8tlPkibcsd5Ecc8sTYa77HxNeIa6DRuObC5H9t85ALJyDVZC3Y4ES/u61Q7LDnB3kG9MnXJsJiQxm1pLkE7Zfxy29d5JaXbbfwhCDSjE4+dUQoq2MVIt2qVjZSo5Hd/bAFGU1Lmc7GkFeLiLjNTOfECF52ms/dks92Wx/glfRuK4h/fcxtGB4Q2VXu5k68e/2uojs6jnFsMKVe+FVUDkQIDAQAB";
  const _0xc7daa3 = new JSEncrypt();
  _0xc7daa3["setPublicKey"](_0x506402);
  return encodeURIComponent(_0xc7daa3["encrypt"](_0x50f9fa));
}
window["decrypt"] = _0x4f6d79;
function getResult(timestamp){
  for (var m = 0x1; m <= 3; m++) {
	var res = decrypt(timestamp) + 'r';
  }
  var cookie = (m - 0x1)['toString']() + res;
  return cookie
}

```

事后角度看这个题真不难，卡壳的地方在于没有用对解混淆的工具，猿人学自己的解混淆的工具针对这个题的代码做了一些调整，所以并不是那么好用的，另外意识到的一点是在js逆向中修补环境是很重要的，浏览器的运行环境和node是有很多差异的，也会有写代码在此设坑，最后是自行简化代码时要做到不遗漏，拿不准的都去控制台执行一下，能换的都给换掉

----

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[Python反反爬之动态字体---随风漂移](https://syjun.vip/archives/283.html)

[Python反反爬之图文点击---生僻字验证码](https://syjun.vip/archives/284.html)

[【使用AST还原JS混淆代码应用】 猿人学爬虫攻防赛 第九题详解](https://www.52pojie.cn/thread-1340226-1-1.html)

{{% /spoiler %}}