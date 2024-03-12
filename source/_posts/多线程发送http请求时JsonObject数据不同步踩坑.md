---
title: 多线程发送http请求时JsonObject数据不同步踩坑
tags:
  - Java
  - 多线程
  - JsonObject
categories: Java
abbrlink: a8b409a0
date: 2023-07-18 14:38:14
---

# 问题描述

博主在开发微信公众号后台接口的时候，要利用`OkHttpClient`给微信公众号服务器发送批量的请求，如果一个一个发请求这样肯定不行，所以必须采用多线程的形式。发送请求的代码如下：

```
        // 给所有用户发送模板消息
        JSONObject jsonObjectSendInfo = new JSONObject();
        jsonObjectSendInfo.put("data",wechatInfoTemplate);
        jsonObjectSendInfo.put("template_id",TEMPLATE_ID);
        jsonObjectSendInfo.put("url","https://baidu.com");
        AtomicInteger failCount = new AtomicInteger(0);
        CountDownLatch countDownLatch = new CountDownLatch(ids.size());
        ids.forEach(id ->{
           new Thread(()->{
               try {
                   log.info("正在发送模板消息给"+id);
                   JSONObject jsonObject = new JSONObject(jsonObjectSendInfo);
                   jsonObject.put("touser",id);
                   log.error(">>>>>>>>>>>>>>>>>>>>>>>>>>>"+jsonObject);
                   String res = HttpPostJSONUtil.postJson("https://api.weixin.qq.com/cgi-bin/message/template/send?access_token="+ accessToken,jsonObject);
                   Integer errcode = JSONObject.parseObject(res).getInteger("errcode");
                   if (0 != errcode){
                       failCount.getAndIncrement();
                       log.warn("发送消息模板给"+id+"错误!\n"+res);
                   }
                   countDownLatch.countDown();
               } catch (MyException e) {
                   failCount.getAndIncrement();
                   log.error("发送消息模板给"+id+"失败!");
               }
           }).start();
        });

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
```

这样其实很好理解，遍历`ids`列表，根据每个`id`生成不同的请求体，每次开个线程去发请求，同时利用`CountDownLatch`等待所有线程完成，继续之后的处理逻辑。想法很不错，但是当我进行接口调试的时候，发现会给一个用户发多个请求，而其他用户却没有发请求，也就是请求体数据出现了线程同步问题。

进行调试的时候，查看打印的日志，更加让我摸不着头脑了，日志如下：

![image-20230718145013805](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/07/18_14_50_24_image-20230718145013805.png)

也就是说在发送请求之前，`JsonObject`的内容明明会根据`id`的不同而不同，但是当发送请求的时候`JsonObject`这个对象却又变更了。

# 问题解决

博主经过分析，大概觉得是`JsonObject`这个对象是从外面构造过来的，抱着试一试的想法，我直接把赋值语句写在了线程里面。

```
        // 给所有用户发送模板消息
        AtomicInteger failCount = new AtomicInteger(0);
        CountDownLatch countDownLatch = new CountDownLatch(ids.size());
        ids.forEach(id ->{
           new Thread(()->{
               try {
                   log.info("正在发送模板消息给"+id);
                   JSONObject jsonObjectSendInfo = new JSONObject();
                   jsonObjectSendInfo.put("data",wechatInfoTemplate);
                   jsonObjectSendInfo.put("template_id",TEMPLATE_ID);
                   jsonObjectSendInfo.put("url","https://baidu.com");
                   jsonObjectSendInfo.put("touser",id);
                   String res = HttpPostJSONUtil.postJson("https://api.weixin.qq.com/cgi-bin/message/template/send?access_token="+ accessToken,jsonObjectSendInfo);
                   Integer errcode = JSONObject.parseObject(res).getInteger("errcode");
                   if (0 != errcode){
                       failCount.getAndIncrement();
                       log.warn("发送消息模板给"+id+"错误!\n"+res);
                   }
                   countDownLatch.countDown();
               } catch (MyException e) {
                   failCount.getAndIncrement();
                   log.error("发送消息模板给"+id+"失败!");
               }
           }).start();
        });

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
```

再次运行代码，发现问题成功解决了。博主这样解决也只是误打误撞，多次修改代码才解决的，这个时候还不理解问题的产生和怎么这样修改就把问题给解决了。

# 问题分析

于是博主仔细对比前后代码，发现最开始的请求对象是从外部的`jsonObjectSendInfo`构造来的，理论上我已经`new`了一个新对象了，和原来那个对象现在是隔离了的了，于是我打上断点调试验证一下我的想法：

![image-20230718150143274](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/07/18_15_1_45_image-20230718150143274.png)

王德发？外面的对象怎么也被修改了？这两个对象的内容不就是一模一样的吗？

那看看`JsonObject`的构造方法是怎么样的：

![image-20230718150357397](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/07/18_15_3_58_image-20230718150357397.png)

这不就是直接把引用传递过来了，有点蚌埠住了，一般`new`出一个对象，不应该是进行值拷贝吗？两个对象直接完全隔离了吗？感觉阿里这样设计真的是害人精。

于是就好理解问题的产生了：

> 两个线程并发执行，第一个线程和第二个线程都改变了`jsonObject`的值，因而我们从打印的日志上来看就是`jsonObject`是有区别的，但是当发送http请求的时候，第二个线程改变`jsonObject`更慢一点，此时把第一个改变的`id`又改回了第二个`id`的值并发送请求，于是我们就看到了第二个用户接受到了两条消息，第一个用户却一条消息都接受不到的情况。

其实问题的产生还是博主对多线程不够熟悉，遇到这种问题硬是分析了半天才解决，并且最后还是误打误撞解决的；另外也是开发时的所以然思想，要是仔细看看文档也自然没有这种问题的产生了。
