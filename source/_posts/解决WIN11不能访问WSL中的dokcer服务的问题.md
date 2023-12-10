---
title: 解决WIN11不能访问WSL中的docker服务的问题
date: 2023-12-10 14:26:38
tags: [Windows11,wsl,Debian,docker]
categories: Linux
---

# 问题描述

博主上个月重装了一次系统，直接升级到了Win11，之前配置的WSL直接被干掉了，后面懒就一直没有再重装，前两天闲的没事，就又重新装了一次WSL，这次还装了个Debian的系统。

![image-20231210143306870](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_14_33_14_image-20231210143306870.png)

安装倒是挺顺利的，该配置的也配了，但是今天忽然想在docker里面起个MySQL，结果就遇到了问题。主要就是MySQL在docker里面是确实跑起来了的，但是我在宿主机，也就是Windows11里面想用`navicat`连上去却发现一直连接不上。

![image-20231210143831662](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_14_38_31_image-20231210143831662.png)

# 问题分析

首先想到的是`docker`起的`MySQL`本身是不是~~有问题~~？直接看日志就行了：

![image-20231210144035180](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_14_40_35_image-20231210144035180.png)

明显没有问题。

于是楼主又想到WSL是不是现在和宿主机不能共用一个IP了？于是直接起个`nginx`来看看：

```bash
sudo apt install nginx
sudo systemctl start nginx
```

宿主机访问看看：

![image-20231210144353438](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_14_43_53_image-20231210144353438.png)

换成局域网IP也一样能访问，那就说明WSL和宿主机还是能共用一个IP的啊，那么问题就不在在这里。

那么是不是`docker`的问题呢？`docker`里面起一个`nginx`看看：

```bash
sudo docker pull nginx
sudo docker run -d --name tmp -p 80:80 nginx
```

再在Window11里面的浏览器访问一下，发现失败了，也就是说**确实就是docker的原因**！

![image-20231210145054219](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_14_50_54_image-20231210145054219.png)

这下终于找到问题所在了，但是在测试分析的时候我又发现一个更加奇怪的现象，就是**`docker`起的服务虽然宿主机访问不了，但是在`WSL`里面却能访问!**

![image-20231210145511450](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_14_55_11_image-20231210145511450.png)

这下迷糊了，只知道是`wsl`中`docker`的原因，具体怎么解决博主也是一筹莫展，但是博主猜测应该也是`docker`，`WSL`,`Windows`，有三层网络链路的时候，就导致服务无法访问了吧。

# 问题解决

