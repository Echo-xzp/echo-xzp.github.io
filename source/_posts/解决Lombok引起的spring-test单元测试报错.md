---
title: 解决Lombok引起的spring-test单元测试报错
tags:
  - Java
  - Java Web
  - Spring
categories: Java
abbrlink: dbd0a823
date: 2023-03-10 11:06:54
---

今天进行Spring单元测试的时候遇到了一个报错：

`org.junit.jupiter.api.extension.ParameterResolutionException: No ParameterResolver registered for parameter`

经过网上查找，说的都是test测试方法**加入了参数**导致的，但是实际上我却并没有给我的test方法加参数啊。

于是只能一层一层的调，我把测试方法全部都注释了，只让他打印一行字符串，但还是没结果，于是继续往上，把所有的注入的bean注释，这次成功打印。

所以分析大概是注入bean的时候造成了异常。而我是通过Lombok中的 `@RequiredArgsConstructor` 注解注入的，把它替换成一般的 `@Autowired`注入就行了。

但是为什么会造成这种现象发生呢。通过对源码的查阅之后发现，<code>@RequiredArgsConstructor</code>注解实际上做了两件事：

1. 添加了一个实例化对象的静态方法；
2. 添加了一个私有构造方法服务于该静态方法，在需要自定义的字段上面加入<code>@NonNull</code>后，这些字段就会按照属性在类中声明的先后顺序作为该方法的形参，同时会对这些字段做非空验证，如果为空就会报空指针异常。

大概就是这个原因所造成的吧。