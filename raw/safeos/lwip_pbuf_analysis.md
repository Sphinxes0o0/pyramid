# pbuf 管理层分析 — T-112

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: pbuf 结构、pbuf 类型、pool/heap/malloc 分配、refcount 机制

---

## 1. 概述

pbuf 是 lwIP 的**数据包缓冲区结构**，负责：
1. 存储网络数据包内容
2. 支持链式结构 (pbuf chain)
3. 引用计数管理 (refcount)
4. 多种分配策略 (pool/heap/ROM/ref)

### 1.1 pbuf 在网络栈中的位置

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Application                                 │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Socket API                                   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       lwIP Protocol Stack                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      struct pbuf                               │   │
│  │   ┌─────────────────────────────────────────────────────┐   │   │
│  │   │  pbuf chain (单链表)                                │   │   │
│  │   │  ┌───────┐   ┌───────┐   ┌───────┐               │   │   │
│  │   │  │ pbuf0 │──►│ pbuf1 │──►│ pbuf2 │──► NULL        │   │   │
│  │   │  └───────┘   └───────┘   └───────┘               │   │   │
│  │   └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DMA / NIC                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. pbuf 结构详解

**文件**: `external/lwip_ds_mcu/src/include/lwip/pbuf.h:205`

```c
struct pbuf {
    // ============================================
    // 链表结构
    // ============================================
    struct pbuf *next;  // 单链表 next 指针

    // ============================================
    // 数据指针
    // ============================================
    void *payload;  // 指向数据起始位置

    // ============================================
    // 长度字段
    // ============================================
    u16_t tot_len;  // 此 pbuf + 后续所有 pbuf 的总长度
    u16_t len;        // 此 pbuf 的数据长度

    // ============================================
    // 类型和来源
    // ============================================
    u8_t type_internal;  // pbuf 类型 + 分配来源

    // ============================================
    // 优先级和标志
    // ============================================
    u8_t priority;  // 缓冲区优先级 (用于 QoS)
    u8_t flags;     // 各种标志

    // ============================================
    // 引用计数
    // ============================================
    LWIP_PBUF_REF_T ref;  // 引用计数

    // ============================================
    // 网络接口索引
    // ============================================
    u8_t if_idx;  // 输入 netif 的索引

    // ============================================
    // 连接追踪扩展 (SafeOS 特供)
    // ============================================
    #ifdef NIO_LWIP_LWCT
    u64_t _lwct;  // 连接追踪状态
    #endif
};
```

---

## 3. pbuf 链表结构

### 3.1 链表不变量

```
p->tot_len == p->len + (p->next ? p->next->tot_len : 0)
```

### 3.2 示例

```
pbuf chain (tot_len = 1500, len = 1500)
    │
    ├── pbuf[0]: len=1500, tot_len=1500, payload→ Ethernet Header
    │       next
    │       ▼
    ├── pbuf[1]: len=1460, tot_len=1460, payload→ IP Header + TCP
    │       next
    │       ▼
    └── NULL
```

---

## 4. pbuf 类型

### 4.1 类型定义

**文件**: `pbuf.h:155-170`

```c
// PBUF_RAM: 从 heap 分配，用于 TX
// 特点: 数据和结构都在 heap，malloc 分配
#define PBUF_RAM (PBUF_ALLOC_FLAG_DATA_CONTIGUOUS | \
                   PBUF_TYPE_FLAG_STRUCT_DATA_CONTIGUOUS | \
                   PBUF_TYPE_ALLOC_SRC_MASK_STD_HEAP)

// PBUF_ROM: 从 pool 分配，只读数据
// 特点: 数据在 ROM/flash，结构在 pool
#define PBUF_ROM PBUF_TYPE_ALLOC_SRC_MASK_STD_MEMP_PBUF

// PBUF_REF: 从 pool 分配，引用外部数据
// 特点: 数据由外部提供，只引用
#define PBUF_REF (PBUF_TYPE_FLAG_DATA_VOLATILE | \
                  PBUF_TYPE_ALLOC_SRC_MASK_STD_MEMP_PBUF)

// PBUF_POOL: 从 pool 分配，用于 RX
// 特点: 结构和数据都在 pool，快速分配
#define PBUF_POOL (PBUF_ALLOC_FLAG_RX | \
                    PBUF_TYPE_FLAG_STRUCT_DATA_CONTIGUOUS | \
                    PBUF_TYPE_ALLOC_SRC_MASK_STD_MEMP_PBUF_POOL)
```

### 4.2 类型对比

| 类型 | 分配位置 | 数据位置 | 用途 |
|------|---------|---------|------|
| **PBUF_RAM** | heap (malloc) | heap | TX (应用发送) |
| **PBUF_POOL** | pool (快速) | pool | RX (NIC 接收) |
| **PBUF_ROM** | pool | ROM/flash | 常量数据 |
| **PBUF_REF** | pool | 外部引用 | 零拷贝引用 |

---

## 5. pbuf_alloc — 分配函数

**文件**: `external/lwip_ds_mcu/src/core/pbuf.c`

