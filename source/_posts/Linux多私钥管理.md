---
title: Linux多私钥管理
date: 2023-07-24 16:17:15
tags: [Linux,ssh,CentOS]
categories: Linux
---

# 需求分析

今天老大让我上线操作我们的测试服务器，并且直接给了我他的`ssh`私钥。惊，本来以为能给我`root`的密码就已经我都没想过，他这直接把私钥给我了，~~这也太相信我了吧~~。。。

那么现在就是如何用给我的私钥登录服务器了，直接替换本地我自己的私钥肯定是不可能的，不过其实`ssh`命令中指定个参数就能实现了：

```
ssh <username>@<server ip> -i <your private key's position >
```

虽然给干参数就能实现了，但是每次都输一次多少有点麻烦，那么能不能直接在本地管理你的多个私钥呢？

# 需求解决

其实通过配置文件`<./.ssh/config>`就可以解决问题了。关于这个文件[官方文档](https://linux.die.net/man/5/ssh_config)有具体说明，英文好一点的看看能解决基本所有的问题了。

那么根据文档，那我的需求很好解决了，直接上配置文件内容：

```
# Read more about SSH config files: https://linux.die.net/man/5/ssh_config
Host alias
HostName your host
IdentityFile C:\Users\xiao\.ssh\id_rsa.another
PreferredAuthentications publickey
```

其实配置也就四个内容：

- Host：主机别名，用来检索链接名的，比如你给个别名`*.github.com`，那么你`ssh@github.com`的所有请求将使用你配置的那个私钥，根据这个特性其实你还能省去了敲IP或者域名的麻烦，比如现在我就给`Host`配置成`myhost`，那我就能直接用`myhost`来代替实际主机名了。
- HostName：实际主机名，可以是你的服务器或者你的域名。
-  IdentityFile：你的私钥地址，匹配到`Host`就会使用该私钥。
- PreferredAuthentications：优先验证身份规则。

现在按照上面的分析，直接配置，使用`ssh`，直接不用密码连接上，说明私钥配置成功。

![image-20230724164632238](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/07/24_16_46_42_image-20230724164632238.png)
