---
title: Linux一般用户去掉sudo权限使用docker的方法
tags:
  - Linux
  - docker
  - Arch Linux
categories: Linux
abbrlink: 68633db7
date: 2023-05-29 10:23:50
---

在Linux中安装Docker后，每次使用docker都要添加`sudo`命令，这样既不方便而且在操作的时候可能还会由于权限问题导致各种问题。虽然可以通过切换`root`用户来解决，但是在Linux中直接使用`root`用户操作系统本身就是一种不太安全的行为。那么如何是一般用户不用`sudo`命令也能操作docker呢？

未使用`sudo`命令报错：

![image-20230529103007763](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230529103007763.png)

从报错中可以直接看出，普通用户对`/var/run/docker.sock`没有读写权限，于是被拒绝操作。

那么知道问题所在就可以解决问题了，最简单的方法就是直接把这个文件的权限放开，让一般用户都有都写权限：

```
sudo chmod a+rw /var/run/docker.sock
```

理论上这样就可以解决问题了，但是既然docker默认不让其他用户访问这个文件肯定有它的安全性考虑，上面这样的操作实际生产环境中应该是不可取的。

那么再分析，先看看这个文件的属性，执行： 

```
ls -l /var/run/docker.sock
```

文件信息如下：

![image-20230529103739147](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230529103739147.png)

就像刚才所说，它的**所有者和组成员才有读写权限，其他用户没有任何操作权限**。

看信息可知它的所属组为：`docker`，那么把**普通用户加入docker组**不就解决问题了吗？

执行：

```
 sudo gpasswd -a $USER docker
```

添加组成员，再查看是否添加成功。

执行：

```
sudo   cat /etc/group |grep $USER
```

可见成功添加：

![image-20230529104408897](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230529104408897.png)

添加后一定要记得**刷新组信息**：

```
newgrp docker
```

现在应该可以不用`sudo`，直接使用docker命令了。

![image-20230529104640655](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230529104640655.png)

