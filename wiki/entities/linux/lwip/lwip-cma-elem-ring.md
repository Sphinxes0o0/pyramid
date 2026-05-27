---
type: entity
tags: [safeos, lwip, cma, elem-ring, dma, lock-free, shared-memory, sel4]
created: 2026-05-27
sources: [safeos-lwip-analysis-summary]
---

# CMA Buffer 与 elem_ring

## 定义

CMA (Contiguous Memory Area) 是 NSv 与 NIC Driver 之间共享的 96MB 内存区域。elem_ring 是基于 CMA 的无锁单生产者/单消费者环形缓冲区，用于传递 DMA buffer 指针。

## CMA 架构

```
CMA Region (96MB)
├── empty_rx_buf_ring    (RX 空缓冲，NSv→NIC)
├── used_rx_buf_ring     (RX 已收包，NIC→NSv)
├── pending_tx_buf_ring  (TX 待发，NSv→NIC)
├── used_tx_buf_ring     (TX 已完成，NIC→NSv)
├── DMA Buffers (pbuf)
└── Reserved Memory
```

### 核心操作

| 操作 | 函数 | 说明 |
|------|------|------|
| 分配 DMA buffer | `alloc_dma_buf()` → `dma_pbuf_alloc()` | slab cache 分配 |
| VA→PA 转换 | `cma_va_to_pa()` | 虚拟地址转物理地址 |
| PA→VA 转换 | `cma_pa_to_va()` | 物理地址转虚拟地址 |

## elem_ring 无锁队列

### 核心操作

```c
// elem_ring_put (生产者)
dmb(ish);                          // 内存屏障：确保数据写入在索引更新之前
ring->elems[ring->put_idx] = e;  // 写入数据
ring->put_idx = (ring->put_idx + 1) % ring->n;

// elem_ring_get (消费者)
data = ring->elems[ring->get_idx]; // 读取数据
dmb(ish);                          // 内存屏障
ring->get_idx = (ring->get_idx + 1) % ring->n;
```

### 关键保证

1. **数据-索引一致性**: `dmb(ish)` 确保先写数据，后更新索引
2. **单生产者/单消费者**: 无锁设计
3. **零拷贝**: 传递的是 buffer 指针 (PA/VA)，不是数据本身

### 状态检查

```c
// 满: (put_idx + 1) % n == get_idx
// 空: get_idx == put_idx
```

## RX/TX 数据流

```
RX: empty_rx_buf_ring (提供空 buffer)
        ↓ NIC DMA 写入
    used_rx_buf_ring (通知 NSv)
        ↓ elem_ring_get
    cma_pa_to_va → rx_callback → ethernet_input

TX: ethernet_output → ethif_link_output
        ↓ elem_ring_put
    pending_tx_buf_ring (提供给 NIC)
        ↓ sel4_signal
    NIC DMA 读取 → 发送
        ↓ elem_ring_put
    used_tx_buf_ring
        ↓ free_dma_buf
    pbuf_free
```

## 性能特征

| 操作 | 延迟 |
|------|------|
| CMA 分配 (`kr_malloc`) | ~50-100ns |
| CMA 释放 (`kr_free`) | ~20-50ns |
| VA↔PA 转换 | ~5-10ns |
| elem_ring put/get | ~10-20ns |

## 相关概念

- [[entities/linux/safeos/safeos-nsv]] — NSv 网络服务器
- [[entities/linux/lwip/lwip-sel4-performance-boundary]] — seL4 + lwIP 性能边界
- [[lwip-index]] — lwIP 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引

## 来源详情

- [[sources/safeos-lwip-analysis-summary]] — lwIP 深度分析文档汇总
