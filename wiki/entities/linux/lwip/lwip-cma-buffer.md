---
type: entity
tags: [linux, lwip, network, dma, memory, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP CMA Buffer — Contiguous Memory Area

## 定义

CMA (Contiguous Memory Area) 是 SafeOS NSv 中用于 NIC DMA 的物理连续内存区域，桥接 NSv 虚拟地址和 NIC 驱动物理地址访问，实现零拷贝数据传输。

## 核心数据结构

```c
struct cma {
    vaddr_t va;      // 虚拟地址起始
    paddr_t pa;      // 物理地址起始
    size_t size;     // 区域大小
    kr_malloc_t mem; // 内核内存分配器
};
```

## 初始化 (init_ds_ring)

```
1. sys_mem_map(getpid(), &cma.pa, &cma.va, CMA_SIZE, PAGE_DMA)
   → 从 CMA 分配连续物理内存

2. sys_dspace_create(cma.size, attr, &cma.va, &ds)
   → 创建 DSpace 共享内存抽象

3. sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr)
   → 授予 NIC Driver 访问权限

4. sys_ds_ring_share(nic_ep, ds)
   → 共享 DS Ring 给 NIC 驱动
```

## CMA 内存布局

```
CMA Region (CMA_SIZE bytes)
┌────────────────────────────────────────┬─────────────────┐
│         DS Ring / DMA Buffers         │  Reserved       │
│         (用于 RX/TX pbuf 分配)        │  Memory         │
│                                        │  (NIC init)     │
│  cma.va ─────────────────────────►    │                 │
└────────────────────────────────────────┴─────────────────┘
         ↑                                      ↑
         └──────────────┬───────────────────────┘
                        │
                   cma.pa (物理地址)
```

## VA/PA 地址转换

| 函数 | 功能 |
|------|------|
| `cma_va_to_pa()` | 虚拟地址 → 物理地址 (NIC DMA 用) |
| `cma_pa_to_va()` | 物理地址 → 虚拟地址 (NSv 访问用) |

## RX/TX Ring 与 CMA 的关系

| Ring | 方向 | 内容 | 生产者 | 消费者 |
|------|------|------|--------|--------|
| `empty_rx_buf_ring` | NIC → NSv | 可用 DMA buffer PA | NSv (预填充) | NIC (写入) |
| `used_rx_buf_ring` | NIC → NSv | 已接收数据 DMA buffer PA | NIC | NSv |
| `pending_tx_buf_ring` | NSv → NIC | 待发送数据 DMA buffer PA | NSv | NIC |
| `used_tx_buf_ring` | NIC → NSv | 已发送完成 DMA buffer PA | NIC | NSv |

## 数据流

```
RX:
empty_rx_buf_ring (提供空 buffer)
        │
        ▼ NIC DMA 写入
used_rx_buf_ring (通知 NSv)
        │
        ▼ elem_ring_get
nic_rx_thread: cma_pa_to_va → rx_callback
        │
        ▼
ethernet_input(p, &vnet_if)

TX:
ethernet_output → ethif_link_output
        │
        ▼ elem_ring_put
pending_tx_buf_ring (提供给 NIC)
        │
        ▼ sel4_signal
NIC: DMA 读取 buffer → 发送
```

## 与 elem_ring 的关系

CMA 缓冲区的物理地址通过 `elem_ring` 传递给 NIC：
- `elem_ring_put(pending_tx_buf_ring, elem{e.pa})` — 写入 buffer PA
- `elem_ring_get(used_rx_buf_ring)` — 读取 buffer PA
- `cma_pa_to_va(&cma, e.pa)` — PA → VA

## 相关概念

- [[entities/linux/lwip/lwip-elem-ring]] — 无锁环形缓冲区
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 机制
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 结构

## 来源详情

- [[sources/safeos-lwip-extensions]]
