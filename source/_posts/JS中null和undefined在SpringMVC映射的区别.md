---
title: JS中null和undefined在Spring MVC映射的区别
tags:
  - Java
  - Spring
  - Spring MVC
  - JS
  - 前端
categories: Java
abbrlink: bef4573
date: 2023-10-17 16:21:20
---

# 问题描述

博主在项目全栈开发的时候，要将前端传过来的表单数据在Spring里进行映射，直接将请求数据转换成Java对象。

前端是通过`Ajax`请求处理的：

```js
// 参数在后端没有意义并且可能会造成映射是转换错误，因此赋予空值
data[0].opTime = null;
data[0].remarks = null;
$.ajax({
         type:"POST",
         url:"/gaitsys/patients/patientsBaseInfo/userDelete",
         data: data[0],//这一行数据
         success : function(data){
			// TODO
         },
         error: function(XMLHttpRequest, textStatus, errorThrown) {
			// TODO
         }
});
```

后端直接通过`Spring MVC`的自动映射解决：

```java
@PostMapping(value="/userDelete")
@ResponseBody
public String userDelete(PatientsBaseInfoVO user){
    // TODO

    return "success";
}
```

代码实现很简单，但是实际启动的测试的时候却报错：

```
2023-10-17 16:31:24.514  WARN 13684 --- [p-nio-80-exec-9] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.validation.BindException: org.springframework.validation.BeanPropertyBindingResult: 1 errors<EOL>Field error in object 'patientsBaseInfoVO' on field 'opTime': rejected value []; codes [typeMismatch.patientsBaseInfoVO.opTime,typeMismatch.opTime,typeMismatch.java.util.Date,typeMismatch]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [patientsBaseInfoVO.opTime,opTime]; arguments []; default message [opTime]]; default message [Failed to convert property value of type 'java.lang.String' to required type 'java.util.Date' for property 'opTime'; nested exception is org.springframework.core.convert.ConversionFailedException: Failed to convert from type [java.lang.String] to type [@javax.persistence.Column java.util.Date] for value []; nested exception is java.lang.IllegalArgumentException]]
```

大概就是参数`opTime`后端拿到了但是是一个~~空值~~，因此自然就转换不了。

前端请求负载：

<img src="https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/10/17_16_36_53_image-20231017163644191.png" alt="image-20231017163644191" style="zoom:150%;" />

# 问题分析

其实问题定位找到了其实问题就解决一半了。既然是前端表单空值造成的，那么回到最初的问题，我为什么要给这几个参数复制为`null`呢?究其原因是我不想这几个参数的值传到后端去，也就是我想要**剔除这几个属性**。那么换个思路，除了~~赋值为`null`可以剔除属性外~~，还有其他的方法吗？

网上找到这篇[文章](https://www.51cto.com/article/661604.html)，上面说了好几种方法，比较推荐的是**给属性赋值为`undefined`**。

这时候博主忽然醒悟，JS里面**`null`和`undefined`有很大区别的**！

> 在涉及类型转换的情况下，`null` 和 `undefined` 表现略有不同。以下是它们在类型转换方面的一些注意事项：
>
> 1. **To String 转换：**
>    - 将 `null` 或 `undefined` 转换为字符串时，它们都会变成字符串 "null" 和 "undefined"。
>
>    ```javascript
>    String(null);        // "null"
>    String(undefined);   // "undefined"
>    ```
>
> 2. **To Number 转换：**
>    - 在数学运算中，`null` 会被转换为 `0`。
>    - `undefined` 转换为 `NaN`（Not a Number）。
>
>    ```javascript
>    Number(null);        // 0
>    Number(undefined);   // NaN
>    ```
>
> 3. **To Boolean 转换：**
>    - 当 `null` 或 `undefined` 被用作条件表达式的时候，它们都被视为 `false`。
>    - 当它们被用作逻辑运算的操作数时，它们也被视为 `false`。
>
>    ```javascript
>    Boolean(null);        // false
>    Boolean(undefined);   // false
>          
>    if (null || undefined) {
>        // 这个条件不会被执行
>    }
>    ```
>
> 总体而言，虽然 `null` 和 `undefined` 在某些方面有相似之处，但在类型转换中它们的行为是有差异的。在使用时，了解它们在不同上下文中的行为可以帮助你更好地处理 JavaScript 中的类型转换。

也就是说值为`null`的变量会自动转换类型，那也就是说，*我给后端传一个值为`null`的变量，它实际并不会转换成我期待的Java里面的`null`，反而会转换成一些类型的默认值，比如`String`类型他就会转换为一个空串，而不是`null`!*

那么也就很好解释报错了，后端接受到一个空串，空串尝试转换为日期类型，显然这是无法完成转换的，于是就直接报错！

# 问题解决

找到原因其实就很好解决了：

```js
// 参数在后端没有意义并且可能会造成映射是转换错误，因此赋予空值
data[0].opTime = undefined;
data[0].remarks = undefined;
$.ajax({
         type:"POST",
         url:"/gaitsys/patients/patientsBaseInfo/userDelete",
         data: data[0],//这一行数据
         success : function(data){
			// TODO
         },
         error: function(XMLHttpRequest, textStatus, errorThrown) {
			// TODO
         }
});
```

直接给原来赋值为`null`变量赋值为`undefined`就可以解决了。

查看请求负载：

![image-20231017165824375](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/10/17_16_58_26_image-20231017165824375.png)

直接表单数据里面没有那几个变量了。

另外其实博主在原来分析的时候还找到器其他的解决方法。也就是不用表单发送请求，而是通过`Json`形式的负载发送请求，这样也能解决上面遇到的问题。

前端发送`JSON`负载的请求：

```js
data[0].opTime = null;
data[0].remarks = null;
$.ajax({
         type:"POST",
         url:"/gaitsys/patients/patientsBaseInfo/userDelete",
         data: JSON.stringify(data[0]), //这一行是JSON形式数据
         contentType: 'application/json; charset=UTF-8',
         success : function(data){
			// TODO
         },
         error: function(XMLHttpRequest, textStatus, errorThrown) {
			// TODO
         }
});
```

那么后端也要稍微修改一下，加个`@RequestBody`的注解

```java
@PostMapping(value="/userDelete")
@ResponseBody
public String userDelete(@RequestBody PatientsBaseInfoVO user){
    // TODO

    return "success";
}
```
