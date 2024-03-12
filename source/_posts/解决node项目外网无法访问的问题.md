---
title: 解决node项目外网无法访问的问题
tags:
  - 前端
  - nodejs
categories: 前端
abbrlink: 51ecfb0e
date: 2023-03-21 21:30:42
---

lz在docker里面部署node前端项目的时候遇到了项目端口外网无法访问的问题。

首先检查docker容器内部是否能正常访问端口。执行`docker exec -it`  容器 进入，用wget 检查端口，发现可以正常访问。此时自然想到是否是端口映射存在问题，但是lz清楚记得运行容器的时候指定了端口映射啊。于是又打开Linux firewalld防火墙对应端口，发现还是不能够访问。那么问题在哪里？是不是node项目本身就不允许外网访问？

秉着这个想法，我把node项目又在本地允许，果然，通过对外ip访问的时候被拒绝。那到底是什么导致了这个问题呢？我对前端本就不太熟悉，又是node项目，更是不会了。正当我苦思冥想的时候，我忽然发现项目允许下面有行提示：

![image-20230522103209437](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230522103209437.png)

`Network: use --host to expose`

是不是我没有指定ip，所以被限制了？于是在网上查找一番，终于找到了[结果](https://blog.csdn.net/qq_41664096/article/details/118961381) 。

大概默认的node ip策略是只允许本地127.0.0.1访问，将他改成0.0.0.0映射全部IP即可。

于是在`vite.conf`里面新增 `host: '0.0.0.0',`即可：

![image-20230522103027345](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230522103027345.png)

运行，成功解决问题。
