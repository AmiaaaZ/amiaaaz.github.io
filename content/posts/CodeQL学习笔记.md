---
title: "CodeQL学习笔记"
slug: "codeql-study-notes"
description: "可能是最后一个学CodeQL的人了（自豪脸（不是"
date: 2023-07-23T02:08:39+08:00
categories: ["NOTES&SUMMARY"]
series: ["自动化代审"]
tags: ["CodeQL"]
draft: false
toc: true
---

*更新中，博客存在更新延迟

----

## 环境配置

- 本体

Visual Studio Code安装CodeQL插件

https://github.com/github/codeql/tags下载规则库

- 创建项目

Github项目支持直接url导入，其它的也可以指定文件夹

*哈哈 忍不住开骂了，nmd傻逼vscode就是脑残。这个垃圾插件做的，没有一次是导入一次数据库就可以成功的 一定要导入两次 然后工作区就会多两个文件夹，如果我删除一个 插件页面就会丢失数据库 还需要重新导入，我测死你的🐎

- 执行查询

运行`*.ql`文件时需要当前工作区里存在`qlpack.yml`文件，但那个语法我真的望而生畏（codeql你也司马了）我选择直接打开`codeql-codeql-cli-<version>/<language>/ql`目录，然后在test目录下默默写查询规则

