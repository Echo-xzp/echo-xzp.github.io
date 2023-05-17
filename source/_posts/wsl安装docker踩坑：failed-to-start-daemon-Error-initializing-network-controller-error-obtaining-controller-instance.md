---
title: >-
  wsl2安装docker报错：failed to start daemon: Error initializing network controller:
  error obtaining controller instance
date: 2022-08-30 13:12:14
tags: wsl 
categories: 杂项
cover: /img/cover6.png
---

<h1>wsl2 安装docker无法启动</h1>
lz今天在wsl2 ubuntu系统中安装docker时遇到了奇怪的事情。docker安装完毕之后，执行

service docker start 后没有正常运行

报错 :

`Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`

执行 ：`service docker status` 后发现实际docker没有运行

查看日志 ：`cat /var/log/docker.log`

发现端倪 ：

failed to start daemon: Error initializing network controller: error obtaining controller instance`

在网上找了一遍，最终找到一个帖子说清的原因和解决方法: [点击查看](https://juejin.cn/post/7063767884744359973)

执行 ：

`update-alternatives --config iptables`

输入1切换模式即可。

再次运行docker,发现完美解决问题。
