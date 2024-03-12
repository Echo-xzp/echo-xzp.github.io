---
title: 接入Spring Boot Admin的踩坑记录
tags:
  - Spring
  - Spring Boot
  - Spring Boot Admin
  - Spring Boot Actuator
categories: Java
abbrlink: 88126f7e
date: 2023-05-23 10:47:57
---



# 什么是 Spring Boot Admin ?

在它的[开发文档](https://consolelog.gitee.io/docs-spring-boot-admin-docs-chinese/)中已经说的很清楚了：*Spring Boot Admin 是 codecentric 公司开发的一款开源社区项目，目标是让用户更方便的管理以及监控 [Spring Boot](http://projects.spring.io/spring-boot/) ® 应用。 应用可以通过我们的Spring Boot Admin客户端（通过HTTP的方式）或者使用Spring Cloud ®（比如Eureka，consul的方式）注册。 而前端UI则是使用Vue.js，基于Spring Boot Actuator默认接口开发的。*我的理解它就是一个类似注册中心一样的**实例检测中心**，能够直白的监测到实例的运行状态和系统占用情况，还能配置通知中心实现实例异常报警。

![image-20230523105632276](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230523105632276.png)

# 如何接入？

还是上面提到的[开发文档](https://consolelog.gitee.io/docs-spring-boot-admin-docs-chinese/)，比起在网上看各种良莠不济的博文，不如直接看它的开发文档，这不比其他的更加详细清楚？但是我还是讲讲自己是如何接入的吧，就当做个学习记录。

## 客户端依赖导入

Spring Boot Admin 可以看成两部分，一部分是客户端，一部分是服务端。

客户端就是要接入Spring Boot Admin 的实例，导入Spring Boot Actuator依赖就可以了，而这个依赖就是给实例自动加入一个监测端点，暴露出一些关于实例运行状态的接口，而Spring Boot Admin 服务端检测这些接口，在进行数据可视化。

导入依赖：

```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

服务端导入的依赖相对多了一点：

```
<dependencies>
        <!-- Nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <!-- de.codecentric -->
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
            <version>${sba.version}</version>
        </dependency>
        <dependency>
            <groupId>org.jolokia</groupId>
            <artifactId>jolokia-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
		// 邮箱报警依赖
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>

    </dependencies>
```

我采用集成nacos的方案，单体项目或者用其他的注册中心的可以参照文档自行修改。

## 基本配置

*启动项yaml:*

```
server:
  servlet:
    context-path: /sba
spring: 
  boot:
    admin:
      ui:
        title: xuecheng
   # 登录所用用户名密码
  security:
    user:
      name: admin
      password: 123456
  cloud:
    nacos:
      discovery:
       # 配置元数据
        metadata:
         # sba检测端点前缀添加
          management:
            context-path:  ${server.servlet.context-path}/actuator
         # 
          username: ${spring.security.user.name}
          userpassword: ${spring.security.user.password}
  # 报警邮箱配置
  mail:
    host: smtp.qq.com
    port: 25
    username: xxxxxxxxx@qq.com
    password: xxxxxxxxx
  boot:
    admin:
      notify:
        mail:
          enabled: true
          # 自定义发送页面
		 # template: "classpath:/META-INF/spring-boot-admin-server/mail/status-changed.html"
          to: xxxxxx@qq.com
          from: xxxxxxx@qq.com # 要和mail中的username一致，否则报错
          
management:
 # 暴露端点
  endpoint:
    health:
      show-details: ALWAYS
    web:
      exposure:
        include: '*'
```

**注意**：这个配置是不完全的，只提供了主要的配置项，具体需要根据自己的项目进行补全。



由于引入了`Spring Security` ，所以还需要写一个它的配置类用来给登录接口放行，创立`SecuritySecureConfig.java`

```
package work.echoes.sba.config;

import de.codecentric.boot.admin.server.config.AdminServerProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {
    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminContextPath + "/");

        http.authorizeRequests()
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .anyRequest().authenticated()
                .and().formLogin().loginPage(adminContextPath + "/login")
                .successHandler(successHandler)
                .and().logout().logoutUrl(adminContextPath + "/logout")
                .and().httpBasic()
                .and().csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringAntMatchers(adminContextPath + "/instances", adminContextPath + "/actuator/**");
        // @formatter:on
    }
}

