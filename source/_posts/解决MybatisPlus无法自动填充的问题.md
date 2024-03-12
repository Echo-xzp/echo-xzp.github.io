---
title: 解决MybatisPlus无法自动填充的问题
tags:
  - Java
  - Mybatis
  - Mybatis Plus
categories: Java
abbrlink: 729736eb
date: 2023-04-02 18:31:08
---

MP能够通过注解：`@TableField(fill = FieldFill.INSERT)`  给数据对象指定填充类型，并通过注入`MetaObjectHandler` 完成自动填充:

```
@Component
public static class MyFillHandler implements MetaObjectHandler {

@Override
public void insertFill(MetaObject metaObject) {
    // 起始版本 3.3.0(推荐使用)
    this.strictInsertFill(metaObject, "createDate", Date.class, new Date());
}

@Override
public void updateFill(MetaObject metaObject) {
    // 起始版本 3.3.0(推荐)
    this.strictUpdateFill(metaObject, "changeDate", Date.class, new Date());
    }
}
```

但是我今天在使用的时候希望通过自动填充完成数据库修改日志的记录，但是当我实际运行的时候却无法完成。
首先我想到的是这个组件有没有成功注入呢？这点可以通过打断点的方式完成，于是进行Debug调试，发现成功进入断点位置。既然成功注入了，那么为什么没能够成功自动填充呢？
在mp自动填充的时候，要注意两点：

1. 属性的映射要一样，字段属性名是什么，你写的字符串名就应该一样
2. 数据类型必须一样，字段类型如何，在填充的方法里面就应该直接指明字段类型。

有了这两点的基础，我回过头来看，发现自己的实体对象的日志属性才用的是`LocalDateTime`类型，但是在我填充的时候却采用了`Date.class`的类型，类型不一致于是导致了我无法自动填充。
修改为：

```
@Component
public static class MyFillHandler implements MetaObjectHandler {

@Override
public void insertFill(MetaObject metaObject) {
// 起始版本 3.3.0(推荐使用)
this.strictInsertFill(metaObject, "createDate", LocalDateTime.class, LocalDateTime.now());
}

@Override
public void updateFill(MetaObject metaObject) {
// 起始版本 3.3.0(推荐)
this.strictUpdateFill(metaObject, "changeDate", LocalDateTime.class, LocalDateTime.now());
}
```

}

成功解决问题.