```c
struct pbuf *
pbuf_alloc(pbuf_layer layer, u16_t length, pbuf_type type)
{
    struct pbuf *p;
    u16_t offset;
    int err;

    // ============================================
    // Step 1: 计算 header 偏移
    // ============================================
    offset = 0;
    switch (layer) {
        case PBUF_LINK:      // + link layer header
            offset = SIZEOF_ETH_HDR;
            break;
        case PBUF_IP:        // + IP header
            offset = SIZEOF_ETH_HDR + IP_HLEN;
            break;
        case PBUF_TRANSPORT: // + transport header
            offset = SIZEOF_ETH_HDR + IP_HLEN + TCP_HLEN;
            break;
        case PBUF_RAW:      // 无 header
        default:
            break;
    }

    // ============================================
    // Step 2: 根据类型分配
    // ============================================
    switch (type) {
        case PBUF_POOL:
            p = pbuf_alloced_alloc(pool, offset + length);
            if (p == NULL) {
                return NULL;
            }
            p->payload = (u8_t *)p + SIZEOF_STRUCT_PBUF + offset;
            break;

        case PBUF_RAM:
            p = mem_malloc(length + SIZEOF_STRUCT_PBUF + offset);
            if (p == NULL) {
                return NULL;
            }
            p->payload = (u8_t *)p + SIZEOF_STRUCT_PBUF + offset;
            break;

        case PBUF_ROM:
        case PBUF_REF:
            p = memp_malloc(MEMP_PBUF);
            if (p == NULL) {
                return NULL;
            }
            // payload 由调用者设置
            break;
    }

    // ============================================
    // Step 3: 初始化字段
    // ============================================
    p->next = NULL;
    p->len = p->tot_len = length;
    p->type_internal = type;
    p->ref = 1;  // 初始 ref = 1
    p->if_idx = NETIF_NO_INDEX;

    return p;
}
```

---

## 6. refcount 机制

### 6.1 引用计数规则

```
ref == 0: pbuf 已被释放，可回收
ref > 0:  pbuf 被引用，不能释放
```

### 6.2 pbuf_ref — 增加引用

```c
void pbuf_ref(struct pbuf *p)
{
    SYS_ARCH_SET(p->ref, p->ref + 1);
}
```

### 6.3 pbuf_free — 减少引用

```c
u8_t pbuf_free(struct pbuf *p)
{
    u8_t claimed = 0;

    while (p != NULL) {
        struct pbuf *next = p->next;

        if (p->ref > 1) {
            p->ref--;  // 减少引用
        } else {
            // ref == 1 或 0，真正释放
            pbuf_free_impl(p);
            claimed++;
        }

        p = next;
    }

    return claimed;
}
```

---

## 7. pbuf 层级 (Layer)

**文件**: `pbuf.h:75-115`

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Application Data                              │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_TRANSPORT: + TCP/UDP Header                                  │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_IP: + IP Header                                              │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_LINK: + Ethernet Header                                     │
├─────────────────────────────────────────────────────────────────────┤
│  PBUF_RAW: 无 header                                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. SafeOS 中的 pbuf 扩展

### 8.1 if_idx 字段

```c
u8_t if_idx;  // 输入 netif 的索引
```

用于记录 packet 来自哪个 netif，在 UDP socket bind 检查中使用。

### 8.2 _lwct 字段

```c
#ifdef NIO_LWIP_LWCT
u64_t _lwct;  // 连接追踪状态
#endif
```

用于 LWFW 连接追踪，存储 connection state。

---

## 9. pbuf 与 DMA 的关系

### 9.1 RX 路径 (NIC → pbuf)

```
NIC DMA
    │
    ▼
CMA buffer (物理地址)
    │
    ▼
pbuf_alloc(PBUF_POOL)  ← 从 pool 分配
    │
    ▼
elem_ring_put(used_rx_buf_ring)  ← 传递 pbuf
    │
    ▼
nic_rx_thread:
    cma_pa_to_va()  ← 转换为虚拟地址
    │
    ▼
rx_callback(p)  ← pbuf 传递给协议栈
```

### 9.2 TX 路径 (pbuf → NIC)

```
tcp_output(p)
    │
    ▼
pbuf_alloc(PBUF_RAM)  ← 从 heap 分配
    │
    ▼
ethernet_output()
    │
    ▼
ethif_link_output()
    │
    ▼
elem_ring_put(pending_tx_buf_ring)
    │
    ▼
sel4_signal(nic_tx_ntfn)
    │
    ▼
NIC DMA
```

---

## 10. 性能特征

### 10.1 分配速度

| 类型 | 分配方式 | 速度 |
|------|---------|------|
| **PBUF_POOL** | pool (固定大小) | 最快 (~10 cycles) |
| **PBUF_RAM** | heap (malloc) | 较慢 (~100+ cycles) |
| **PBUF_REF** | pool | 快 |

### 10.2 内存占用

```
PBUF_POOL: 每个 pbuf 大小固定 (e.g., 128 bytes)
           总数 = PBUF_POOL_SIZE

PBUF_RAM: 按需分配，大小 = header + data
```

---

## 11. 总结

### 11.1 pbuf 的核心作用

```
存储网络数据
    │
    ├─► 单链表结构，支持大数据分段
    │
    ├─► 多种分配策略 (pool/heap/ROM/ref)
    │     - PBUF_POOL: RX，快速
    │     - PBUF_RAM:  TX，heap
    │
    ├─► 引用计数管理
    │     - ref > 0: 不能释放
    │     - ref == 0: 可回收
    │
    └─► 层级支持
          - PBUF_RAW → PBUF_LINK → PBUF_IP → PBUF_TRANSPORT
```

### 11.2 关键设计

1. **单链表**: 简单的 pbuf chain，支持大数据
2. **引用计数**: 避免重复释放，支持共享
3. **多种类型**: pool 用于 RX 高性能，heap 用于 TX 灵活性
4. **header 预留**: pbuf_alloc 支持预留各层 header 空间

### 11.3 SafeOS 特供

1. **if_idx**: 记录输入 netif 索引
2. **_lwct**: 连接追踪状态扩展
3. **priority**: VLAN PCP 传递
