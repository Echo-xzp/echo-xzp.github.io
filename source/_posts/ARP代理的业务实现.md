---
title: ARP代理的业务实现
keywords: [ARP,免费ARP,Tire树,计算机网络]
date: 2025-05-18 15:22:19
tags: [ARP,免费ARP,Tire树, 计算机网络]
categories: 计算机网络
cover:
description:
---

# 背景

定制需求需要实现NAT规则上，SNAT的转换源IP地址，DNAT的目的IP地址，能够实现自动响应ARP请求。

# ARP

## ARP代理

ARP代理（**ARP Proxy**）是一种网络技术，用于在两个物理上分离的网络之间转发ARP请求，从而实现跨网段通信。它的核心作用是**代替某个IP地址回应ARP请求**，使得请求者可以通过代理找到目标主机。

简而言之，**就是某个主机收到要代理的IP的ARP请求包时，直接根据规则响应自己的MAC地址，哪怕实际上本机上没有这个IP**

## ARP数据包

一个标准 ARP 请求包如下：

```
arduino复制编辑以太网帧头（Ethernet Header）: 14 字节
ARP 报文（ARP Payload）: 28 字节
```

以太网帧头（Ethernet Header）

| 字段         | 长度（字节） | 说明                      |
| ------------ | ------------ | ------------------------- |
| 目标MAC地址  | 6            | FF:FF:FF:FF:FF:FF（广播） |
| 源MAC地址    | 6            | 发送主机的MAC地址         |
| 类型（Type） | 2            | `0x0806` 表示 ARP 协议    |



ARP 报文结构

| 字段            | 长度 | 内容说明               |
| --------------- | ---- | ---------------------- |
| 硬件类型        | 2    | `0x0001`（Ethernet）   |
| 协议类型        | 2    | `0x0800`（IPv4）       |
| 硬件地址长度    | 1    | `6`（MAC地址长度）     |
| 协议地址长度    | 1    | `4`（IPv4地址长度）    |
| 操作码          | 2    | `0x0001`（1 表示请求） |
| 发送方 MAC 地址 | 6    | 请求者的 MAC 地址      |
| 发送方 IP 地址  | 4    | 请求者的 IP 地址       |
| 目标 MAC 地址   | 6    | 0x00...00（未知）      |
| 目标 IP 地址    | 4    | 要查询的 IP 地址       |

### 请求包

假设：

- 主机A（发送者）：MAC = `aa:bb:cc:dd:ee:ff`，IP = `192.168.1.10`
- 要查询的IP：`192.168.1.20`

ARP 请求包（简化十六进制）：

```nginx
# 目标MAC地址（广播）
AA BB CC DD EE FF    # 源MAC地址
08 06                # 类型：ARP

00 01                # 硬件类型：Ethernet
08 00                # 协议类型：IPv4
06                   # 硬件地址长度：6
04                   # 协议地址长度：4
00 01                # 操作码：请求

AA BB CC DD EE FF    # 发送方MAC
C0 A8 01 0A          # 发送方IP（192.168.1.10）
00 00 00 00 00 00    # 目标MAC（未知）
C0 A8 01 14          # 目标IP（192.168.1.20）
```

### 响应包

实际也是ARP数据包，和请求包内容上有点出入：

区别在于：

- **目标MAC不再是广播**
- **操作码变为 0x0002**
- **目标MAC/IP 是原请求者的地址**
- **目标MAC 字段为请求者的MAC，发送者MAC为被查询者的地址**

例如：

```nginx
# 目标MAC地址（原请求者）
11 22 33 44 55 66    # 源MAC地址（192.168.1.20 所属设备）
08 06                # 类型：ARP

00 01                # 硬件类型：Ethernet
08 00                # 协议类型：IPv4
06                   # 硬件地址长度
04                   # 协议地址长度
00 02                # 操作码：响应

11 22 33 44 55 66    # 发送方MAC
C0 A8 01 14          # 发送方IP（192.168.1.20）
AA BB CC DD EE FF    # 目标MAC（原请求者）
C0 A8 01 0A          # 目标IP（192.168.1.10）
```

