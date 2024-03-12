---
title: 解决Spring容器自调引起的依赖冲突
tags:
  - Spring
  - Spring 事务
categories: Java
abbrlink: 13af8a62
date: 2023-04-10 19:57:17
---

lz今天在处理Spring事务的时候遇到一个有意思的坑。

大概就是我在业务层中一个Service要调用自身的其他方法，而它本身又开启了Spring声明式事务，很明显，如果直接用this执行其他要调用的方法，将会导致Spring事务失效（原因：事务本质是Aop，而Aop通过代理的方式完成，this指向会破坏代理关系从而导致事务失效）。

解决自调引起的事务失效大概有三种方法：
<ul>
 	<li>容器中注入容器本身---但是这会引起依赖循环，也就是后面要讲的。</li>
 	<li>采用AopContext获取当前的代理对象。</li>
 	<li>使用Spring的编程事务</li>
</ul>
于是lz采取最简便的第一种方式，但是运行便遇到了依赖循环的报错：

```
┌─────┐
 mediaFileServiceImpl defined in file [D:\MyCode\JavaCloud\xuecheng-plus\xuecheng-plus-media\xuecheng-plus-media-biz\target\classes\work\echoes\media\biz\service\impl\MediaFileServiceImpl.class]
└─────┘
```

自然而然的想到了高版本Spring默认关闭了依赖循环，于是在配置中打开：

```
spring:
	main:
		allow-bean-definition-overriding: true
```

但是运行依旧是同样的报错，那么为什么不能解决依赖循环呢？

首先从源头开始，Spring采用三级缓存的方式来解决依赖循环：
<ul class="ul-level-0">
 	<li>singletonObjects, 一级缓存</li>
 	<li>earlySingletonObjects, 二级缓存</li>
 	<li>singletonFactories 三级缓存</li>
</ul>
容器创立流程（<a href="https://cloud.tencent.com/developer/article/2019359">参考地址</a>）：
<ol class="ol-level-0">
 	<li>对象A要创建到Spring<a href="https://cloud.tencent.com/product/tke?from=20065&amp;from_column=20065" target="_blank" rel="noopener" data-text-link="443_2019359" data-from="20065">容器</a>中，从一级缓存singletonObject获取A，不存在，开始实例化A，最终在三级缓存singletonObjectFactory添加(A，A的函数式接口创建方法)，这时候A有了自己的内存地址</li>
 	<li>设置属性B，B也从一级缓存singletonObject获取B，不存在，开始实例化B，最终在三级缓存singletonObjectFactory添加(B，B的函数式接口创建方法)，这时候B有了自己的内存地址</li>
 	<li>B中开始给属性A赋值，此时会找到三级缓存中的A，并将A放入二级缓存中。删除三级缓存</li>
 	<li>B初始化完成，从三级缓存singletonObjectFactory直接put到一级缓存singletonObject，并删除二级和三级缓存的自己</li>
 	<li>A成功得到B，A完成初始化动作，从二级缓存中移入一级缓存，并删除二级和三级缓存的自己</li>
 	<li>最终A和B都进入一级缓存中待用户使用</li>
</ol>
那么源头可能就是我容器注入的方式不对了。回过头来看，我通过**lombok**的注解 `@RequiredArgsConstructor` 配合 final属性，一键完成注入，而这个注解本身就是通过<strong>构造器注入</strong>的方式注入，那么结合上面的分析，不难发现构造器不能构造自身，<strong>于是导致了无法在第三级的缓存总生成未完全的容器对象，进而导致了三级缓存机制无法解决依赖循环问题。</strong>

了解到根本原因，那么好修改了，只要把这种构造器注入的方式改为setter注入或者`@autowired`注入，便可以解决问题。
