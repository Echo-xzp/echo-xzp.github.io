---
title: Hyper-V部署K8S踩坑记录
keywords: [hyper-v,k8s,archlinux,kubelet,kubectl]
date: 2025-08-20 17:30:57
tags: [hyper-v,k8s,archlinux,kubelet,kubectl]
categories: Linux
cover:
description:
---

# 前言

之前就很想搞个自己的`k8s`集群了，但是一直偷懒没有干，最近闲的没事干正好部署一下吧。

部署主要参考：

- Hype-V部署`arhclinux`：参考知乎的一篇文章：[Hyper-V 虚拟机环境安装 ArchLinux](https://zhuanlan.zhihu.com/p/360882712#)
- K8S部署：直接参考[archwiki](https://wiki.archlinux.org/title/Kubernetes)，然后就是`gpt`在细节上的一些答疑

# Hyper-V准备

## 启用hyper-v功能

不需要其他的`VM`平台，直接`win11`自带的`hyper-v`就足够了。之前博主用`WSL`的时候已经把`window`相关功能启用了，所以这里能够直接用了。

## 镜像准备

直接到`archlinux`的官网上下载即可，[官网地址](https://archlinux.org/)

国内速度貌似有点慢？或者直接到镜像源去下载也行

- [aliyun.com](https://mirrors.aliyun.com/archlinux/iso/2025.02.01/)
- [bfsu.edu.cn](https://mirrors.bfsu.edu.cn/archlinux/iso/2025.02.01/)
- [cqu.edu.cn](https://mirrors.cqu.edu.cn/archlinux/iso/2025.02.01/)
- [hit.edu.cn](https://mirrors.hit.edu.cn/archlinux/iso/2025.02.01/)
- [hust.edu.cn](https://mirrors.hust.edu.cn/archlinux/iso/2025.02.01/)
- [jcut.edu.cn](https://mirrors.jcut.edu.cn/archlinux/iso/2025.02.01/)
- [jlu.edu.cn](https://mirrors.jlu.edu.cn/archlinux/iso/2025.02.01/)
- [jxust.edu.cn](https://mirrors.jxust.edu.cn/archlinux/iso/2025.02.01/)
- [neusoft.edu.cn](https://mirrors.neusoft.edu.cn/archlinux/iso/2025.02.01/)
- [nju.edu.cn](https://mirrors.nju.edu.cn/archlinux/iso/2025.02.01/)
- [njupt.edu.cn](https://mirrors.njupt.edu.cn/archlinux/iso/2025.02.01/)

在官网的下载地址里面好像就有每个国家的镜像源地址了，这点好评。

## hyper-v导入镜像

过程很简单的，说几个踩坑点

### 关闭安全启动

这个点比较坑，在新建虚拟机的时候并没有这个选项，但是`安全启动`又是默认启用的，这就导致你创建虚拟机，导入镜像开机直接开机失败，之前博主一直以为是我自己的镜像有问题，或者hyper-v本身和arch的镜像不兼容，结果发现是这里禁用导致的。

![QQ_1758361976662](C:\Users\xiao\AppData\Local\Temp\QQ_1758361976662.png)

### 内存和CPU核数

直接用默认就OK了，hyper-v是在虚拟机运行的时候动态给资源的，比如内存刚开始会比较小，你启动进程多了内存也会同步增长，这点感觉好评，而且博主本身也不需要限制虚拟机的资源，默认感觉就最好了

### 其他

其他的，先用默认的，但是后面也有坑，后面再细说了。

# ArchLinux安装

首先第一点，`archlinux`的安装本身并没有说的那么麻烦，尤其是网络能正常使用的情况下，安装起来更加方便了

之前博主基本都是看这边教程安装的：[archlinux 简明指南](https://arch.icekylin.online/)

里面主要是手动安装，但是我现在重点是关注`k8s`的部署，没必要搞这么多花里胡哨的东西，因此直接使用了`archinstall`一键安装了。过程其实也很简单。

## 更换镜像源

网上找点速度快点的镜像源，改配置文件即可：

```bash
vim /etc/pacman.d/mirrorlist
```

在上面的博文中，还要禁用`reflector`服务，博主觉得这是多此一举了，反而你改一下`reflector`的配置，直接换源都不用了：

```
vim /etc/xdg/reflector/reflector.conf
```

内容：

```
# Set the output path where the mirrorlist will be saved (--save).
--save /etc/pacman.d/mirrorlist

# Select the transfer protocol (--protocol).
--protocol https

# Select the country (--country).
# Consult the list of available countries with "reflector --list-countries" and
# select the countries nearest to you or the ones that you trust. For example:
--country China

# Use only the  most recently synchronized mirrors (--latest).
--latest 12

# Sort the mirrors by synchronization time (--sort).
--sort rate

# 增加天朝的位置
-c China
```

最后加上一个天朝的参数，手动执行：

```bash
 systemctl start reflector.service
```

直接就能根据自己实际的情况找到最快的源了。

## 更新archinstall

```bash
pacman -S archinstall
```

非必要步骤吧，最新的iso里面的都是最新的软件包了，不过更新一下还是好点。

## archinstall

直接手动执行`archinstall`即可，基本是可视化的`TUI`界面了，选择自己需要的一些选项即可，中间重要的就是要**关闭交换分区**，其他的话参照`archwiki`和`archinstall`自己的文档基本上就差不多了。

## 踩坑

1. 没有关闭删除光驱启动和调整启动顺序

   安装完毕之后，光驱倒是可删除可不删除，但是启动顺序一定要调整，这就是非常坑的一个点：

   ![QQ_1758363698754](C:\Users\xiao\AppData\Local\Temp\QQ_1758363698754.png)

​	这个非常的坑，默认磁盘启动竟然是最后一个，博主之前没注意，以为把光驱删除了就OK了，结果开机直接卡在进入	界面，还以为是自己的安装哪里有问题，磁盘挂载出错了，结果是引导顺序的问题。

2. 网络工具缺失

   这个也比较坑，`archinstall`没指定额外软件包的话，默认进系统里面啥基础的网络管理工具都没有，然后`archlinux`本身就是毛坯房，这就搞得十分的痛苦，网络正常还就用不了

   几个解决方法：

   - `archinstall`的时候指定额外的软件包，`network-manager`，`dhcpd`这种，实际进系统里面起码网络是可用的，也能继续安装其他软件包了

   - 手动改网络，这点也比较蠢，博主之前搞不懂hyper-v的网络拓扑，不知道改配什么IP和网关，后面才知道这个是网络适配器里面能看到的。默认的是`defualt-swtich`，这个实际是有对应的，Windows终端里面能直接看到的：

     ![QQ_1758364273326](C:\Users\xiao\AppData\Local\Temp\QQ_1758364273326.png)

     那就好办了：

     ```bash
     # 配置和swtich同网段ip
     ip a a 172.26.160.123/20 dev ens18
     # 添加默认路由规则，下一跳指向swtich
     ip r a default via 172.26.160.1
     ```

     这个是临时改IP，重启失效，不过起码网络先能用再说

   - 重新挂载磁盘，用`pacstrap`重新安装网络工具。最傻逼的方法，也是博主用的，重新从光驱启动，进入arch安装，手动把已经分好区的磁盘挂载，然后安装相关工具：

     ```bash
     # 实际上btrfs分了好几个卷，还不能这样只挂载根目录，演示就这样吧
     mount -t btrfs -o subvol=/@,compress=zstd /dev/sdxn /mnt
     pacstrap /mnt neovim
     ```

3. 网络失效

   同样傻逼的事情，博主配置好了固定IP，也能正常使用了，但是重启电脑发现，虚拟机的网络全部掉了，百思不得其解。研究了半天发现是`hyper-v`自己默认的`swtich`导致的，和上面说的，我设置固定IP的时候，把网关指定为`swtich`的IP了，但是这个IP实际是**动态的**！每次重启都会改变，那这就傻逼了吧。因此需要自己配置一个虚拟机交换机。

   参照这篇博客[Hyper-v虚拟机设置静态IP - 嘘，别吵 - 博客园](https://www.cnblogs.com/macho8080/p/15180419.html)，一个是新增虚拟交换机，然后就是加NAT规则，博主测试是能够正常使用的

# K8S部署

本来以为会是很复杂的，但是部署下来，就开发环境的话，博主部署还是很顺利的，大部分原因还是由于网络原因造成的（众所周知的原因），另外来说，archlinux就是包多，k8s的包都不用在`AUR`里面找，之前在`CentOS`上，还得加`yum`的`k8s`源，安装容器运行时还有依赖冲突（貌似是`containerd`和`podman`依赖的`runc`有问题），`arch`就比较顺利了。

## 控制面节点

-  [kubernetes-control-plane](https://archlinux.org/groups/x86_64/kubernetes-control-plane/)，包括[ kube-apiserver](https://archlinux.org/packages/extra/x86_64/kube-apiserver/)，[ kube-controller-manager](https://archlinux.org/packages/extra/x86_64/kube-controller-manager/)，[ kube-proxy](https://archlinux.org/packages/extra/x86_64/kube-proxy/)，[ kube-scheduler](https://archlinux.org/packages/extra/x86_64/kube-scheduler/)，[kubelet](https://archlinux.org/packages/extra/x86_64/kubelet/)。
- 容器运行时：[containerd](https://archlinux.org/packages/?name=containerd)。可以选其他的，博主直接安装`docker`了，实际还是`containerd`
- k8s引导工具：[kubeadm](https://archlinux.org/packages/?name=kubeadm)，博主比较菜，直接用一键引导了，实际效果也是好，几个命令就搭建好集群环境了。
- 控制面前端工具：[kubectl](https://archlinux.org/packages/?name=kubectl)

## 数据面节点

- [kubernetes-node](https://archlinux.org/groups/x86_64/kubernetes-node/)，包括[ kube-proxy](https://archlinux.org/packages/extra/x86_64/kube-proxy/)，[kubelet](https://archlinux.org/packages/extra/x86_64/kubelet/)。
- 容器运行时：[containerd](https://archlinux.org/packages/?name=containerd)。可以选其他的，博主直接安装`docker`了，实际还是`containerd`
- k8s引导工具：[kubeadm](https://archlinux.org/packages/?name=kubeadm)，博主比较菜，直接用一键引导了，实际效果也是好，几个命令就搭建好集群环境了。

## 启动服务

- 启动容器运行时

  ```bash
  systemctl enable --now containerd.service
  ```

- 启用kubelet

  ```bash
  systemctl enable --now kubelet.service
  ```

有个点,`kubelet`集群没初始化成功，他会一直失败重启，这个属于正常现象。

## 踩坑点

基本上都是国内容器镜像拉取的问题，因此要改一下`containerd`的镜像代理：

```bash
vim /etc/containerd/config.toml
```

代理源自己找吧，我用的也不稳定，看到文章的时候十有八九已经寄了。。。

不过主要时`pause`基础镜像这个，倒是可以改成阿里云的代理：

```toml
   [plugins.'io.containerd.cri.v1.images'.pinned_images]
      sandbox = 'registry.aliyuncs.com/google_containers/pause:3.10'
```

提问：pause镜像什么作用？

答：pod的基础镜像，由Pause镜像申请到一个`namespace`,`pod`里面的其他容器加入到这个`namespace`里面。

## 控制面节点

控制面启动节点：

```bash
kubeadm init --apiserver-advertise-address=<your_ip> --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=Mem  --image-repository=registry.aliyuncs.com/google_containers
```

两个点：

1. 使用阿里云的代理镜像源，不然拉取进行十有八九直接失败
2. 忽略内存不足错误，上面说的，hyper-v动态给内存，导致启动的时候检查到内存过小

应该会正常启动，启动失败就看`kubelet`的日志，十有八九是容器启动的问题，可以去看`containerd`的日志，估计就是拉取什么镜像超时了。

然后应该有相关的提示，按照它给的提示，等下再到数据面节点上操作，让节点加入即可。

执行成功应该能看到自己`kube-system`下的pod都就绪OK了。

```bash
kubectl get pods -n kube-system
```

结果：

```
NAME                                   READY   STATUS    RESTARTS       AGE
coredns-757cc6c8f8-9n8tl               1/1     Running   1 (139m ago)   5d22h
coredns-757cc6c8f8-vml9b               1/1     Running   1 (139m ago)   5d22h
etcd-k8s-master01                      1/1     Running   2 (139m ago)   5d22h
kube-apiserver-k8s-master01            1/1     Running   2 (139m ago)   5d22h
kube-controller-manager-k8s-master01   1/1     Running   2 (139m ago)   5d22h
kube-proxy-frwl4                       1/1     Running   1 (139m ago)   5d22h
kube-proxy-jqvhm                       1/1     Running   1 (139m ago)   5d22h
kube-scheduler-k8s-master01            1/1     Running   2 (139m ago)   5d22h
```

然后就是部署`CNI`，选择`flannel`或者`calico`，博主就是自己的测试环境，直接`flannel`就OK了，这里也是

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

没代理，估计就失败了，然后又是在虚拟机里面，博主是直接把这个描述文件下载下来再执行的。

存在问题：里面的容器镜像也是用不来的，自己找代理改一下吧。。

执行完成，

```bash
kubectl get pods -n kube-flannel
```

应该也是就绪OK

## 数据面节点

上面控制面节点初始化成功的时候的提示，现在执行

```bash
kubeadm join 172.23.166.250:6443 --token xxxxx --discovery-token-ca-cert-hash xxxxx
```

等待加入集群就行了，注意，要把刚才主节点上的改代理的操作也做一遍，不然pod起不来，会看到数据节点一直是*No Ready*状态，一般没问题，有问题就是镜像的问题。

执行完成，控制面执行：

```bash
kubectl get nodes
```

应该节点都是OK的了。

```
NAME           STATUS   ROLES           AGE     VERSION
k8s-master01   Ready    control-plane   5d22h   v1.33.4
k8s-worker01   Ready    worker          5d22h   v1.33.4
```

# 总结

简单的一个控制节点和数据节点，和k8s要求最基础的三个工作节点和两个控制节点还差很多，后面再补坑吧。。

不过发现hype-v还有个差异卷的东西，能实现虚拟机复制的效果，还减少空间，应该实现还是比较简单的，

另外一个想法，WSL应该和虚拟机是可以互通的，凭借这个，WSL也可以做控制节点，这样再工作上应该更加方便很多了，很多IDE集成了WSL了，我再Hype-v里面开两个虚拟机，效果应该很不错吧。

环境倒是有了，分布式的学习好久都停了，最近搞个`envoy`的`service mesh`服务试试，，，