## 免费ARP/GARP

**Gratuitous ARP** 是指主机主动发送的一种 **无需请求响应的 ARP 报文**，其目的是：

- **声明或更新自己的IP和MAC绑定关系**
- **检测是否有IP冲突**
- **通知其他主机更新ARP缓存**

------

## 🧩 免费 ARP 的特点

| 特性           | 描述                                                     |
| -------------- | -------------------------------------------------------- |
| 非请求响应模型 | 它是主动发送的，无需其他主机发送ARP请求                  |
| 请求包格式     | 操作码是“ARP 请求”（`Opcode = 1`），但目标IP地址就是自己 |
| 响应包格式     | 有时是“ARP 响应”（`Opcode = 2`），但通常不期待回应       |
| 目标 MAC       | 可以是广播地址（FF:FF:FF:FF:FF:FF）或特定主机            |



------

## 📦 免费 ARP 的数据包内容

以 IPv4 为例，假设主机的 IP 是 `192.168.1.10`，MAC 是 `aa:bb:cc:dd:ee:ff`：

### 示例 1：GARP 请求包（最常见）

| 字段              | 值                                 |
| ----------------- | ---------------------------------- |
| 操作码            | `0x0001`（ARP 请求）               |
| 发送方 IP/MAC     | `192.168.1.10 / aa:bb:cc:dd:ee:ff` |
| 目标 IP/MAC       | `192.168.1.10 / 00:00:00:00:00:00` |
| 以太网目标MAC地址 | `FF:FF:FF:FF:FF:FF`（广播）        |



主机向网络广播：“我就是 192.168.1.10，我的 MAC 是 aa:bb:cc:dd:ee:ff”。

------

## 🎯 免费 ARP 的用途

| 场景                 | 描述                                                       |
| -------------------- | ---------------------------------------------------------- |
| IP冲突检测           | 系统初始化时发送GARP，若收到回应则说明IP冲突               |
| 动态更新ARP缓存      | 通知网络中其他主机“我的MAC变了”或“我上线了”                |
| 高可用场景（如VRRP） | 主/备切换时，新主设备发GARP更新网关ARP缓存                 |
| 网络负载均衡/漂移IP  | 虚拟IP在不同服务器之间漂移时，需要GARP更新网络上的绑定关系 |

# 业务实现

## 前提

原有业务上已经有ARP包的处理了，包括ARP包的构造，接受和响应。对于我现在的需求来说我要做的实际就是插入现在的ARP响应规则。

对于响应规则的细节其实也不是很重要，简单的逻辑代码而已。重要的时如何**知道这个请求IP的ARP包要不要响应**。

## 方案

参考之前的实现，发现之前的实现实际是用`Trie树`来实现的，也就是配置下发的时候将用户所有的配置插入到一个`Trie树`中，这个树启动进程的时候就存在在内存中，现在响应的时候直接去这个树中查找有没有这个IP，有的话直接走之后的响应逻辑，没有的话直接丢包处理。

## 方案思考

从描述上来说其实就是实现一个很重要的地方就是判断IP*在不在*规则中。刚开始做到这个需求时，没看之前的实现，博主默认想到了`布隆过滤器`，但转念一想，`布隆过滤器`的特性是*有的时候可能有，没有的时候一定没有*。而我现在实际是判断*一定有*吗，实际上还是需要最终有个兜底的查询，所以实际只能说做优化处理，在查询`Trie树`前先查一遍布隆过滤器，提升一下性能。

或者一个最简单和愚蠢的方法，直接用个`哈希表`把所有要代理的IP都存起来，这样复杂度岂不是很低？但是实际上业务场景中这个代理是在之前的`NAT`规则上增加的功能，常见的情况就是`NAT`的IP是海量的，这样处理必然导致内存消耗巨量。

