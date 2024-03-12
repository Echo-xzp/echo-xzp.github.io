---
title: Spring多环境开发
tags:
  - Spring
  - Maven
  - 多环境
categories: Java
abbrlink: 638285de
date: 2023-08-05 10:05:00
---

# 需求分析

Spring项目中在各个环境之中的配置信息都有所区别，如果每次都要手动对每个不同环境下的配置进行选择，未免太过重复和繁琐。因此对每个环境下的配置进行统一管理，当切换环境的时候直接一键切换配置，就显得十分重要。

# 需求解决

## 基于Maven多环境

`maven`中工程属性设置提供了多环境配置设置的，通过在`pom`文件中设置不同环境下的属性配置，再利用`idea`的`maven`插件，即可轻松的切换环境允许程序和打包。

`maven`工程文件`pom`示例：

```
<profiles>
    <profile>
        <id>dev</id>
        <properties>
                <profiles.active>dev</profiles.active>
                <nacos-server>127.0.0.1:8848</nacos-server>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault><!--此处将dev设置为默认环境-->
        </activation>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <profiles.active>prod</profiles.active>
            <nacos-server>172.25.177.179:8848</nacos-server>
        </properties>
    </profile>
    <profile>
        <id>test</id>
        <properties>
            <profiles.active>test</profiles.active>
            <nacos-server>172.25.177.124:8848</nacos-server>
        </properties>
    </profile>
</profiles>
```

这个只是`pom`文件的一部分，主要也就是利用`profiles`标签进行配置。现在已经有了多环境的配置，只要再在Spring的配置文件中读取即可。

Spring配置文件`application.properties`示例：

```

# application
spring.application.name=user-service
# main
spring.main.allow-bean-definition-overriding=true
# 读取当前环境
spring.profiles.active=@profiles.active@
# server
server.port=8186
# nacos
spring.cloud.nacos.username=nacos
spring.cloud.nacos.password=nacos
# 读取nacos地址
spring.cloud.nacos.server-addr=@nacos-server@
spring.cloud.nacos.namespace=${spring.profiles.active}
spring.cloud.nacos.discovery.namespace=${spring.cloud.nacos.namespace}
spring.cloud.nacos.config.namespace=${spring.cloud.nacos.namespace}
spring.cloud.nacos.config.extension-configs=application.properties,${spring.application.name}.properties
```

从`maven`中读取属性采用`@配置名@`的形式读取，如上所示，读取了当前开发环境和`nacos`的配置信息。

但是现在允许程序会发现直接报错，查看日志发现`@配置名@`并没有被替换成目标配置，因而导致项目报错。解决这个问题，就要再在`pom`文件中进行配置，启动配置替换。

示例：

```
<build>
    <resources>
        <resource>
            <!--打包该目录下的 application.yml -->
            <directory>src/main/resources</directory>
            <!-- 启用过滤 即该资源中的变量将会被过滤器中的值替换 -->
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

也就是在`build`标签里面的`resource`标签里添加资源目录的路径，并且启动`filtering`过滤。

现在多环境配置可以说已经基本完成了，现在切换环境，只需在`maven`插件里面勾选所需的环境即可。

如图：

![image-20230805104805733](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/08/5_10_48_17_image-20230805104805733.png)

如上图右所示，即打开`maven`插件，`在配置文件`中勾选所需环境，注意，环境名是由`pom`文件中多环境`profiles`标签里的`id`决定的。

## 基于Spring配置文件

Spring的配置文件本身就支持多环境的配置，如下：

```
server:
  port: 63050

spring:
  application:
    name: media-service
  profiles:
    active: @profile.active@	#当前环境，从maven的pom文件中读取属性

  cloud:
    nacos:
      discovery:
        namespace: ${spring.profiles.active}
      config:
        namespace: ${spring.profiles.active}
        file-extension: yaml
        extension-configs:
          - data-id: application.yaml
            group: DEFAULT_GROUP
            refresh: true # 是否自动刷新配置，默认为 false
# 多环境配置
---
spring:
  profiles: dev
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848
      username: nacos
      password: nacos
---
spring:
  profiles: prod
  cloud:
    nacos:
      server-addr: 172.25.177.179:8848
      username: nacos
      password: nacos
---
```

如上，在YAML配置文件中，我通过分块，配置了不同环境下的`nacos`的配置，同时当前环境的属性从从`maven`的`pom`文件中读取，也就是读取`@profile.active@`，结合上一点所说的方法，也要在父工程的`pom`文件中对这个属性进行多环境配置。从根本上来说，这种方法是在第一种方法的基础上又利用了spring自带的多环境配置。



## 区分

两种多环境各有优点，第二种方法更可以说是第一种方法的基础上来的。两种方法主要是配置细粒度的区别，如我每个服务的`nacos`地址都一样，那就可以采用第一种方式，直接一个`pom`文件配置所有地址；相反，如果我每个地址都有所区别，就可以采用第二种方式。但是总的来说两种方式并不矛盾冲突的，还是要根据实际项目需求进行不同的选择。
