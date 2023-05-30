---
title: 解决nacos-v1.4.5由于未配置jwt签名令牌导致的无法登录问题
date: 2023-05-30 14:42:34
tags: [nacos,Spring,"Spring Cloud","Spring Cloud Alibaba"]
categories: Java
---

# 问题描述

博主在docker中部署`nacos`遇到了数据库配置正常却无法登录的问题。

![image-20230530144609733](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230530144609733.png)

看报错提示首先想到的是不是我数据库里面的`nacos`登录密码有问题。一般默认登录名和密码都是 *nacos/nacos* 。既然初始密码不行那我就自己把数据库里面的~~密码改了不就行了~~。`nacos`的密码采用了**Bcrypt加密算法**，通过[在线工具](https://www.jisuan.mobi/nX7.html)生成密码再修改数据库，再次登录。还是报密码错误。

# 分析问题

看来是另有原因导致了这个问题。于是打开浏览器开发者工具，查看nacos登录接口，发现状态码是**500**，可知是**内部服务器出错**。那么查看docker中nacos的日志文件。nacos的日志文件为nacos目录下的logs目录的`nacos.log`。

查看日志 `vim ./logs/nacos.log` ，通过查找发现：

![image-20230530150321619](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230530150321619.png)



现在错误很明显了。报错的原因是**由未配置JWT的令牌密钥导致的登录无法签发JWT令牌，从而显示~~密码错误~~**。(真是害人精，你要么在运行服务的时候直接报错让服务无法启动，要么你就直接报啥错，说个密码错误等下用户一直以为是自己密码出问题了。)现在找到问题所在就好解决了。

报错日志：

```
caused: The specified key byte array is 0 bits which is not secure enough for any JWT HMAC-SHA algorithm.The JWT JWA Specification(RFC 7518,
Section 3.2)states that keys used with HMAC-SHA algorithms MUST have a size>=256 bits(the key size must be greater than or equal to the hash output size).Consider using the io.jsonwebtoken.security.Keys #secretKeyFor(SignatureAlgorithm)method to create a key guaranteed to be secure enough for your preferred HMAC-SHA algorithm.See https: //tools.ietf.org/html/rfc7518#section-3.2 for more information.;
```



# 解决方法

根据报错在网上查找解决方法，最后发现有篇[CSDN博客](https://blog.csdn.net/dawei1980/article/details/129315241)有详细的解决方法，同时在nacos的github上的[issue](https://github.com/alibaba/nacos/issues/10047)也有相关的讨论。总的来说就是在nacos的配置文件中配置个密钥就行了。配置文件为nacos目录下的：`conf/application.properties`。

```
#不同版本配置项有出入，根据你的nacos版本选择配置
nacos.core.auth.default.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789

#2.1.0版本后
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789
```

但是博主是通过docker部署的，理论上上面的方法我也可以采用，但是直接修改配置文件就又要配置数据卷，显得麻烦一点。通过进入容器，查看nacos容器的默认配置文件。

![image-20230530152117928](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230530152117928.png)

发现它是通过读取一个名为：`NACOS_AUTH_TOKEN`的环境变量实现的。那么只要在启动容器的时候**配置这个环境变量**不就好了？

配置`docker-compose.yaml`:

![image-20230530152424401](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230530152424401.png)

使用`docker-compose`构建容器：

```
docker-compose up -d
```

再次登录，成功解决问题。

![image-20230530152951639](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230530152951639.png)
