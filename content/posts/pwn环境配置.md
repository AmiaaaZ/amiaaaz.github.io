---
title: "pwn环境配置"
slug: "pwn-env-init"
description: "稍微横向点一点技能树，不管学成什么样子，环境先给搭好（"
date: 2021-10-28T00:41:15+08:00
categories: ["环境配置"]
series: ["PWN-从入门到入坟"]
tags: ["pwn"]
draft: false
toc: false
---

以下为Ubutun16.04&py2的安装过程，其他版本命令几乎相同的~（Ubuntu20&py3的我也装了一个，毕竟py3是主流，技术总是向前发展的嘛~）

————毕竟我是以初学者的视角来准备pwn的环境，肯定还有很多不全面&想不到的地方，之后用到了会回来补上的~

安装pip2&pip3&ipython

```
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
sudo python get-pip.py
# sudo apt-get install python3-pip
curl https://bootstrap.pypa.io/pip/3.5/get-pip.py -o get-pip.py
sudo python3 get-pip.py
sudo apt-get install ipython
```

安装py2的pwntools

```
pip2 install pathlib2   # py3无需此条
pip2 install pwntools
```

安装GDB&插件

```
sudo apt-get install git
git clone https://github.com/pwndbg/pwndbg.git
./setup.sh
```

```
# 使用peda-heap
git clone https://github.com/Mipu94/peda-heap.git
echo "source ~/peda-heap/peda.py" > ~/.gdbinit
# 使用peda
git clone https://github.com/longld/peda.git
echo "source ~/GDB-Plugins/peda/peda.py" > ~/.gdbinit
# 使用gef
git clone https://github.com/hugsy/gef.git
echo "source ~/gef/gef.py" > ~/.gdbinit
# 使用pwndbg
echo "source ~/pwndbg/gdbinit.py" > ~/.gdbinit
# 自动切换脚本 https://blog.csdn.net/aptx4869_li/article/details/81566541
```

安装one_gadget

```
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get install ruby2.7 ruby2.7-dev
sudo apt-get install gem
sudo gem install one_gadget
```

安装32位库

```
sudo apt-get install lib32ncurses5
sudo apt-get install lib32z1
```

安装checksec

```
sudo apt instal checksec
```

安装seccomp-tools

```
sudo gem install seccomp-tools
```

安装zshell&插件

```
sudo apt-get install zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
echo "source ${(q-)PWD}/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc
source ./zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
# zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh
```

```
# 如遇报错 error: RPC failed; result=35, HTTP code=0; fatal: The remote end hung up unexpectedly
# 将git clone https://xxxx/xxxx.git 换为git clone git://xxxx/xxxx.git
```

安装virtualbox增强功能

```
# 遇报错 unable to access “VBox_GAS_6.0.0 iso9660”
sudo apt-get install --reinstall linux-image-$(uname -r)
```

最后结果

ubuntu20+py3+zsh（应该能看出来我更喜欢这个x

![image-20211028002711973](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211028002711973.png)

ubuntu16+py2

![image-20211028003012735](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211028003012735.png)

------

第一次装的时候直接卡在了pwntools上，后来发现原因是pip2版本过低且无法直接通过`pip -m pip install --upgrade pip`进行升级，只能手动下载get-pip.py后通过py2运行来安装

这次配完之后最大的感触就是该换个大一点的硬盘了，512G属实绷不住我搞这么多虚拟机……

之后可能会抽空学一下pwn，属于是web狗卷不动了要给自己找个后路（虽然百分之九十九的概率是两个都很菜，xs），卷起来卷起来！！！