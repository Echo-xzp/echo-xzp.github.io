---
title: 解决MySQL版本‘sql_mode=only_full_group_by’引起的查询错误
tags:
  - mysql
  - Java
categories: Java
abbrlink: c0d9da45
date: 2023-05-06 16:49:42
---

博主在部署一个项目后进行测试，发现了一个界面报服务器500错误，于是查看报错误日志：

```
Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'study.user_study.chapter_id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

注意到他所说的是重点是： <em>incompatible with sql_mode=only_full_group_by</em>

大概就是查询语句和sql的模式冲突了。在MySQL5.7后默认加入了<strong>sql_mode=only_full_group_by  </strong>于是对有分组要求的结果必须出现在分组中，而在之前的版本中确实允许这样的操作。

于是在网上查找相关解决方法，并发现了有三种：

- MySQL降低版本。既然是MySQL版本引起的问题，把MySQL降低到5.7的版本就行了。而对于我所部署的项目来说它要求的是5.6.x的版本，我部署的时候直接拉取了MySQL最新的镜像，考虑到项目的稳定性，这个确实是一个可行的方法。但是我的想法一直就是有新事物新特性就要学习和接受，对于个人学习来说用着几年前的技术未免有些固步自封。
- 修改sql_mode。既然是sql_mode引起的问题，那么把他恢复到原来可以接受这种查询语句的模板就行了。修改sql_mode也是有两种方法，一个是直接在数据库中执行sql修改，这种方法快但是MySQL重启之后就会恢复以前的配置；在对应的数据库执行: `SET sql_mode ='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'`; 即可；另外就是直接修改MySQL配置文件就行了 编辑/etc/my.cnf 加入 `sql-mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` 这行就行了。
- 优化sql语句。这个才是最可行的方法，既然mysql默认不支持这种查询方式，自然有它的意义。only_full_groupby模式要明确分组之后select的值应该取哪一个，如果取的字段无法适用于任何聚合函数的话，可以用any_value 但是any_value到底是取得哪一个呢，常用的场景是先排序再分组取第一个。

不过由于我的项目已经部署上去了，再打包部署上服务器就繁琐了一点，最后还是选择了方式二修改配置解决了问题。
