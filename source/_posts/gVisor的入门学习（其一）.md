---
title: gVisor的入门学习（其一）
keywords:
  - gVisor
  - 容器
  - OCI
  - runsc
  - runc
  - Kubernetes
  - Linux
  - 安全沙箱
tags:
  - gVisor
  - 容器
  - OCI
  - runsc
  - runc
  - Kubernetes
categories: Linux
description: 从容器运行时、OCI 接口到用户态内核语义模拟，梳理 gVisor 的架构定位、安全模型与适用场景。
abbrlink: 24a56b3f
date: 2026-04-25 17:29:56
cover:
---

## 前言

最近在看容器隔离相关的东西，顺手把 `gVisor` 的定位和实现思路过了一遍。刚开始接触时，很容易把它和普通容器运行时、`namespace`、`cgroup`、`eBPF` 这些机制混在一起，所以这里先做一篇入门梳理，重点搞清楚几个问题：

- `gVisor` 到底是什么
- 它和 `runc`、`runsc`、OCI 的关系是什么
- 它为什么能在不引入完整虚拟机的情况下提升隔离性

## 一、gVisor 是什么？

`gVisor` 是 Google 开源的一种容器沙箱技术，它的核心目标不是提升性能，而是：

> 在不使用完整虚拟机的情况下，显著增强容器隔离安全性。

可以用一句话概括：

> `gVisor = 运行在用户态的 Linux 内核语义模拟层`

## 二、gVisor 是不是容器运行时？

从严格定义上看，它属于容器运行时生态的一部分，也实现了 OCI runtime 接口，但它并不是传统意义上的 `runc` 这类运行时。

在 Kubernetes / OCI 体系中，传统容器运行路径大致是这样：

```text
containerd -> OCI runtime -> Linux kernel
```

而 `gVisor` 所在的位置更接近这样：

```text
containerd -> runsc -> gVisor Sentry -> Linux kernel
```

其中：

- `runsc`：`gVisor` 提供的 OCI runtime 入口，可以理解为 `runc` 的替代者
- `Sentry`：`gVisor` 的核心组件，负责在用户态模拟一部分 Linux 内核语义

## 三、gVisor 的核心思想：用户态小内核

`gVisor` 的本质可以理解为：

> 在用户态重新实现一部分 Linux syscall 语义。

它主要做三件事：

### 1. syscall 拦截与模拟

- 捕获应用发起的系统调用
- 在用户态处理其中的大部分逻辑

### 2. 虚拟 OS 语义

- 进程模型，如 PID、`fork`、`exec`
- 文件系统语义，如 VFS
- 网络行为，如 socket 相关接口

### 3. 必要时下沉到宿主机内核

- 内存分配
- 真实 I/O
- 设备访问

也就是说，`gVisor` 不是完全脱离 Linux 内核，而是在 Linux 内核之上再包了一层“用户态内核解释层”。

## 四、gVisor 和 runc 的本质区别

`runc` 是 Linux 容器的标准实现之一，它的特点是直接利用宿主机现成的内核能力来运行容器。

### runc 模式

```text
容器 -> Linux namespace / cgroup -> kernel
```

特点：

- 直接使用 Linux 内核能力
- 性能高
- 安全性高度依赖内核本身的隔离质量

### gVisor 模式

```text
容器 -> gVisor -> Linux kernel
```

特点：

- 多了一层 syscall 解释与转发
- 降低了容器直接接触宿主机内核的范围
- 会有一定性能损耗，但换来了更强的隔离能力

## 五、gVisor 和 eBPF / namespace / cgroup 的关系

很多人会觉得：

> `eBPF + namespace + cgroup` 已经足够做容器隔离了。

这个说法不能算错，但并不完整。因为这些技术更偏向于“约束内核如何处理容器”，而 `gVisor` 是在“用户态重建一层系统语义”。

它们的差别大致可以这样理解：

| 技术 | 本质 |
| --- | --- |
| `eBPF` | 在内核中做过滤、观测与控制 |
| `namespace / cgroup` | 做资源隔离与视图隔离 |
| `gVisor` | 在用户态模拟一部分 syscall 与 OS 语义 |

