---
title: Java Web项目学习踩坑
tags:
  - Java
  - JavaWeb
  - Tomcat
categories: Java
cover: /img/cover5.png
abbrlink: ec4db7f6
date: 2022-07-20 03:50:54
---



# Failed to parse configuration class [com.echo.config.SpringConfig]; nested exception is java.io.FileNotFoundException: Could not open ServletContext resource [/]的解决方法

lz今天练习SSM项目的时候遇到一个无语的报错:

```
Failed to parse configuration class [com.echo.config.SpringConfig]; nested exception is java.io.FileNotFoundException: Could not open ServletContext resource [/jdbc.properties]
```

部署tomcat找不到resource下的文件，后面网上一搜，发现是部署时路径会有变化，于是便把`SpringConfig`里面的 

`@PropertySource({"jdbc.properties"})`

改成了:

`@PropertySource({"classpath : jdbc.properties"})` 


结果运行还是报错，这就让我百思不得其解了，后面忽然脑袋一清醒，又改成:

`@PropertySource({"classpath:jdbc.properties"})` 

即去掉引号中间习惯性敲的空格，完美解决问题，但是自己都整无语了。