## Trie树

Trie 树（也叫 **字典树** 或 **前缀树**）是一种高效的树形数据结构，专门用于处理字符串集合。它的核心思想是利用**公共前缀**来减少字符串比较，从而提高查询效率。

### **Trie 树的基本结构**

1. **节点（Node）**：
   - 每个节点代表一个字符。
   - 根节点通常为空，不存储任何字符。
   - 每个节点可以有多个子节点，分别对应不同的字符。
2. **边（Edge）**：
   - 从根节点到某个节点的路径表示一个字符串。
   - 每条边连接两个节点，代表一个字符的延续。
3. **标志位（End Flag）**：
   - 用于标记某个节点是否是一个完整单词的结尾

### 实现

```go
package main

import (
    "fmt"
)

// TrieNode 节点
type TrieNode struct {
    children map[byte]*TrieNode
    isEnd    bool
}

// Trie 树结构
type Trie struct {
    root *TrieNode
}

// NewTrieNode 创建节点
func NewTrieNode() *TrieNode {
    return &TrieNode{
        children: make(map[byte]*TrieNode),
        isEnd:    false,
    }
}

// Constructor 创建 Trie
func Constructor() *Trie {
    return &Trie{root: NewTrieNode()}
}

// Insert 插入 IP 地址（字符串形式）
func (t *Trie) Insert(ip string) {
    node := t.root
    for i := 0; i < len(ip); i++ {
        ch := ip[i]
        if _, ok := node.children[ch]; !ok {
            node.children[ch] = NewTrieNode()
        }
        node = node.children[ch]
    }
    node.isEnd = true
}

// Search 查找完整 IP 是否存在
func (t *Trie) Search(ip string) bool {
    node := t.root
    for i := 0; i < len(ip); i++ {
        ch := ip[i]
        if _, ok := node.children[ch]; !ok {
            return false
        }
        node = node.children[ch]
    }
    return node.isEnd
}

// StartsWith 判断是否有 IP 以某前缀开头
func (t *Trie) StartsWith(prefix string) bool {
    node := t.root
    for i := 0; i < len(prefix); i++ {
        ch := prefix[i]
        if _, ok := node.children[ch]; !ok {
            return false
        }
        node = node.children[ch]
    }
    return true
}

// 示例
func main() {
    trie := Constructor()
    trie.Insert("192.168.1.1")
    trie.Insert("192.168.1.2")
    trie.Insert("10.0.0.1")

    fmt.Println(trie.Search("192.168.1.1"))   // true
    fmt.Println(trie.Search("192.168.1.3"))   // false
    fmt.Println(trie.StartsWith("192.168"))   // true
    fmt.Println(trie.StartsWith("172."))      // false
}
```

示意图：

```
(root)
├── '1'
│   └── '9'
│       └── '2'
│           └── '.'
│               └── '1'
│                   └── '6'
│                       └── '8'
│                           └── '.'
│                               └── '1'
│                                   └── '.'
│                                       ├── '1' (isEnd)
│                                       └── '2' (isEnd)
└── '1'
    └── '0'
        └── '.'
            └── '0'
                └── '.'
                    └── '0'
                        └── '.'
                            └── '1' (isEnd)

```

### 改进

实际现在的情况是：要代理的IP不只是**单个IP**，还包括**子网类型**（1.1.1.0/24）和**范围IP类型**（1.1.1.1-2.2.2.2），之前的树结构就不行了。现在在之前结构的基础上，再实现自己的一个逻辑：*增加一个标记节点，比如1.1.1.0/24，，在24层的前缀时，插入一个标记节点，后续再插入的时候检查到有标记节点就不再插入，查找同理；对于范围IP，可以看作多个子网IP的集合，插入时同理*

#### 📂 节点结构新增字段：

