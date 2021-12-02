---
title: "PHP远程debug环境配置"
slug: "php-remote-debug-env-init"
description: "坑点多多，痛苦面具，目前非最优解，长期更新"
date: 2021-11-25T18:44:34+08:00
categories: ["LTS", "环境配置"]
series: ["PHP代码审计"]
tags: ["debug", "PHP"]
draft: false
toc: true
---

首先，这里的情况与一般的本地调试localhost不同，是本地debug远程主机的docke内的php项目，也就是这里的laravel；可能有的人觉得多此一举，直接在本地搭建项目、运行、debug一气呵成，速度还快，何必到远程跑一通？那我只能说too young，年轻人还是要有点长远的打算（

一切的起因是我想复现cve-2021-3129，顺便亲自动手跟一跟&挖一挖laravel框架反序列化的链子，结果代码审计之前先被环境配置困住了；被这个糟心的调试从周二下午折磨到周四的凌晨，几乎是把能踩的坑踩了个遍（痛苦面具），直到现在的解决方案也不能算是最优解，不过是能正常稳定的跑起来了，次优解；感谢portainer让我能轻松创建docker

把详细的步骤记录到这篇博客里，希望可以帮助到看到的人~欢迎友好交流！

------

先说下本次配置的环境整体需求吧：

- 远程vps的docker(FROM php:7.4-apache): 需开放web页面端口 + ssh端口（无需挂载目录
- laravel 8.4.2, laravel/framework 8.22.1, facade/ignition 2.5.1
- 本地ide: PHPStorm + 一个映射到公网ip的供调试连接使用的端口（后面会解释原因）

除了最后的laravel之外就都是调php的通用流程啦，让我们开始吧~

## 远程docker&laravel的环境搭建

以php:7.4-apache镜像作为基础进行创建，我这里用的是portainer（别问为什么不写dockerfile，问就是还在学着怎么写（一定不鸽 近期会更新到这里的

![image-20211125164513693](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125164513693.png)

```bash
apt update
apt-get install -y ssh git unzip
vim /etc/ssh/sshd_config	# 开启对应选项 比如允许root连接什么的 不用我细嗦 各位带手子们都应该懂
pecl install xdebug
touch /var/log/php/xdebug_remote.log	# 如果需要log的话要提前创建好噢
vim /usr/local/etc/php/php.ini	# 将下面框中内容粘到末尾即可
```

```ini
[Xdebug]
; 这一项安装好之后会有提示
zend_extension="/usr/local/lib/php/extensions/no-debug-non-zts-20190902/xdebug.so"
xdebug.mode=debug
xdebug.client_host=your_vps
xdebug.client_port=port
xdebug.idekey=laravel
xdebug.remote_handler="dbgp"
xdebug.log="/var/log/php/xdebug_remote.log"
```

