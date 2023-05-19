---
title: 解决Spring中maven依赖冲突
date: 2023-03-09 20:12:15
tags: [spring,java,"JavaWeb",maven]
categories: Java
---

lz今天做微服务的时候刚启动项目测试，便遇到了：

`org.springframework.core.metrics.ApplicationStartup`

大概意思应该是spring boot依赖版本冲突。

于是我根据maven的子模块往父模块一层的往上找，但还是没找到问题所在。由于模块之间又层级继承，我只能一个一个注释测试，最终终于找到了问题所在。

在根父工程里面，由于我使用`spring initialzr`创立工程，默认的继承了`spring-boot-parent`父工程。同时在 pom文件`dependence manager`标签 里面我又重新指定了spring dependencies 的版本，于是造成了冲突。

`自有父依赖：`

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.6.11</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

`声明的依赖：`

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>${spring-cloud.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```



去掉父工程继承之后完美解决问题。
