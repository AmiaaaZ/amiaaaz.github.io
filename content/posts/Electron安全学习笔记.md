---
title: "Electron安全学习笔记"
slug: "electron-study-notes"
description: "Chromium+NodeJS≈Electron"
date: 2023-08-05T21:49:47+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["Electron"]
draft: false
toc: true
---

正在学习中....

因为学的时候比较发散，不是所有开了的标题下都写完了，就先把标题放上来占坑了！

---

## 什么是Electron

### Electron Runtime的设想

## 调试

### 开启调试

### 源码

### 源码加密

## Electron的架构

## 使用例

### Hello Electron

### IPC通信

## 安全相关的flag

### 攻击思路

## NI1, SBX0, CISO0

### CVE-2021-43908

## SBX0, CISO0 (prototype pollution)

### attack preload.js

### attack Electron internal code

### Discord RCE 1

#### 原型污染

#### XSS (iframe)

#### Navigation restriction bypass (CVE-2020-15174)

## NI0, SBX0, CISO1

### Discord RCE 2

## NI0, SBX1, CISO0

## Electron Shellcode Loader