所以：

- `eBPF` 能做 syscall 过滤、网络控制、安全策略
- `namespace / cgroup` 能做资源隔离
- 但它们并不会替代 Linux 内核本身去“扮演一个内核”

而 `gVisor` 恰恰是在做这件事。

## 六、为什么云厂商仍然需要 gVisor？

即使已经有了 `eBPF`、`namespace`、`cgroup`，云厂商仍然会使用 `gVisor`，原因通常有下面几类。

### 1. 内核攻击面依然存在

普通容器本质上仍然共享宿主机的 Linux 内核。一旦内核漏洞被利用，影响的往往不是单个容器，而是整台宿主机上的所有容器。

### 2. 控制不等于重建语义

`eBPF` 很擅长做过滤、限制和观测，但它不能：

- 重建进程模型
- 虚拟文件系统语义
- 模拟一套更受控的 OS 行为边界

### 3. 多租户不可信环境的需求

典型场景包括：

- Serverless，如 Cloud Run
- 在线代码执行平台
- SaaS 多租户系统

这些场景里最重要的前提是：

> 容器里的代码不一定可信。

在这种情况下，普通容器的隔离强度往往不够，而完整虚拟机的成本又偏高，于是 `gVisor` 就成了中间路线。

## 七、gVisor 的安全模型本质

`gVisor` 的安全模型可以概括成一句话：

> 把攻击面从宿主机内核前移到用户态。

对比来看：

### runc 模型

```text
攻击者 -> Linux kernel
```

### gVisor 模型

```text
攻击者 -> gVisor Sentry -> Linux kernel
```

攻击者首先打到的是 `gVisor` 的用户态实现，而不是直接进入高权限的宿主机内核路径。这并不意味着绝对安全，但确实缩小了直接暴露给容器的内核攻击面。

## 八、runsc 不是 runc + gVisor

`gVisor` 的 runtime 实现叫 `runsc`，它不是“在 `runc` 外面再包一层 `gVisor`”，而是 `gVisor` 自己实现的一套 OCI runtime。

正确理解应该是：

- `runc`：直接基于 Linux 内核运行容器
- `runsc`：通过 `gVisor` 的用户态内核来运行容器

所以 `runsc` 不是“增强版 runc”，而是另一条实现路径。

## 九、OCI 为什么允许 gVisor / Kata 这类实现？

OCI 标准规定的重点其实是容器生命周期接口，比如：

- `create`
- `start`
- `kill`
- `delete`

但 OCI 并不强制规定：

- 一定要直接使用 Linux kernel
- 一定要基于 `namespace`
- 一定不能引入 VM

所以同一个 OCI 体系下，完全可以同时容纳多种实现：

- `runc`：kernel-based
- `gVisor`：user-space kernel
- `Kata Containers`：VM-based

这也是为什么在 Kubernetes 生态里，我们能看到多种不同隔离模型并存。

## 十、gVisor 的本质总结

可以从三个层面来理解 `gVisor`：

### 1. 架构层

用户态 syscall 解释器 + sandbox runtime。

### 2. 安全层

降低 Linux kernel attack surface。

### 3. 云原生层

面向多租户、不可信代码执行场景的隔离方案。

## 十一、一句话总结

> `gVisor` 是一种用户态 Linux 内核子集实现，它通过拦截并模拟 syscall，在不使用完整虚拟机的情况下，为容器提供更强的安全隔离能力。

## 十二、核心对比总结

| 方案 | 隔离方式 | 性能 | 安全 |
| --- | --- | --- | --- |
| `runc` | Linux kernel namespace | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| `gVisor` | 用户态内核模拟 | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| `Kata` | VM 隔离 | ⭐⭐ | ⭐⭐⭐⭐⭐ |

## 十三、最终理解

`gVisor` 并不是“更高级的容器”，而是位于“容器共享内核”和“虚拟机完整隔离”之间的一种折中安全层。

它的核心价值不在于替代 `runc`，而在于：

> 在不可信计算场景中，把最敏感的内核风险尽量隔离到用户态边界之外。