```go
TrieNode struct {
    children map[byte]*TrieNode
    isEnd    bool    // 精确IP结束
    isCIDR   bool    // 是否是CIDR掩码边界节点
}
```

#### 插入逻辑说明

1. **插入单个 IP**（如 `1.1.1.1`）：

- 按字符或按 octet 分层插入。
- 遇到中间已有 `isCIDR = true` 节点，提前终止插入。
- 否则，插入到末尾，并设置 `isEnd = true`。

2. **插入 CIDR 段**（如 `1.1.1.0/24`）：

- 转为二进制，保留前 24 位，插入对应节点。
- 在第 24 层打上 `isCIDR = true` 标记。
- 后续插入中如遇此节点，停止插入，表示已有覆盖。

3. **插入 IP 范围**（如 `1.1.1.1 - 2.2.2.2`）：

- 将范围划分成 **最少数量的 CIDR 段**（可以使用现成算法，如 [ip-ranges-to-cidr](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_blocks))。
- 将每个 CIDR 段按上述方式插入。

------

#### 🔍 查询逻辑说明

- 对某个 IP 查询：
  - 从根节点逐层匹配字符。
  - 过程中若遇 `isCIDR == true`，说明该 IP 已被某段覆盖，返回 true。
  - 若能完整匹配一个 `isEnd == true` 的节点，也返回 true。
  - 否则返回 false。

####  插入逻辑伪代码示意（CIDR）

```go
func (t *Trie) InsertCIDR(cidr string) {
    bits := getFirstNBitsFromCIDR(cidr) // 返回前 n 位的二进制字符串
    node := t.root
    for i := 0; i < len(bits); i++ {
        if node.isCIDR {
            return // 已被更大范围覆盖，跳过
        }
        bit := bits[i]
        if _, ok := node.children[bit]; !ok {
            node.children[bit] = NewTrieNode()
        }
        node = node.children[bit]
    }
    node.isCIDR = true // 在前缀边界标记
}
```

#### 图示

```
(root)
├── 0
│   └── ...  // 假设其他非 1 开头的 IP
└── 1
    └── 00000001 (1)  → 1
        └── 00000001 (1)  → 1
            └── 00000001 (1)  → 1
                ├── 00000000 (0)  → 1.1.1.0/24
                │   └── (isCIDR) ✅
                ├── 00000001 (1)  → 1.1.1.1
                │   └── (isEnd) ✅
                └── 00000010 (2)  → 1.1.1.2（插入时被上层 CIDR 拦截）
            └── 00000010 (2)  → 1.1.2.0/24
                └── (isCIDR) ✅

```

#### 进一步改进

之前的插入的时候直接插入到最大深度了，在IP冲突比较少的情况下，可以这样子处理：**每次插入的时候都倾向往少的层次插入，只有存在前缀冲突时，才继续往更深的层次插入**。当然对此查找时，也要进行**二次确认**，确认当前是不是实际的IP。

### 其他

在之前说，其实还要处理**免费ARP**，主要就是设备**主备切换**时，MAC必然也会切换，这个时候就要发起免费ARP，刷新广播域里面设备的**ARP邻居表**，这里感觉做的就比较粗糙，直接将代理的IP记录下来，发送的时候直接遍历一遍发送了。

其实这里有个潜在的问题，代理的IP如果时海量的，那某个时间段发生主备切换，必然也会发送海量的免费ARP包，主机到是能够承受，但是同一广播域里面的性能比较差的设备不就直接被打爆了？现在的处理就是做了免费ARP数目限制，也算是种规避手段了吧。

# 总结

实际上业务处理还有很多细节上的问题，中间也比较波折吧，但是测试结果上来看功能起码是没有问题了。、

主要还是Trie树上的改动，实际上还加了一个`Bitmap`来记录当前有没有命中规则，插入和查找的逻辑更是有点出入，具体实现懒得讲了，其实大致不差吧。
