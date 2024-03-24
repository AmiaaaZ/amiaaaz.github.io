---
title: "共享宿主机PID namespace与容器逃逸"
slug: "pid-namespace-share-and-escape"
description: "不实用的攻击面"
date: 2024-03-24T23:52:57+08:00
categories: []
series: ["容器逃逸"]
tags: ["docker", "escape"]
draft: false
toc: true
---

## 前提

- 容器共享宿主机PID name space（非常苛刻 这属于docker默认开启的安全措施）
- 以uid为0运行
- 没有启用user namespace（默认就不启用）
- 没有添加额外的capbilities

可以利用某些进程的`/proc/[pid]/root`实现逃逸

理想docker启动命令：

```bash
docker run --rm -it --pid=host --security-opt=apparmor=unconfined ubuntu bash
```

## /proc/[pid]/root

根据proc(5)介绍，通过该符号链接可以访问任意进程的rootfs，如论当前进程和目标进程是否属于同一mount namespace，而调用/访问该符号链接需要经过ptrace(2)的权限检查

根据手册，ptrace access mode checking会基于两进程的credentials和capabilities进行检查，其中的`PTRACE_MODE_READ`就是针对符号链接的访问

那具体这个check会如何进行呢？相关函数调用关系如下

```
/proc/[pid]/root
-> proc_pid_get_link() # 获取链接的实际路径
-> proc_fd_access_allowed() # 检查权限 通过符号链接的inode拿到进程的task_struct实例
-> ptrace_may_access() # 检查权限 调用该函数时第二个参数为PTRACE_MODE_READ_FSCREDS
-> security_ptrace_access_check()
-> cap_ptrace_access_check()
```

在ptrace_may_access中完全允许同一线程组进行访问

![image-20240218210450630](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218210450630.png)

因为有标志位，需要检查当前进程和目标进程的fsuid和fsgid是否一致，如果不一致就会调用ptrace_has_cap()检查当前进程在目标进程的name userspace中是否拥有CAP_SYS_PTRACE权限，如果没有则拒绝访问

![image-20240218212029516](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218212029516.png)

如果以上都顺利则跳转ok部分，当目标进程被设置为nondumpable且当前进程在目标进程的user namespace中没有CAP_SYS_PTRACE权限时拒绝访问

![image-20240218212344103](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218212344103.png)

最后调用security_ptrace_access_check()进行最终检查（和LSM有关 略过yama等的实现），对于commcap, security_ptrace_access_check()最终调用的时/security/commcap.c文件中的cap_ptrace_access_check()函数

![image-20240218212714427](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218212714427.png)

根据是否设置标志位来决定使用effective capbility set有效能力集还是permitted capbility set许可能力集执行权限检查，接着如果两进程属于同一user namespace 且目标进程的许可能力集是当前进程能力集的子集，则通过检查，否则接着检查当前进程在目标进程所在的user namespace中是否拥有CAP_SYS_PTRACE能力，有则通过

在默认配置下所有容器拥有相同的能力集，所以共享了宿主机PID namespace的容器能够访问其它容器进程的rootfs

无法访问宿主机上以非root用户运行进程的rootfs，是因为当前进程/调用进程的fsuid, fsgid分别与目标进程的euid, suid, uid, egid, sgid, gid不匹配；无法访问宿主机上以root用户运行进程的rootfs，是因为能力集的问题（非子集）

## 利用思路

根据以上研究，我们可以有这样的结论

- 调用进程和目标进程属于统一个进程组，允许访问
- 如果访问模式中指定PTRACE_MODE_FSCREDS标志位，则在接下来检查调用进程的filesystem UID(fsuid)和filesystem GID(fsgid)，如果指定PTRACE_MODE_REALCREDS标志位，则检查real UID(uid)和real GID(gid)
- 调用进程的fsuid, fsgid需要分别与目标进程的euid, suid, uid, egid, sgid, gid匹配
- 调用进程在目标进程的user namespace中需要拥有CAP_SYS_PTRACE能力
- 如果目标进程被设置为nondumpable，且调用进程在目标进程的user namespace中没有CAP_SYS_PTRACE能力，那么拒绝访问
- 如果不能满足以下任意一个条件，那么内核的commcap LSM模块会拒绝访问：
  - 调用进程和目标进程属于同一个user namespace，且调用进程的能力集是目标进程的许可能力集的超集
  - 调用进程在目标进程所在的user namespace中拥有CAP_SYS_PTRACE能力

所以可以如何逃逸？我是没想出来，Nitro师傅：

- 解决fsuid等不匹配：通过PID namespace共享找到宿主机上以非root用户运行的进程，我们在容器中创建一个uid和gid都与目标进程uid gid相同的用户，再在容器中su切换，就有权限访问目标进程的/proc/[pid]/root了，这个目标进程必须是dumpable的

非常好理解，但是也非常苛刻：我们怎么知道目标进程的uid和gid呢？

## 实操

```bash
docker run --rm -it --pid host ubuntu
```

这样flag下运行的docker，可以显示和宿主机一样的进程列表

![image-20240218215359045](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218215359045.png)

在其中我们可以找到以普通权限运行的进程

```bash
ps -a -o pid,ppid,user,group,args
```

![image-20240218215825070](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218215825070.png)

我们创建对应用户

```
useradd -m -u 1000 -U nise
su nise
```

但是却没办法做到`ls -l /proc/[pid]/root/`，permission denied

![image-20240218223210267](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218223210267.png)

看了一下以这个为原型的RWCTF 2024的题，原来docker flag还需要加上这个flag

```bash
docker run --rm -it --pid=host --security-opt=apparmor=unconfined ubuntu bash
```

![image-20240218223813997](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240218223813997.png)

即`--security-opt=apparmor=unconfined`选项同时开启的情况下，可以完成逃逸

简便措施：

```bash
for fd in `find /proc/*/root`; do ls -al $fd | grep \>; done
```

直接找哪些进程的root能打开

------

anyway, but

和进程安全有关的选项，如果谁脑子想不开了用这个flag，想必它的主机也不安全

参考文章：

[一个未公开的容器逃逸方式](https://www.anquanke.com/post/id/290540)

RWCTF 2024 出题思路&wp: https://github.com/WinMin/Be-a-Docker-Escaper/blob/main/Be-a-Docker-Escaper-4/README.md