```

现在的基本工作就完成了，登录时使用在yaml中配置的Security中的`username`和`password`即可登录。

![image-20230523144507302](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230523144507302.png)

# 踩坑点

## 项目配置context-path造成的服务一直显示Down状态

### 问题描述

在实际项目中，多个服务往往通过添加`context-path`参数来添加项目前缀和实现服务区分，但是这也造成了该服务明明实际运行正常却在Spring Boot Admin中一直显示Down的状态。

![image-20230523145142443](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230523145142443.png)

原因其实也很简单，参照图片上它所检测的服务状态接口地址我们知道，它的默认`actuator` 检测为：`ip:端口号/actuator`，换句话说由于添加了项目前缀，造成了检测的地址与实际的地址不符，进而`Spring Boot Admin`拿不到服务信息，于是便一直出于Down状态。

实际健康状态端口是加了*项目前缀*的：

![image-20230523145635560](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230523145635560.png)

### 解决方法

通过在在ymal中给注册中心的元数据中配置`Spring Boot Admin`的前缀即可：

```
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          management:
            context-path: ${server.servlet.context-path}/actuator
```

再次重启项目，即可正常访问：

![image-20230523150119832](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230523150119832.png)

**注意**：上述所描述的问题和解决方法是由于配置了`context-path`，没有配置的话无需这样操作，否则由于 `${server.servlet.context-path}`为空造成异常报错！

## 加入 Spring Security 导致的服务401 Unauthorized错误

### 问题描述

在上面所说，`Spring Boot Admin`为了安全起见一般要加入`Spring Security` 。但是也造成了一个问题：我现在要访问我的服务状态接口还得要账号密码作为访问凭证啊。如果没有好好配置，就会造成服务`401 Unauthorized的`错误。

![image-20230523151945760](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230523151945760.png)

### 解决方法：

根据开发文档，从源码上，`Spring Boot Admin`使用了一个名为`BasicAuthHttpHeaderProvider.class`的类来处理验证请求：

```
public class BasicAuthHttpHeaderProvider implements HttpHeadersProvider {
    private static final String[] USERNAME_KEYS = new String[]{"user.name", "user-name", "username"};
    private static final String[] PASSWORD_KEYS = new String[]{"user.password", "user-password", "userpassword"};

    public BasicAuthHttpHeaderProvider() {
    }

    public HttpHeaders getHeaders(Instance instance) {
        String username = getMetadataValue(instance, USERNAME_KEYS);
        String password = getMetadataValue(instance, PASSWORD_KEYS);
        HttpHeaders headers = new HttpHeaders();
        if (StringUtils.hasText(username) && StringUtils.hasText(password)) {
            headers.set("Authorization", this.encode(username, password));
        }

        return headers;
    }
}
```

从这部分源码中可以知道，它从元数据中心的K-V数据中尝试取出名为`user-name`和`password`的数据，然后进行提交验证，验证成功重设返回头，完成验证。那么按照这个思路，只需要在元数据中配置`Spring Security`的用户凭证即可：

```
spring:
    cloud:
        nacos:
          discovery:
            metadata:
              management:
                context-path:  ${server.servlet.context-path}/actuator
              username: ${spring.security.user.name}
              userpassword: ${spring.security.user.password}
```

重启服务，即可解决问题。

# 拓展

在上面所提到的两个问题和解决方法中，都是通过配置元数据的方式解决的。那么上面是元数据呢，它其中的配置信息和配置中心的配置有上面区别和联系呢？

下面是我的一些理解：

- 元数据可以看成一个实例的一些信息，比如版本号。
- 元数据和配置中心的配置数据最大的区别就是元数据是暴露给其他服务的，比如上面两个问题的元数据，都是暴露给Spring Boot Admin用的。

具体对元数据的描述我也没找到更加详细的描述，阿里的开发文档中是结合Dubbo讲的，在Spring的官方文档中我也只看它提了一嘴，没啥详细的案例说明，这些也只是我根据Spring Boot Admin的使用总结出来的。
