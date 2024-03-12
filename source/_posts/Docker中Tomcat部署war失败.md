---
title: 解决Docker中Tomcat部署war失败
tags:
  - Spring
  - Tomcat
  - war
  - docker
categories: Java
abbrlink: 6f162e5f
date: 2023-11-23 10:53:26
---

# 问题描述

博主在接手实验室祖传项目的开发的时候，本地测试都没啥问题，于是直接部署上云服务器，但是死活`404`部署失败。首先由于是祖传项目，用的还是**JDK8**，而且打包的方式也是**war**包的形式，打包成功再部署到`Tomcat`里面。但是部署进去后，访问项目路径，却一直是*404*错误。

<img src="https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_11_19_19_image-20231123111814307.png" alt="image-20231123111814307" style="zoom: 150%;" />

# 问题分析

博主一开始以为是`Tomcat`的问题，因为项目在本地跑的好好的，而且部署的时候用的是`docker`，我就怀疑是不是我`docker`起的`Tomcat`容器有没有问题，于是最简单的方式在`webapps`目录下建立一个`ROOT`文件夹，里面写个Hello World，然后再访问根路径看有没有。

事实证明`Tomcat`是没有问题的：

![image-20231123112408789](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_11_24_8_image-20231123112408789.png)

那么就是**war**包启动的时候有问题了，最好的方法就是看日志了。

进入`Tomcat`的`logs`目录，直接查看日志：

![image-20231123112745572](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_11_27_46_image-20231123112745572.png)

日志一堆，但是最主要的是看`catalina`加日期的那个启动日志文件，那么查看今天的日志：

![image-20231123113015529](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_11_30_15_image-20231123113015529.png)

![image-20231123113031945](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_11_30_32_image-20231123113031945.png)

从上往下分析，发现是启动的时候Spring里面的一个容器无法完成注入，再往下分析，发现在日志的最后一个错误是：

```
Caused by: java.lang.IllegalStateException: Failed to introspect Class [com.gsy.ms.wechat.wechatmp.service.impl.WechatMPServiceImpl] from ClassLoader [ParallelWebappClassLoader^M
  context: ms^M
  delegate: false^M
----------> Parent Classloader:^M
java.net.URLClassLoader@49097b5d^M
]
                at org.springframework.util.ReflectionUtils.getDeclaredMethods(ReflectionUtils.java:507)
                at org.springframework.util.ReflectionUtils.doWithMethods(ReflectionUtils.java:404)
                at org.springframework.util.ReflectionUtils.doWithMethods(ReflectionUtils.java:389)
                at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors(AutowiredAnnotationBeanPostProcessor.java:248)
                ... 57 more
        Caused by: java.lang.NoClassDefFoundError: javax/xml/bind/JAXBException
                at java.base/java.lang.Class.getDeclaredMethods0(Native Method)
                at java.base/java.lang.Class.privateGetDeclaredMethods(Class.java:3402)
                at java.base/java.lang.Class.getDeclaredMethods(Class.java:2504)
                at org.springframework.util.ReflectionUtils.getDeclaredMethods(ReflectionUtils.java:489)
                ... 60 more
        Caused by: java.lang.ClassNotFoundException: javax.xml.bind.JAXBException
                at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1420)
                at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1228)
                ... 64 more
```

最重要的就是这句：

```
Caused by: java.lang.ClassNotFoundException: javax.xml.bind.JAXBException
```

不就是找不到类吗？但是这就比较奇怪了，本地启动为啥就没有报错了呢？

直接搜索这句报错，发现是**Java版本的问题造成的**：

> [这个错误是因为在Java 11及以上版本中，`javax.xml.bind`包已经不再包含在系统库中](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)[1](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)[2](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)[。这个包在Java 6/7/8中是作为JDK的一部分提供的，但在Java 9中，它被认为是Java EE API，因此不再包含在默认的类路径中](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)[2](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)[。在Java 11中，它们完全从JDK中移除了](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)[2](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)。
>
> 解决这个问题的方法有两种：
>
> 1. [如果你正在使用Java 9或10，你可以在运行时通过指定以下命令行选项来使JAXB API可用：`--add-modules java.xml.bind`](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)[2](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)。
> 2. [无论你使用的是哪个Java版本，你都可以通过添加JAXB作为一个单独的依赖项来解决这个问题](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)[1](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)[2](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception)[。如果你的项目使用Maven，你可以在pom.xml文件的``标签中添加相应的依赖](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)[1](https://stackoverflow.com/questions/61177573/javax-xml-bind-cannot-be-resolved)。

就是在Java8的时候这个包还在`rt.jar`下面，可以直接用，到了Java9之后，由于Java9的模块化，很多这种包都被闭包处理了，直接就用不了了。

那么更近一步分析就知道，就是Tomcat容器它用的JDK版本肯定不是Java8了。看一下我这个容器是用的什么镜像，查看启动容器的`docker-compose.yaml`:

```yaml
version: '3.3'
services:
         tomcat:
           image: tomcat:8
           restart: always
           hostname: tomcat
           container_name: tomcat
           ports:
             - 8080:8080
           environment:
             TZ: Asia/Shanghai
           volumes:
             - ./logs:/usr/local/tomcat/logs
             - ./conf:/usr/local/tomcat/conf
             - ./webapps:/usr/local/tomcat/webapps
```

只知道用的是Tomcat8的容器，至于JDK版本就看不到了，那就进入这个容器看看呗：

```bash
docker exec -it tomcat bash	 #进入容器
```

查看Java版本：

![image-20231123115322250](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_11_53_22_image-20231123115322250.png)

好家伙，用的Java版本这么新吗？直接17了。以前光听说企业现在都在往Java17和Java21转移，没想到这种容器真就用最新的了，牛的，支持一波。

那么问题可以说都分析完了，解决起来也就很简单了。

# 问题解决

就如上所说，问题的根源就是**Tomcat容器里面的JDK版本太高了**，解决那也就很简单了，直接**给JDK降低版本**不就行了。但是我这个是用的docker镜像啊？怎么修改？直接到[dockerhub](https://hub.docker.com/_/tomcat/)里面找一下有没有JDK是8的版本不就行了，直接搜索JDK8，果然有一堆：

![image-20231123120117534](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/11/23_12_1_17_image-20231123120117534.png)

找了一下，就拉个写起来简单的吧：

```bash
docker pull tomcat:9-jdk8
```

重新启动个Tomcat容器，部署，成功解决问题。
