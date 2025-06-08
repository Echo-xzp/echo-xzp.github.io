---
title: 浅谈NPTv6的实现
keywords: [NPTv6,SNAT,DNAT,NAT,计算机网络]
date: 2025-06-08 15:31:27
tags: [NPTv6,SNAT,DNAT,NAT,计算机网络]
categories: 计算机网络
cover:
description:
---

# 引入

最近有个需求，在之前的`NAT`实现上，实现`NPTv6`。这需求我连名字都没有听过啊，直接两眼一黑。但是后面查阅相关资料之后，发现其实也就是`ipv6`的一个前缀转换，类似`nat66`了吧。实现起来也相对简单了。

# RFC

了解`NPTv6`，直接看`RFC`文档就知道了，[RFC6296](https://datatracker.ietf.org/doc/html/rfc6296)

根据文档，大概可以知道：

# 工作原理

- NPTv6 是一种**无状态**的前缀转换，转换规则是：
   **内部前缀 ↔ 外部前缀**，比如 `2001:db8:1::/48` ↔ `2001:db8:2::/48`。
- 它通过对地址中**主机部分**保持不变，仅转换前缀（网络部分）；
- 为了保持校验和一致，可能还需要调整 IPv6 报文中的校验和（使用调整和算法）。

------

# 🧾 关键特点

| 特性             | 说明                                       |
| ---------------- | ------------------------------------------ |
| 无状态           | 不维护每个连接状态，只需要知道前缀映射规则 |
| 一对一映射       | 避免传统 NAT 的多对一、端口复用问题        |
| 保留端到端可达性 | 如果两端都知道映射规则，通信仍可达         |
| 校验和修正       | 必须调整 IPv6 报文的伪首部校验和           |

# 调整计算

`NPTv6`并不是简单的前缀替换，实际还涉及到一个`调整/adjustment`值的计算。

详见[RFC6296](https://datatracker.ietf.org/doc/html/rfc6296)中的定义：

> 3.4 NPTv6 with a /48 or Shorter Prefix
>
> When an NPTv6 Translator is configured with internal and external prefixes that are 48 bits in length (a /48) or shorter, the adjustment MUST be added to or subtracted from bits 48..63 of the address.
>
> This mapping results in no modification of the Interface Identifier (IID), which is held in the lower half of the IPv6 address, so it will not interfere with future protocols that may use unique IIDs for node identification. NPTv6 Translator implementations MUST implement the /48 mapping.
>
> 3.5 NPTv6 with a /49 or Longer Prefix
>
> When an NPTv6 Translator is configured with internal and external prefixes that are longer than 48 bits in length (such as a /52, /56, or /60), the adjustment must be added to or subtracted from one of the words in bits 64..79, 80..95, 96..111, or 112..127 of the address.  While the choice of word is immaterial as long as it is consistent, these words MUST be inspected in that sequence and the first that is not initially 0xFFFF chosen, for consistency's sake. 
>
> NPTv6 Translator implementations SHOULD implement the mapping for longer prefixes

实际也就是前缀长度在`48`及以下的时候，`adjustment`是加到IP第**4**个16位上（实际就是子网位）的，而大于`48`的时候，是找到`IID`中第一个不是`全F`的十六位。

# 例子

完整的`IPV6`的16位的划分如下

```
  0 15 16   31 32   47 48   63 64   79 80   95 96  111 112    127
 +-------+-------+-------+-------+-------+-------+-------+-------+
 |     Routing Prefix    | Subnet|   Interface Identifier (IID)  |
 +-------+-------+-------+-------+-------+-------+-------+-------+
```

按照RFC上给的示范，如下：

> For the network shown in Figure 1, the Internal Prefix is FD01:0203:0405:/48, and the External Prefix is 2001:0DB8:0001:/48.
>
> If a node with internal address FD01:0203:0405:0001::1234 sends an outbound datagram through the NPTv6 Translator, the resulting external address will be 2001:0DB8:0001:D550::1234.  The resulting address is obtained by calculating the checksum of both the internal and external 48-bit prefixes, subtracting the internal prefix from the external prefix using one's complement arithmetic to calculate the "adjustment", and adding the adjustment to the 16-bit subnet field (in this case, 0x0001).
>
> To show the work:
>
> The one's complement checksum of FD01:0203:0405 is 0xFCF5.  The one's complement checksum of 2001:0DB8:0001 is 0xD245.  Using one's complement arithmetic, 0xD245 - 0xFCF5 = 0xD54F.  The subnet in the original datagram is 0x0001.  Using one's complement arithmetic,
>
> 0x0001 + 0xD54F = 0xD550.  Since 0xD550 != 0xFFFF, it is not changed to 0x0000.
>
> So, the value 0xD550 is written in the 16-bit subnet area, resulting in a mapped external address of 2001:0DB8:0001:D550::1234.

# 内核实现

实际`NPTv6`在内核上就是有实现了的，查看内核实现：[ip6t_NPT.c](https://github.com/torvalds/linux/blob/master/net/ipv6/netfilter/ip6t_NPT.c)

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * Copyright (c) 2011, 2012 Patrick McHardy <kaber@trash.net>
 */

#include <linux/module.h>
#include <linux/skbuff.h>
#include <linux/ipv6.h>
#include <net/ipv6.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv6.h>
#include <linux/netfilter_ipv6/ip6t_NPT.h>
#include <linux/netfilter/x_tables.h>

static int ip6t_npt_checkentry(const struct xt_tgchk_param *par)
{
	struct ip6t_npt_tginfo *npt = par->targinfo;
	struct in6_addr pfx;
	__wsum src_sum, dst_sum;

	if (npt->src_pfx_len > 64 || npt->dst_pfx_len > 64)
		return -EINVAL;

	/* Ensure that LSB of prefix is zero */
	ipv6_addr_prefix(&pfx, &npt->src_pfx.in6, npt->src_pfx_len);
	if (!ipv6_addr_equal(&pfx, &npt->src_pfx.in6))
		return -EINVAL;
	ipv6_addr_prefix(&pfx, &npt->dst_pfx.in6, npt->dst_pfx_len);
	if (!ipv6_addr_equal(&pfx, &npt->dst_pfx.in6))
		return -EINVAL;

	src_sum = csum_partial(&npt->src_pfx.in6, sizeof(npt->src_pfx.in6), 0);
	dst_sum = csum_partial(&npt->dst_pfx.in6, sizeof(npt->dst_pfx.in6), 0);

	npt->adjustment = ~csum_fold(csum_sub(src_sum, dst_sum));
	return 0;
}

static bool ip6t_npt_map_pfx(const struct ip6t_npt_tginfo *npt,
			     struct in6_addr *addr)
{
	unsigned int pfx_len;
	unsigned int i, idx;
	__be32 mask;
	__sum16 sum;

	pfx_len = max(npt->src_pfx_len, npt->dst_pfx_len);
	for (i = 0; i < pfx_len; i += 32) {
		if (pfx_len - i >= 32)
			mask = 0;
		else
			mask = htonl((1 << (i - pfx_len + 32)) - 1);

		idx = i / 32;
		addr->s6_addr32[idx] &= mask;
		addr->s6_addr32[idx] |= ~mask & npt->dst_pfx.in6.s6_addr32[idx];
	}

	if (pfx_len <= 48)
		idx = 3;
	else {
		for (idx = 4; idx < ARRAY_SIZE(addr->s6_addr16); idx++) {
			if ((__force __sum16)addr->s6_addr16[idx] !=
			    CSUM_MANGLED_0)
				break;
		}
		if (idx == ARRAY_SIZE(addr->s6_addr16))
			return false;
	}

	sum = ~csum_fold(csum_add(csum_unfold((__force __sum16)addr->s6_addr16[idx]),
				  csum_unfold(npt->adjustment)));
	if (sum == CSUM_MANGLED_0)
		sum = 0;
	*(__force __sum16 *)&addr->s6_addr16[idx] = sum;

	return true;
}

static struct ipv6hdr *icmpv6_bounced_ipv6hdr(struct sk_buff *skb,
					      struct ipv6hdr *_bounced_hdr)
{
	if (ipv6_hdr(skb)->nexthdr != IPPROTO_ICMPV6)
		return NULL;

	if (!icmpv6_is_err(icmp6_hdr(skb)->icmp6_type))
		return NULL;

	return skb_header_pointer(skb,
				  skb_transport_offset(skb) + sizeof(struct icmp6hdr),
				  sizeof(struct ipv6hdr),
				  _bounced_hdr);
}

static unsigned int
ip6t_snpt_tg(struct sk_buff *skb, const struct xt_action_param *par)
{
	const struct ip6t_npt_tginfo *npt = par->targinfo;
	struct ipv6hdr _bounced_hdr;
	struct ipv6hdr *bounced_hdr;
	struct in6_addr bounced_pfx;

	if (!ip6t_npt_map_pfx(npt, &ipv6_hdr(skb)->saddr)) {
		icmpv6_send(skb, ICMPV6_PARAMPROB, ICMPV6_HDR_FIELD,
			    offsetof(struct ipv6hdr, saddr));
		return NF_DROP;
	}

	/* rewrite dst addr of bounced packet which was sent to dst range */
	bounced_hdr = icmpv6_bounced_ipv6hdr(skb, &_bounced_hdr);
	if (bounced_hdr) {
		ipv6_addr_prefix(&bounced_pfx, &bounced_hdr->daddr, npt->src_pfx_len);
		if (ipv6_addr_cmp(&bounced_pfx, &npt->src_pfx.in6) == 0)
			ip6t_npt_map_pfx(npt, &bounced_hdr->daddr);
	}

	return XT_CONTINUE;
}

static unsigned int
ip6t_dnpt_tg(struct sk_buff *skb, const struct xt_action_param *par)
{
	const struct ip6t_npt_tginfo *npt = par->targinfo;
	struct ipv6hdr _bounced_hdr;
	struct ipv6hdr *bounced_hdr;
	struct in6_addr bounced_pfx;

	if (!ip6t_npt_map_pfx(npt, &ipv6_hdr(skb)->daddr)) {
		icmpv6_send(skb, ICMPV6_PARAMPROB, ICMPV6_HDR_FIELD,
			    offsetof(struct ipv6hdr, daddr));
		return NF_DROP;
	}

	/* rewrite src addr of bounced packet which was sent from dst range */
	bounced_hdr = icmpv6_bounced_ipv6hdr(skb, &_bounced_hdr);
	if (bounced_hdr) {
		ipv6_addr_prefix(&bounced_pfx, &bounced_hdr->saddr, npt->src_pfx_len);
		if (ipv6_addr_cmp(&bounced_pfx, &npt->src_pfx.in6) == 0)
			ip6t_npt_map_pfx(npt, &bounced_hdr->saddr);
	}

	return XT_CONTINUE;
}

static struct xt_target ip6t_npt_target_reg[] __read_mostly = {
	{
		.name		= "SNPT",
		.table		= "mangle",
		.target		= ip6t_snpt_tg,
		.targetsize	= sizeof(struct ip6t_npt_tginfo),
		.usersize	= offsetof(struct ip6t_npt_tginfo, adjustment),
		.checkentry	= ip6t_npt_checkentry,
		.family		= NFPROTO_IPV6,
		.hooks		= (1 << NF_INET_LOCAL_IN) |
				  (1 << NF_INET_POST_ROUTING),
		.me		= THIS_MODULE,
	},
	{
		.name		= "DNPT",
		.table		= "mangle",
		.target		= ip6t_dnpt_tg,
		.targetsize	= sizeof(struct ip6t_npt_tginfo),
		.usersize	= offsetof(struct ip6t_npt_tginfo, adjustment),
		.checkentry	= ip6t_npt_checkentry,
		.family		= NFPROTO_IPV6,
		.hooks		= (1 << NF_INET_PRE_ROUTING) |
				  (1 << NF_INET_LOCAL_OUT),
		.me		= THIS_MODULE,
	},
};

static int __init ip6t_npt_init(void)
{
	return xt_register_targets(ip6t_npt_target_reg,
				   ARRAY_SIZE(ip6t_npt_target_reg));
}

static void __exit ip6t_npt_exit(void)
{
	xt_unregister_targets(ip6t_npt_target_reg,
			      ARRAY_SIZE(ip6t_npt_target_reg));
}

module_init(ip6t_npt_init);
module_exit(ip6t_npt_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("IPv6-to-IPv6 Network Prefix Translation (RFC 6296)");
MODULE_AUTHOR("Patrick McHardy <kaber@trash.net>");
MODULE_ALIAS("ip6t_SNPT");
MODULE_ALIAS("ip6t_DNPT");
```

内核里面是做成驱动了，具体的算法实际也就是`RFC`里面说的了，和之前说的一致。

# 自己实现

先不说如何嵌入到项目已有的`NAT`实现，先实现自己的算法再说。直接`GO`写个转换`demo`吧。

```go
package main
// main.go
import (
	"encoding/binary"
	"fmt"
	"net"
	"flag"
	"os"
)

// 一补加法
func onesComplementAdd(a, b uint32) uint16 {
	sum := a + b
	for (sum >> 16) > 0 {
		sum = (sum & 0xFFFF) + (sum >> 16)
	}
	return uint16(sum)
}

// 一补减法
func onesComplementSubtract(a, b uint16) uint16 {
	sum := uint32(a) + uint32(^b)
	if sum > 0xFFFF {
		sum = (sum & 0xFFFF) + 1
	}
	return uint16(sum)
}

// 计算IPv6地址的16-bit一补和
func checksum16(addr net.IP) uint16 {
	var sum uint32
	for i := 0; i < 16; i += 2 {
		sum += uint32(binary.BigEndian.Uint16(addr[i : i+2]))
	}
	for sum>>16 != 0 {
		sum = (sum & 0xFFFF) + (sum >> 16)
	}
	return uint16(^sum)
}

// 前缀掩码生成器
func makeMask(pfxLen int) [16]byte {
	var mask [16]byte
	for i := 0; i < pfxLen/8; i++ {
		mask[i] = 0xFF
	}
	if pfxLen%8 != 0 {
		mask[pfxLen/8] = ^byte((1 << (8 - pfxLen%8)) - 1)
	}
	return mask
}

// 模拟 NPTv6 地址转换（带 prefix 替换 + 校验和调整）
func mapPrefix(src, dst net.IP, pfxLen int, addr net.IP, adj uint16) (net.IP, bool) {
	if len(addr) != 16 || len(src) != 16 || len(dst) != 16 {
		return nil, false
	}

	checksumIndex := 0
	mask := makeMask(pfxLen)
	newAddr := make([]byte, 16)
	for i := 0; i < 16; i++ {
		newAddr[i] = (addr[i] &^ mask[i]) | (dst[i] & mask[i])
	}

	if pfxLen < 49 {
		checksumIndex = 3
	} else {
		for i := 4; i < 8; i++ {
			oldSum := binary.BigEndian.Uint16(addr[i*2 : i*2+2])
			if oldSum != 0xFFFF {
				checksumIndex = 4
				break
			}
		}
	}
	
	// 校验和字段位于地址的哪个位置（以 2 字节为单位）
	oldSum := binary.BigEndian.Uint16(addr[checksumIndex*2 : checksumIndex*2+2])
	newSum := onesComplementAdd(uint32(oldSum), uint32(adj))

	// 特殊值 CSUM_MANGLED_0 的处理（Linux中是0xFFFF）
	if newSum == 0xFFFF {
		newSum = 0
	}
	binary.BigEndian.PutUint16(newAddr[checksumIndex*2:], newSum)

	return newAddr, true
}

func main() {

	// 命令行参数
	srcStr := flag.String("src", "", "源前缀 (IPv6)")
	dstStr := flag.String("dst", "", "目标前缀 (IPv6)")
	addrStr := flag.String("addr", "", "待转换地址 (IPv6)")
	pfxLen := flag.Int("plen", 64, "前缀长度")
	flag.Parse()

	// 检查输入
	if *srcStr == "" || *dstStr == "" || *addrStr == "" {
		fmt.Println("用法: go run main.go -src <SRC_PFX> -dst <DST_PFX> -addr <ADDR> [-plen <PREFIX_LEN>]")
		os.Exit(1)
	}

	srcPfx := net.ParseIP(*srcStr).To16()
	dstPfx := net.ParseIP(*dstStr).To16()
	addr := net.ParseIP(*addrStr).To16()

	srcSum := checksum16(srcPfx)
	dstSum := checksum16(dstPfx)
	adj := onesComplementSubtract(dstSum, srcSum)
	fmt.Printf("调整值: 0x%04X\n", adj)

	newAddr, ok := mapPrefix(srcPfx, dstPfx, *pfxLen, addr, adj)
	if ok {
		fmt.Printf("原地址: %s\n", addr)
		fmt.Printf("转换后: %s\n", net.IP(newAddr))
	} else {
		fmt.Println("转换失败")
	}
}

```

运行结果：

```bash
go run .\main.go src FD01:0203:0405:: -dst 2001:0DB8:0001:: -addr FD01:0203:0405:0001::1234  -plen 48
调整值: 0xD54F
原地址: fd01:203:405:1::1234
转换后: 2001:db8:1:d550::1234
```

# 总结

上面只是说了`NPTv6`的算法实现，如何嵌入到已有带代码中又是另外一件事情了。参照内核上的，其实也是在`Netfilter`上做一层钩子的处理就好了吧。我的实现就更简单了，直接把`nptv6`当成特殊的`nat66`处理了，但是这样实际应该和`RFC`上定义的**无状态转换**相违背了，但是从业务角度上来说这样应该也没有问题吧。
