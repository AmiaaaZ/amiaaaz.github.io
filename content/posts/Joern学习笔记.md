---
title: "Joern学习笔记"
slug: "joern-study-notes"
description: "一个不需要预先编译项目的代审工具，浅浅入坑"
date: 2023-07-23T02:12:17+08:00
categories: ["NOTES&SUMMARY"]
series: ["自动化代审"]
tags: ["Joern"]
draft: false
toc: true
---

*更新中，博客存在更新延迟

----

Joern和CodeQL类似，都是静态代码分析类的工具 支持多种语言，但Joern不需要在执行查询前对项目进行编译，语法基于Scala 不会有CodeQL那么难懂的谓词，更特色的是Joern在分析代码时使用的是CPG(Code Property Graph)，看起来不错的样子 于是正式入坑Joern！