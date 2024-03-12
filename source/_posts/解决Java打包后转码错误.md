---
title: 解决Java打包后转码错误
tags:
  - Java
categories: Java
abbrlink: a6abfa2c
date: 2022-10-09 14:37:44
---

# 解决使用Java  URL.encode将中文转码打包运行时自动添加了+号

lz做一个项目时要求将中文利用`URL.encode`转码，本地运行调试一切正常，但是将项目打成jar运行时就一直报错，进行排查后发现时中文转码出现了问题。

`URL.encode`的默认方法是一个被弃用的方法，查看文档可知该方法转码依赖运行的平台，所以达成jar运行就报错了，而再本地运行又没问题。同时空格会被转码为+号。

改为新方法 : `URL.encode(*,"UTF-8")`;

手动指定转码方式，并调用trim裁剪，便可以解决问题了。
