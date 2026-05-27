---
type: entity
tags: [linux, lwip, network, memory, pbuf]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP pbuf Structure

## 定义

`struct pbuf` 是 lwIP 的**数据包缓冲区结构**，负责存储网络数据包内容，支持链式结构 (pbuf chain)，引用计数管理 (refcount)。

## pbuf 结构

```c
struct pbuf {
    struct pbuf *next;      // 单链表 next 指针
    void *payload;          // 指向数据起始位置
    u16_t tot_len;          // 此 pbuf + 后续所有 pbuf 的总长度
    u16_t len;              // 此 pbuf 的数据长度
    u8_t type_internal;     // pbuf 类型 + 分配来源
    u8_t priority;           // 缓冲区优先级 (对应 802.1Q PCP)
    u8_t flags;             // 各种标志
    LWIP_PBUF_REF_T ref;    // 引用计数
    u8_t if_idx;            // 输入 netif 的索引
#ifdef NIO_LWIP_LWCT
    u64_t _lwct;            // 连接追踪状态 (SafeOS)
#endif
};
```

## pbuf 类型

| 类型 | 分配位置 | 数据位置 | 用途 |
|------|---------|---------|------|
| **PBUF_RAM** | heap (malloc) | heap | TX (应用发送) |
| **PBUF_POOL** | pool (快速) | pool | RX (NIC 接收) |
| **PBUF_ROM** | pool | ROM/flash | 常量数据 |
| **PBUF_REF** | pool | 外部引用 | 零拷贝引用 |

## 链表不变量

```
p->tot_len == p->len + (p->next ? p->next->tot_len : 0)
```

## refcount 机制

```
ref == 0: pbuf 已被释放，可回收
ref > 0:  pbuf 被引用，不能释放
```

- `pbuf_ref()`: ref++，增加引用
- `pbuf_free()`: ref--，ref==0 时真正释放

## pbuf 层级 (Layer)

```
┌─────────────────────────────────────────────────────────────────────┐
│  PBUF_RAW: 无 header                                              │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_LINK: + Ethernet Header                                     │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_IP: + IP Header                                              │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_TRANSPORT: + TCP/UDP Header                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## SafeOS 特供字段

- **if_idx**: 记录输入 netif 索引，UDP socket bind 检查中使用
- **_lwct**: 连接追踪状态扩展 (LWFW)
- **priority**: VLAN PCP 传递

## 相关概念
- [[entities/linux/safeos/safeos-lwip-lwfw-plan]]
- [[entities/linux/lwfw/lwfw-core-filtering]]

- [[entities/linux/lwip/lwip-malloc]] — heap 和 pool 分配机制
- [[entities/linux/lwip/lwip-ethernet-input]] — pbuf 在 RX 路径中的使用
- [[entities/linux/lwip/lwip-ethernet-output]] — pbuf 在 TX 路径中的使用
- [[entities/linux/lwip/lwip-ip4-input]] — pbuf 中的 if_idx 字段使用
