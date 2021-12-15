---
title: "关于include/require(_once)的易混淆点"
slug: "smth-about-include-require-in-php"
description: "感觉是一个易混淆的地方，很适合出题"
date: 2021-12-13T01:12:22+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["PHP", "文件包含"]
draft: false
toc: true
---

之前做题遇到后查了一下相关文章，觉得很适合出题，但是感觉不能为了考而考，也就失去了CTF的意义，所以还不如先发出来这一篇

~~（别骂了 就是懒~~

## 关于`include/require(_once)()`

这几个函数应该都不陌生（还应该再加上include_path），干的是文件包含的事；先按照给定参数寻找路径（当定义了路径则会完全忽略include_path），如果只有文件名则按照include_path的指定目录寻找，如果还没找到则会在调用脚本文件所在的目录和当前工作目录下寻找；下面说几个小点

首先是require和include的区别在于require在出错时会产生E_COMPILE_ERROR级别的错误，脚本将中止运行，而include只会产生警告E_WARNING，不影响继续顶着错误继续执行脚本（做题很常见这个 无数的报错下面有正常的回显）

另一个区别在于require是无条件包含函数，即程序执行前就会将文件内容以文本替换的形式引入（所以这份文件只被处理一次），让它变为php脚本的一部分（所以通常放在php文件的最前面）；include则是有条件包含函数，用到时才被加载，且每次执行文件时都要进行读取和评估

当php遇到require/include(_once)函数时，如果给定的参数是个绝对路径则直接引入，但如果给的是相对路径，则要进入另外的逻辑（经过多次调用，宏展开后进入\_php\_stream\_fopen\_with\_path）寻找此文件；在\_php\_stream\_fopen\_with\_path会根据include\_path和当前执行文件的path组成一个待选的目录列表，然后从列表头部开始根据DEFAULT\_DIR\_SEPARATOR（linux下默认为`:` win为`;`）取出路径，然后拼上文件名，如果成功则返回，失败则继续下一个待选择路径（<u>所以如果有重名的就好玩了</u>）

## 几个好玩的小栗子

这里举一个小栗子，文件结构和内容如下（注意这里include_path的写法!!!不仅有path_to_subdir还有`:`分隔符前面代表的current_dir!!!）

![image-20211024095717109](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024095717109.png)

![image-20211024095830345](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024095830345.png)

在不同的情况下运行看看效果

![image-20211024100007587](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024100007587.png)

在root目录下执行时，直接根据1.php的include_path寻找2.php，而2.php中并没有设置include_path，那么路径就变成了列表中第一个待选择路径 即`"."`，而当前工作目录是root，所以就是/root/3.php的输出

换到subdir目录下执行1.php时，列表中第一个待选择路径`"."`变为了subdir，自然输出变为了/root/subdir/3.php的输出

现在我们把include_path仅保留path_to_subdir，继续上面的指令运行一下

![image-20211024190934131](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024190934131.png)

那不管当前路径是什么都会得到subdir的输出

再进一步，在2.php中半道清空include_path会怎么样呢

![image-20211024191919122](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024191919122.png)

我们发现跟上面结果一样，但是这里的逻辑却有差异：2.php中清空include_path后再包含文件将会自动变为current_path的路径，而2.php在/root/subdir下，所以回显和2.php的路径保持一致

再再进一步

![image-20211024192931075](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024192931075.png)

![image-20211024193020147](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211024193020147.png)

诶，可以看到由于这里两次都用到了相对目录的`"./"`进行引用，所以在/root目录下运行很顺畅，而/root/subdir目录下由于不会再存在./subdir/2.php，所以报错退出

## 使用注意

- 在使用include_path和相对路径时会影响性能，所以最好用绝对路径
- 目录相对路径的basedir永远是当前工作路径
- 可以通过`__DIR__`来获得绝对路径
- 建议少使用`include/require_once`

------

*写的不全面，还有一部分需要补充 先鸽了，遇到题或者具体的应用场景我补充上来

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[深入理解PHP之require/include顺序](https://www.laruence.com/2010/05/04/1450.html)

{{% /spoiler %}}