此处安装版本为xdebug3，与xdebug2的配置语法有一些出入，具体更新的内容详见[更新文档](https://xdebug.org/docs/upgrade_guide)，安装好xdebug后执行`php -v`命令会有显示版本

![image-20211125164949111](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125164949111.png)

```bash
wget https://github.com/composer/composer/releases/download/2.1.12/composer.phar
mv composer.phar /usr/local/bin/composer
chmod +x /usr/local/bin/composer
```

```bash
cd /var/www
composer create-project laravel/laravel . "v8.4.2"
vim composer.json	# 修改内容见下
composer update
mv public html
chmod -R 777 /var/www
```

```ini
"laravel/framework": "8.22.1"
"facade/ignition": "2.5.1"
```

一个坑点：即使指定了[laravel/laravel](laravel/laravel)版本，composer创建默认会使用最新的[laravel/framework](laravel/framework)版本&相关组件依赖（比如现在是8.73.2），由于我们需要复现的cve有特殊的laravel组件版本需求，，所以我们需要单独对composer.json进行修改，然后执行`composer update`进行更新

一切顺利的话访问web页面就可以看到laravel啦

![image-20211125165355843](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125165355843.png)

## 与本地的PHPStorm连接起来进行debug

开始之前先自问自答一下：为什么调试要一个能映射到公网的端口？

PHPStorm中集成了遵循DBGp协议的Xdebug插件，当开启监听时会监听本地指定端口，当访问指定server的web页面并带有含有ide_key的特定cookie参数之后，Xdebug会接收到这个参数并开启debug，通过DBGp协议向php.ini中指定的ip&port发送请求，当这个port指向我们ide在监听的端口时，就会建立起Xdebug和ide的连接（使用DBGp协议通讯），随后它们之间进行数据交互，如此反复；更详细的内容请参见->[6.3 IDE to debugger engine communications](https://xdebug.org/docs/dbgp#id32)

这下就好理解了叭？也就是说，需要一个与远程代码运行环境处在同一个网段下的端口（公网的更好）；接下来就是配置啦

### 内网穿透

内网穿透我使用的是nps，配置过程可以参考[官方文档](https://ehang-io.github.io/nps/#/run)，由于我这里配的比较早 这部分跳过

### PHPStorm的设置有一堆

把docker中/www目录打包拖下来，用PHPStorm打开

#### 设置Debug

![image-20211125172830540](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125172830540.png)

20007是本地端口，被映射到公网ip

![image-20211125172903465](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125172903465.png)

这里的20007是公网端口，代理本地的20007

如果我讲清楚前面原理的话，大家应该能理解这两个地方的端口设置二选一即可

#### 设置Deployment

用SFTP与docker进行ssh连接，文件映射

![image-20211125174439189](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125174439189.png)

![image-20211125174449223](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125174449223.png)

#### 设置远程php解释器

选择ssh模式，配一下账号密码端口就行（至于为什么不用docker/docker-compose，之后会说），这里的Path mappings直接就是Deployment中配好的，可以单独再加，但是没必要

![image-20211125173731219](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125173731219.png)

![image-20211125174004041](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125174004041.png)

#### 设置Server

注意高亮地方的设置，是web页面的端口，也要设置好本地的路径映射

![image-20211125173454167](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125173454167.png)

### 验证一下联通情况

![image-20211125175835232](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125175835232.png)

好耶~

### 浏览器插件Xdebug helper

设置Debug Configuration，选PHP Remote Debug模式

![image-20211125175154353](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125175154353.png)

安装插件，设置ide_key，开启小虫子

![image-20211125180035074](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125180035074.png)

![image-20211125180424620](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125180424620.png)

## 终于完了

随便打个断点，开启本地监听

![image-20211125180223037](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125180223037.png)

刷新页面

![image-20211125180453865](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211125180453865.png)

芜湖起飞！

## ***待完善的地方

- 配置远程php解释器可以使用docker/docker-compose，这两个都需要单独挂载关键目录出来，配置上与ssh的区别还是有点大的，也需要注意docker-compose文件的编写，先扔一个参考资料->[Setting up PhpStorm with Xdebug for local development on Docker ... natively and via Deployment Configuration.](https://www.pascallandau.com/blog/setup-phpstorm-with-xdebug-on-docker/#setup-the-docker-containers)
- 关于dockerfile/docker-compose文件的编写，我还不是很熟练，跟不用说这样的php项目了，同样先扔一个参考资料->[Structuring the Docker setup for PHP Projects ... folder structure, Dockerfile templates and general fundamentals](https://www.pascallandau.com/blog/structuring-the-docker-setup-for-php-projects/)
- PHPStorm有内置的docker管理功能，可以创建镜像->配置debug一步到位

------

虽然踩了很多坑，但是也学到了不少东西，希望能多少帮助一些人少走弯路（

限于篇幅原因，代码审计的部分就放到后面啦~

最近更新的不太勤快，我先自我批评TAT，已经很菜了不能再不努力啊啊啊啊啊啊啊啊