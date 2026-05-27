---
type: index
tags: [linux, lwip, source, navigation]
created: 2026-05-25
---

# lwIP Source — Source Index

> lwIP 核心源文件的源码阅读层。提供函数索引、数据结构摘要、调用链。
>
> 对应源码: `external/lwip_ds_mcu/src/core/`

## Source Entity Pages

> 每个页面包含：函数索引表、关键数据结构、调用链、交叉引用。

| Source File | Entity Page | Lines | 功能 |
|-------------|-------------|-------|------|
| [[entities/linux/lwip/source/ip4.c]] | ip4.c.md | 1307 | IPv4 输入/输出、路由查找、转发 |
| [[entities/linux/lwip/source/tcp.c]] | tcp.c.md | 2768 | TCP 连接建立/关闭、定时器、PCB 管理 |
| [[entities/linux/lwip/source/udp.c]] | udp.c.md | 1385 | UDP 分发/发送、PCB 管理 |
| [[entities/linux/lwip/source/pbuf.c]] | pbuf.c.md | 1570 | pbuf 分配/释放、chain 操作、header 调整 |
| [[entities/linux/lwip/source/netif.c]] | netif.c.md | 1913 | netif 管理、地址配置、loopback |

## Architecture Map

```
ethernet_input / ip_input
        ↓
    netif_input (netif.c)
        ↓
    ┌─────────────────────────┐
    │  IPv4  ip4_input      │
    │  ┌──────────────────┐  │
    │  │ tcp_input (tcp_in.c)│  │
    │  │ udp_input (udp.c) │  │
    │  │ icmp_input        │  │
    │  │ igmp_input        │  │
    │  │ raw_input         │  │
    │  └──────────────────┘  │
    └─────────────────────────┘
        ↓
    ┌─────────────────────────┐
    │  TCP  tcp_output        │
    │  UDP  udp_send          │
    │  IP   ip4_output        │
    └─────────────────────────┘
        ↓
    netif->output
        ↓
    ethernet_output / loopback
```

## Source Layer vs Analysis Layer

| 维度 | Source Layer | Analysis Layer |
|------|-------------|---------------|
| 粒度 | 函数级 (行号) | 模块/概念级 |
| 用途 | 快速查阅代码位置 | 理解设计意图 |
| 内容 | 函数索引、数据结构 | 流程分析、调用关系、协议细节 |
| 示例 | "ip4_route L166" | "路由查找通过 NETIF_FOREACH 遍历所有 netif" |

## Related Indexes

- [[lwip-index]] — lwIP 模块完整索引 (~47 entities)
- [[lwfw-index]] — LWFW 防火墙分析
- [[safeos-index]] — SafeOS NSv 架构