那就只能去网上找解决方法了，不过由于知道是什么原因导致的，`WSL`和`docker`在`github上`直接可以提`issue`，直接上去看看有没有和我一样的情况，果然在`WSL`的[issue](https://github.com/microsoft/WSL/issues/10494#issuecomment-1754170770)里面，有一群和我一样的问题的人，并且**里面还有人提出了解决方案**！

这位老哥直接整合大家的解决方法，给出了两种[解决方案](https://github.com/microsoft/WSL/issues/10494#issuecomment-1813011387)：

> So the two workarounds I see so far, both have their negatives but worth listing anyway. Correct any of my misunderstandings.
>
> **Option1 - without docker desktop**
>
> Edit `%USERPROFILE%\.wslconfig` file (create it, if doesn't exist) and add this section
>
> ```
> [experimental]
> networkingMode=mirrored
> hostAddressLoopback=true
> ```
>
> 
>
> Restart WSL, then in your distro, edit `/etc/docker/daemon.json` and add
>
> ```
> {
> "iptables": false
> }
> ```
>
> 
>
> Then `sudo systemctl restart docker`.
>
> With this, both normal port usage like `python3 -m http.server` and docker port usage like `docker run -p "8080:8080" --rm -t mendhak/http-https-echo:26` are accessible from Windows side. The disadvantage is that iptables is now disabled.
>
> **Option 2 - with Docker Desktop**
>
> Edit `%USERPROFILE%\.wslconfig` file (create it, if doesn't exist) and add this section
>
> ```
> [experimental]
> networkingMode=mirrored
> hostAddressLoopback=true
> ignoredPorts = 8000,8080
> ```
>
> 
>
> Restart WSL and ensure Docker Desktop is running. With this, the normal port usage stops working if it uses a port listed above. eg the Python http.server isn't accessible over 8000 anymore (at least in my testing). Instead use a different port, like `python3 -m http.server 8001`. Docker ports will work as normal as long as the port is listed in the `ignoredPorts` above, like `docker run -p "8080:8080" --rm -t mendhak/http-https-echo:26`
>
> The disadvantage here is you have to know which ports you'll be using with Docker. And it doesn't look like `ignoredPorts` accepts ranges of ports either so it can get pretty tedious.
>
> And ignoredPorts only applies to docker containers it seems, not simple python http servers, unless I messed up?
> Did anyone run into the ignoredPorts issue as I did, for non-Docker applications?

其实也就是修改那两处的配置文件，要么`wsl`忽略端口，要么`docker`关闭`iptables`。但是上面也说到两种方案都有弊端，只能现在暂时的解决问题，实际还得官方更新修复这个`bug`。

最终博主也是通过第一种方案解决的，再次`navicat`连接`docker`里面的`MySQL`，成功解决问题：

![image-20231210150826660](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/12/10_15_8_26_image-20231210150826660.png)

不过对于实际什么原因造成的这个`bug`,博主其实还是有点搞不懂，不过[issue](https://github.com/microsoft/WSL/issues/10494#issuecomment-1741605674)里面也有老哥给出了大概的`bug`分析

> there seem to be two issues why Docker containers cannot connect from Windows.
>
> 1. `ignorePorts` works
>    Docker Desktop, probably
> 2. `the port forwarding does work from another machine on the same network as host. Just not on the host machine itself.`
>    docker-ce that installed on Linux
>
> temporary measures for `2` ....
> `disable iptables and use docker-proxy`
>
> /etc/docker/daemon.json
>
> ```
> {
>     "iptables": false
> }
> ```
>
> 
>
> when using mirrored, the behavior seems to be different from the previous localhostforwarding.
>
> use docker-proxy(listen on Linux)
>
> ```
> windows  --->  linux(WSL)
>   localhostforwarding (until)
> ok    127.0.0.1   --->     127.0.0.1(lo)
> ok    127.0.0.1   <---     127.0.0.1(lo)
> 
>   netowrkingMode=mirrored
> ok    127.0.0.1   --->     127.0.0.1(loopback0)
> ok    127.0.0.1   <---     127.0.0.1(loopback0)
> ```
>
> 
>
> interface is different, but the behavior remains the same.
>
> use iptables(listeon on container)
>
> ```
> windows  --->  linux(WSL) ---> Docker container
>   localhostforwarding (until)
> ok    127.0.0.1   ---> 127.0.0.1(lo)    /   172.x.x.1(br-xxx)  --->  172.x.x.x(eth0)
> ok    127.0.0.1   <--- 127.0.0.1(lo)    /   172.x.x.1(br-xxx)  <---  172.x.x.x(eth0)
> 
>   netowrkingMode=mirrored
> ok    127.0.0.1   ---> 127.0.0.1(loopback0)/127.0.0.1(br-xxxx) --->  172.x.x.x(eth0)
>                                           ~~~~~~~~~
> ng                                                           <- - -  172.x.x.x(eth0)
> ```
>
> 
>
> via localhostforwarding(until), source address(Windows) was the docker network gateway (=pointing to linux).
>
> via mirrored, source address is 127.0.0.1.
> for this reason that packet is returned to 127.0.0.1(localhost on container) and does not reach Windows.
> in the case of access from another node(PC), there is no problem because source is his address.
