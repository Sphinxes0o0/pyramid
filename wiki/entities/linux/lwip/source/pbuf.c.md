---
type: entity
tags: [lwip, pbuf, source, memory]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# pbuf.c — Packet Buffer Management

> pbuf：内存分配/释放、header 操作、链管理、复制

## 文件概览

| 属性 | 值 |
|------|-----|
| 路径 | `src/core/pbuf.c` |
| 行数 | 1570 |
| 功能 | pbuf 分配、释放、chain 操作、header 调整、reference 管理 |
| 依赖 | mem, memp, netif |

## 函数索引

### 分配
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_alloc` | 237 | 分配 pbuf (PBUF_POOL/ROM/REF/RAM) |
| `pbuf_alloc_reference` | 357 | 分配 PBUF_REF/ROM (不分配数据内存) |
| `pbuf_alloced_custom` | 393 | 分配自定义 pbuf (用户管理 payload) |
| `pbuf_clone` | 1362 | 克隆整个 pbuf chain |

### 释放
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_free` | 758 | 释放 pbuf chain (refcount--, ref=0 时释放) |
| `pbuf_free_ooseq` | 129 | 释放 TCP out-of-sequence pbuf |
| `pbuf_free_header` | 704 | 跳过前 N 字节并释放中间 pbuf |

### Header 操作
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_add_header` | 584 | 向前移动 payload 指针 (添加 header) |
| `pbuf_add_header_force` | 594 | 强制 add_header (允许 PBUF_REF) |
| `pbuf_remove_header` | 615 | 向后移动 payload 指针 (去掉 header) |
| `pbuf_header` | 679 | 通用 header 调整 (正=add, 负=remove) |
| `pbuf_header_force` | 689 | 强制 header 调整 |

### Chain 操作
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_cat` | 904 | 连接两个 pbuf chain (不增加 ref) |
| `pbuf_chain` | 946 | 连接两个 pbuf chain (h 引用 t) |
| `pbuf_dechain` | 968 | 分离第一个 pbuf 与后续 chain |
| `pbuf_coalesce` | 1334 | 合并 pbuf chain 为单个 pbuf |

### 复制
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_copy` | 1017 | 完整复制 pbuf chain (pbuf → pbuf) |
| `pbuf_copy_partial` | 1083 | 部分复制 (offset + len) |
| `pbuf_get_contiguous` | 1130 | 获取连续内存 (零拷贝或复制) |

### 写入
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_take` | 1252 | 复制数据到 pbuf |
| `pbuf_take_at` | 1296 | pbuf_take 带 offset |

### 其他
| 函数 | 行号 | 功能 |
|------|------|------|
| `pbuf_realloc` | 432 | 缩减 pbuf 到指定长度 |
| `pbuf_ref` | 880 | 增加 reference count |
| `pbuf_clen` | 860 | 计数 pbuf chain 长度 |
| `pbuf_skip` | 1234 | 跳过前 N 字节 |
| `pbuf_get_at` | 1427 | 读取指定偏移字节 |
| `pbuf_put_at` | 1467 | 写入指定偏移字节 |
| `pbuf_memcmp` | 1490 | 比较 pbuf 与内存 |
| `pbuf_memfind` | 1532 | 在 pbuf 中查找内存模式 |
| `pbuf_strstr` | 1559 | 在 pbuf 中查找字符串 |
| `pbuf_split_64k` | 1169 | 分割 pbuf chain 到 64K 以下 |

## 关键数据结构

### pbuf_type (enum)
```c
PBUF_POOL  // 从 memp pool 分配，适合 RX (高速)
PBUF_RAM   // 从 heap 分配，payload 连续
PBUF_ROM   // 引用只读内存，不分配 buffer
PBUF_REF   // 引用可变内存，不分配 buffer
```

### pbuf_layer (enum)
```
PBUF_TRANSPORT  // 预留 transport header 空间
PBUF_IP         // 预留 IP header 空间
PBUF_LINK       // 预留 link layer header 空间
PBUF_RAW        // 无 header 空间
```

### struct pbuf (核心字段)
```c
struct pbuf {
  struct pbuf *next;    // 下一个 pbuf (chain)
  void *payload;        // 数据指针
  u16_t len;            // 本 pbuf 长度
  u16_t tot_len;        // 本 + 后续所有 pbuf 总长
  u16_t ref;            // reference count
  u8_t type_internal;   // pbuf type
  // ...
};
```

## 调用链

### pbuf_alloc 路径
```
tcp_input / udp_input / raw_input
  → pbuf_alloc(PBUF_POOL)
    → memp_malloc(MEMP_PBUF_POOL)
    → pbuf_init_alloced_pbuf()
```

### pbuf_free 释放路径
```
tcp_input / udp_input / ...
  → pbuf_free()
    → ref-- (SYS_ARCH_PROTECT 保护)
    → [ref == 0] → memp_free / mem_free
```

### TCP OOSEQ 释放
```
pbuf_pool_is_empty()
  → tcpip_try_callback(pbuf_free_ooseq_callback)
    → pbuf_free_ooseq()
      → 遍历 tcp_active_pcbs
      → tcp_free_ooseq(pcb)  // 释放 ooseq 队列
```

## 交叉引用

### Analysis 层
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 结构详解
- [[entities/linux/lwip/lwip-malloc]] — 内存管理

### 依赖
- [[entities/linux/lwip/source/tcp.c]] — TCP OOSEQ
- [[entities/linux/lwip/source/udp.c]] — UDP 输入

### 消费者
- [[entities/linux/lwip/lwip-ethernet-input]] — L2 → L3 入口
- [[entities/linux/lwip/lwip-tcp-recv-queue]] — TCP 接收队列
