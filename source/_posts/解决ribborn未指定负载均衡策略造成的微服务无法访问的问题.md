---
title: 解决ribborn未指定负载均衡策略造成的微服务无法访问的问题
date: 2023-03-22 21:12:44
tags: [Java,JavaWeb,"Spring CLoud","Spring Cloud Alibaba",nacos,ribborn]
categories: Java
---

lz在docker部署微服务的时候遇到了不少问题。

首先项目在本地能够正常运行，但是当我在docker中把容器成功运行后，部署的前端接口连接的时候却报connect refuse的错误。首先我就想到是否是我的前端项目部署存在问题，因为在我的测试下，每个微服务的接口都是正常的。于是我不在把前端项目放nginx里面运行，我怀疑可能是nginx反向代理哪些地方我没有配置好。但当我重新运行在node里面，还是不能够运行。所以到底错误在哪里呢？因为在docker与服务器里面不好测试，我又重新在本地部署了一遍前端，但是还是无法运行。

那么是否可以猜测前端项目没有问题呢？我单纯把所有接口通过gateway测试一遍，这时我忽然发现通过gateway返回的json数据正是我在前端所看到的报错。那么大概可以将错误锁定在gateway上面了。但是我在本地上运行gateway却又没有这个报错，到底如何导致了这个问题呢？于是我继续查看gateway的运行日志，这个时候我忽然发现了问题所在：

```
com.netflix.loadbalancer.DynamicServerListLoadBalancer [222] -| Using serverListUpdater PollingServerListUpdater\n","stream":"stdout","time": "2023-03-17T09:37:35.356072646Z"}
8 {"log":"2023-03-17 09:37:35.371 |-INFO [reactor-http-epoll-7] com.netflix.config.ChainedDynamicProperty [115] -| Flipping property: system-service.ribbon.ActiveConnectionsLimit to use NEXT property: niws .loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647\n","stream":"stdout","time":"2023-03-17T09:37:35.372430585Z"}
7 {"log":"2023-03-17 09:37:35.375 |-INFO [reactor-http-epoll-7] com.netflix.loadbalancer.DynamicServerListLoadBalancer [150] -| DynamicServerListLoadBalancer for client system-service initialized: DynamicS erverListLoadBalancer:{NFLoadBalancer:name=system-service,current list of Servers=[localhost:8184],Load balancer stats=Zone stats: {unknown=[Zone:unknown;\u0009Instance count:1;\u0009Active connections coun t: 0;\u0009Circuit breaker tripped count: 0;\u0009Active connections per server: 0.0;]\n","stream":"stdout","time":"2023-03-17T09:37:35.375981402Z"}
```

秉着试一试的态度，我直接将gateway配置中断言的地址写死:

这下接口一下就访问了，错误原来就是`lb://`这里的**ribbon均衡策略**吧。

离谱了嗷，nacos上明明能读到服务ip地址，但是ribbon一负载均衡就变成了localhost/127.0.0.1了呢？这也解释了在本地能够可以通过gateway访问，但是到了服务器docker里面部署就不能够了--容器彼此之间隔离，每个的ip都不同。这个问题也很无语啊，本地阴差阳错的能够运行，不仔细看日志根本发现不了。

找到具体问题所在就好解决问题了，网上一查找，结合我自己的spring项目，我发现原来是自己没有指明ribbon的均衡策略，在nacos配置中心配置加上:

```
app-gateway.ribbon.NFLoadBalancerRuleClassName=com.alibaba.cloud.nacos.ribbon.NacosRule
system-service.ribbon.NFLoadBalancerRuleClassName=com.alibaba.cloud.nacos.ribbon.NacosRule
user-service.ribbon.NFLoadBalancerRuleClassName=com.alibaba.cloud.nacos.ribbon.NacosRule
course-service.ribbon.NFLoadBalancerRuleClassName=com.alibaba.cloud.nacos.ribbon.NacosRule
app-job.ribbon.NFLoadBalancerRuleClassName=com.alibaba.cloud.nacos.ribbon.NacosRule
app-sba.ribbon.NFLoadBalancerRuleClassName=com.alibaba.cloud.nacos.ribbon.NacosRule
```

重启docker容器，成功通过gateway访问到接口。在部署前端，成功完成整个项目的部署。

但是ribbon为何在我没配置策略的时候默认找到本地的服务的呢？我还是不太理解，可能还是微服务项目做的太少了的原因吧。
