---
title: 解决nacos已连接MySQL数据库却无法发布配置的问题
date: 2023-03-17 15:53:54
tags: [nacos,mysql,"Spring Cloud"]
categories: Java
---

lz今天在docker里面部署nacos的时候遇到了一个问题，就是*nacos 已连接MySQL数据库却无法发布配置*。

首先确定是否已经正确连接MySQL，最简单的方法就是我把nacos密码从数据库修改为其他，就无法登录了。于是我正常想到今天建nacos数据表的时候，是随手在网上copy一份下来执行的，大概由于版本问题导致了这个问题了吧。

于是到网上一搜，果然: `nacos 1.4.0以下使用的mysql驱动是8.0以下的，1.4.0以上使用的驱动就是8.0以上的，使nacos和数据库的版本对应。`

nacos一般会在conf/ 目录下有几个关于数据库建表的文件，但是由于我是docker部署的，就偷懒没拷贝出来，于是导致了这个问题。

只好老老实实从容器里面拷贝了： `docker cp nacos:/home/nacos/conf/mysql-schema.sql ./`

再执行sql文件，成功解决问题。果然偷懒还是容易导致一些莫名其妙的问题啊。
