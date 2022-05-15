---
title: "DNS域传送漏洞学习"
slug: "dns-zone-transfer-study-note"
description: "对DNS服务器的不当配置 导致域内信息泄露"
date: 2022-05-06T16:52:58+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["PENTEST"]
draft: false
toc: true
---

DNS服务器提供域名解析的基础性服务，可以细分为主服务器、备份服务器、缓存服务器，备份服务器会从主服务器中拷贝数据来更新自己的数据库，而这个同步的过程中就需要用到DNS域传送，一旦配置不当，就可能会导致任何匿名用户都可以获取DNS服务器某一域的所有记录，泄露重要信息

## 漏洞成因

在域传送中关键的配置项有这两个

```
allow-transfer {ipaddress;}; 通过ip限制可进行域传送的服务器
allow-transfer { key transfer; }; 通过key限制可进行域传送的服务器
```

有两种设置方式，options或是zone，默认优先级为zone 如果为空则遵循options，示例

```
options {
listen-on { 1.1.1.1; };
listen-on-v6 { any; };
directory "/bind";
pid-file "/bind/run/pid";
dump-file "/bind/data/named_dump.db";
statistics-file "/bind/data/named.stats";

    allow-transfer  { any; };
    allow-query    {any;};


};
```

```
zone "wooyun.org" {
type master;
file "/bind/etc/wooyun.org.conf";
allow-transfer {any;};
};
```

如果没有手动添加`allow-transfer`来限制获取记录的来源，就会存在DNS域传送漏洞

## 检测方式

### dig / linux

以vulhub为例，以常规的8.8.8.8作为DNS服务器进行解析，如果发送axfr的请求会failed

![image-20220506160422503](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220506160422503.png)

如果使用存在漏洞的DNS服务器进行查询，则会返回域内所有的字域、A记录和CNAME等信息

![image-20220506160602721](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220506160602721.png)

### nslookup / windows&linux

正常情况：

![image-20220506163501741](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220506163501741.png)

存在漏洞的情况（linux示例）：

![image-20220506164756031](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220506164756031.png)

### Nmap

```bash
nmap -script dns-zone-transfer -script-args dns-zone-transfer.domain=vulhub.org -p 53 -Pn 172.27.0.2
```

### dnsrecon

```bash
dnsrecon -d example.com
```

### dnsenum

```bash
dnsenum example.com
```

----

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[浅析DNS域传送漏洞](https://www.mi1k7ea.com/2021/04/03/%E6%B5%85%E6%9E%90DNS%E5%9F%9F%E4%BC%A0%E9%80%81%E6%BC%8F%E6%B4%9E/)

[DNS域传送信息泄露](http://drops.xmd5.com/static/drops/papers-64.html)

[vulhub复现环境](https://github.com/vulhub/vulhub/blob/master/dns/dns-zone-transfer/README.zh-cn.md)

{{% /spoiler %}}

