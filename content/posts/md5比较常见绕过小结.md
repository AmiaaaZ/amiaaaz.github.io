---
title: "md5比较常见绕过小结"
slug: "bypass-md5-or-sha1-compare"
description: "摁水一篇"
date: 2021-12-03T00:24:56+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["md5"]
draft: false
toc: true
---

> **<u>写在开头：</u>**
>
> **<u>别纠结</u>**
>
> **<u>左侧的符号反转&标题是中文引号&有些符号消失</u>**
>
> **<u>的问题</u>**
>
> **<u>我也很无语</u>**
>
> **寄。**

------

- ```
  md5(array()) = md5(NULL) = d41d8cd98f00b204e9800998ecf8427e
  ```

- sha1绕过和md5绕过方式是一样的

## md5($a)==md5($b) && $a!=$b

数组绕过

```
?a[]=1&b[]=2
```

0e绕过

```
IHKFRNS -> 0e256160682445802696926137988570
QLTHNDT -> 0e405967825401955372549139051580
QNKCDZO -> 0e830400451993494058024219903391
3908336290 -> 0e807624498959190415881248245271
4011627063 -> 0e485805687034439905938362701775
4775635065 -> 0e998212089946640967599450361168
0e215962017 -> 0e291242476940776845150308577824
aabg7XSs -> 0e087386482136013740957780965295
aabC9RqS -> 0e041022518165728065344349536299
```

```
0e251288019 -> 0e874956163641961271069404332409
240610708 -> 0e462097431906509019562988736854
```

md5碰撞

```
a=%D89%A4%FD%14%EC%0EL%1A%FEG%ED%5B%D0%C0%7D%CAh%16%B4%DFl%08Z%FA%1DA%05i%29%C4%FF%80%11%14%E8jk5%0DK%DAa%FC%2B%DC%9F%95ab%D2%09P%A1%5D%12%3B%1ETZ%AA%92%16y%29%CC%7DV%3A%FF%B8e%7FK%D6%CD%1D%DF/a%DE%27%29%EF%08%FC%C0%15%D1%1B%14%C1LYy%B2%F9%88%DF%E2%5B%9E%7D%04c%B1%B0%AFj%1E%7Ch%B0%96%A7%E5U%EBn1q%CA%D0%8B%C7%1BSP
b=%D89%A4%FD%14%EC%0EL%1A%FEG%ED%5B%D0%C0%7D%CAh%164%DFl%08Z%FA%1DA%05i%29%C4%FF%80%11%14%E8jk5%0DK%DAa%FC%2B%5C%A0%95ab%D2%09P%A1%5D%12%3B%1ET%DA%AA%92%16y%29%CC%7DV%3A%FF%B8e%7FK%D6%CD%1D%DF/a%DE%27%29o%08%FC%C0%15%D1%1B%14%C1LYy%B2%F9%88%DF%E2%5B%9E%7D%04c%B1%B0%AFj%9E%7Bh%B0%96%A7%E5U%EBn1q%CA%D0%0B%C7%1BSP
```

## md5($a)==md5(md5($b)) && $a!=$b

数组绕过

```
a=&b[]=1
```

## md5((string)$\_GET['a'])==md5(md5((string)$_GET['b'])) && $a!=$b

0e绕过，一个是正常0e，一个是第二次md5后还为0e的奇葩值

```
aawBzC
aabsbm9
aaaabGG5T
aaaabKGVH
```

## md5($a)==md5($b) && !ctype_alpha($a) && !is_numeric($b)

0e绕过，一个纯数字一个纯字母

```
240610708 -> 0e462097431906509019562988736854
QNKCDZO -> 0e830400451993494058024219903391
```

## md5($a)===md5($b)  &&  $a!=$b

数组绕过

```
?a[]=1&b[]=2
```

## md5($a)===md5(md5($b)) && $a!=$b

数组绕过

```
?a[]=1&b[]=2
```

## md5($a)===md5($b)  &&  $a!==$b

数组绕过

```
?a[]=1&b[]=2
```

## $a!=hash("md4",$a)

0e纯数字绕过

```php
<?php
for ($i = 0; ; $i++) {
    $r = "0e" . $i;
    $md4 = hash("md4", $r);
    if (preg_match("/^0e[0-9]*$/", $md4)) {
        echo ("md4加密前:".$r)."\n";
        echo("md4加密后：".$md4);
        break;
    }
}
```

```
0e251288019 -> 0e874956163641961271069404332409
```

## md5($a)===md5($b) && $a!==$b && strlen($a)<=3 && strlen($b)<=3

INF表示无穷大，而NAN表示一个在浮点数运算中未定义或不可表述的值；除了与True之外，拿NAN与其他任何值进行松散比较或者严格比较返回结果都是FALSE
因为他们都是不确定的值，所以在与自身做比较时，会返回false

![image-20211030175015724](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211030175015724.png)

INF和NAN绕过

```
a=INF&b=INF
```

```
a=NAN&b=NAN
```

## ($this->syc != $this->lover) && (md5($this->syc) === md5($this->lover)) && (sha1($this->syc)=== sha1($this->lover))

做题遇到的

用原生类`Error`（php7）或`Exception`（php5 or 7），它有`__toString`方法，被触发后会以字符串形式输出当前保存情况，包括错误信息和当前报错的行号，而跟传入的参数没有关系；所以说可以构造两个类的实例，它们行号相同（被`__toString`调用后输出信息一样），但是本身不相同（传入参数不等）

```php
$str = "?><?=include~".urldecode(urlencode(~'/flag'))."?>";	// 这个不重要 是那个题的payload
$a=new Error($str,1);$b=new Error($str,2);
$c = new SYCLOVER();
$c->syc = $a;
$c->lover = $b;
```

注意$a和$b写到一行

## $this->token === $this->token_flag

指针取地址绕过

```
$F->token=&$F->token_flag;
```

