---
title: 解决WSL中Nacos无法连接到Mysql数据库的问题
tags:
  - wsl
  - linux
  - nacos
categories: Linux
abbrlink: 9ecb896d
date: 2022-12-28 18:41:29
---

lz今天想在wsl中启动nacos，由于window上已经安装了mysql了，秉着wsl中共享端口的想法，在配置mysql我就连接连接本机的mysql,但是启动的时候始终报错，无法顺利连接到数据库。

查看startup.out

 `Error creating bean with name 'memoryMonitor' defined in URL`

没有什么实际的发现，继续查看nacos.log

`Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure`

可以看到就是mysql的连接有问题。刚开始自然想到由于MySQL的版本是8.0，所有驱动要改成`com.cj.mysql.driver`,以及要驱动url要指定时区`serverTimeZone=Asia/Shanghai`,但是修改之后依旧无法连接，于是上GitHub上issue中找回答，其中一个回答说到`config-final.log`里面有更详细的日志，于是打开分析。

```
 Caused by: java.net.ConnectException: 拒绝连接 (Connection refused)
124 at java.net.PlainSocketImpl.socketConnect(Native Method)
125 at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:476)
126 at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:218)
127 at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:200)
128 at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:394)
129 at java.net.Socket.connect(Socket.java:606)
130 at com.mysql.cj.protocol.StandardSocketFactory.connect(StandardSocketFactory.java:156)
131 at com.mysql.cj.protocol.a.NativeSocketConnection.connect(NativeSocketConnection.java:63)
```

很明显直接被端口拒绝，于是自然想到可能wsl与windows的ip段可能不一样，所以要开启MySQL的远程访问，但是修改之后依旧是这个报错。这个时候我我才恍然大悟，wsl与windows的端口并非一致的，<strong>对于外部的Windows来说，它内部的wsl能映射端口到它上面，并且可以直接通过127.0.0.1来访问，但对于内部的wsl来说，127.0.0.1根本就不能访问外部的3306MySQL服务，</strong>于是将url中的127.0.0.1改为Windows的**对外IP**（192.168.1.201），此时成功解决问题。

