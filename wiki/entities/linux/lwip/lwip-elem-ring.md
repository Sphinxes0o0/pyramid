---
type: entity
tags: [linux, lwip, network, lock-free, ring-buffer, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP Elem Ring — Lock-Free Ring Buffer

## 定义

`elem_ring` 是 **无锁单生产者/单消费者环形缓冲区**，用于 NSv 和 NIC Driver 之间传递 DMA buffer 指针（物理地址/虚拟地址），实现跨进程零拷贝数据传递。

## 核心数据结构

### union elem
```c
union elem {
    void *ptr;    // 通用指针
    paddr_t pa;   // 物理地址 (用于 DMA)
    vaddr_t va;   // 虚拟地址
    size_t size;  // 大小
};
```

### struct elem_ring
```c
struct elem_ring {
    uint32_t n;              // 槽数量
    volatile uint32_t get_idx;  // 消费者索引
    volatile uint32_t put_idx;  // 生产者索引
    union elem elems[0];       // 槽数组 (柔性数组)
};
```

## 核心操作

### elem_ring_put (生产者)
```c
static inline int elem_ring_put(struct elem_ring *er, union elem e)
{
    uint32_t next = (er->put_idx + 1) % er->n;
    if (next == er->get_idx) return -ENOSPC;  // 队列满

    er->elems[er->put_idx] = e;   // 1. 写入数据
    dmb(ish);                        // 2. 内存屏障
    er->put_idx = next;              // 3. 更新索引
    return 0;
}
```

### elem_ring_get (消费者)
```c
static inline union elem elem_ring_get(struct elem_ring *er)
{
    if (er->get_idx == er->put_idx) return e{0};  // 队列空

    e = er->elems[er->get_idx];   // 1. 读取数据
    dmb(ish);                        // 2. 内存屏障
    er->get_idx = (er->get_idx + 1) % er->n;  // 3. 更新索引
    return e;
}
```

## 内存屏障机制

| 指令 | 全称 | 作用 |
|------|------|------|
| `dmb ish` | Data Memory Barrier, Inner Shareable | 确保屏障前的内存访问在屏障后的内存访问之前完成 |

**为什么需要屏障**:
```
生产者: elems[put] = data  ← 可能被重排到
       put_idx = new_idx   ← 索引先更新!

消费者: 看到 put_idx 更新
       但 elems[put] 还是旧数据！
```

## 在 NSv 中的四个 Ring

| Ring | 方向 | 生产者 | 消费者 | 传递内容 |
|------|------|--------|--------|----------|
| `empty_rx_buf_ring` | NIC ← NSv | NSv (预填充) | NIC | 可用 RX buffer PA |
| `used_rx_buf_ring` | NIC → NSv | NIC | NSv | 已接收数据的 buffer PA |
| `pending_tx_buf_ring` | NIC ← NSv | NSv | NIC | 待发送数据的 buffer PA |
| `used_tx_buf_ring` | NIC → NSv | NIC | NSv | 已发送完成的 buffer PA |

## 性能特征

| 操作 | 时间复杂度 | 说明 |
|------|------------|------|
| `elem_ring_put` | O(1) | 固定次数操作 |
| `elem_ring_get` | O(1) | 固定次数操作 |
| 单操作延迟 | ~10-20ns | 主要瓶颈在 seL4 IPC |

## 关键设计保证

1. **数据-索引一致性**: `dmb(ish)` 确保先写数据，后更新索引
2. **无锁设计**: 单生产者/单消费者约束，避免锁开销
3. **零拷贝**: 传递的是 buffer 指针，不是数据本身

## 相关概念

- [[entities/linux/lwip/lwip-cma-buffer]] — CMA 缓冲区
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 机制
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 结构

## 来源详情

- [[sources/safeos-lwip-extensions]]
