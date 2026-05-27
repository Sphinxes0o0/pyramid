# elem_ring 无锁队列分析 — T-002

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: 无锁单生产者/单消费者环形缓冲区实现、内存屏障、边界条件

---

## 1. 概述

### 1.1 什么是 elem_ring

`elem_ring` 是一个**无锁单生产者/单消费者环形缓冲区**，用于 NSv 和 NIC Driver 之间传递 DMA buffer 指针。

### 1.2 设计约束

```
┌─────────────────────────────────────────────────────────────────────┐
│                          elem_ring 约束                             │
├─────────────────────────────────────────────────────────────────────┤
│  1. 单生产者 + 单消费者: 不需要锁                                    │
│  2. 原子性: 生产者和消费者在不同上下文 (线程/进程)                    │
│  3. 零拷贝: 传递的是 union elem (物理地址/虚拟地址)，不是数据本身    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 数据结构

### 2.1 union elem

**文件**: `os-framework/libs/os_libs/libcore/include/core/elem_ring.h:9`

```c
union elem {
    void        *ptr;    // 通用指针
    paddr_t     pa;      // 物理地址 (用于 DMA)
    vaddr_t     va;     // 虚拟地址
    size_t      size;   // 大小
};
```

**使用方式**: RX/TX path 使用 `pa` 字段传递物理地址。

### 2.2 struct elem_ring

**文件**: `elem_ring.h:22`

```c
struct elem_ring {
    uint32_t    n;           // 槽数量
    uint32_t    n_avail;     // 可用数量 (已弃用)
    volatile uint32_t get_idx; // 消费者索引
    volatile uint32_t put_idx; // 生产者索引
    uint32_t    user;         // 用户数据
    union elem  elems[0];    // 槽数组 (柔性数组)
};
```

### 2.3 内存布局

```
struct elem_ring (在 CMA 共享内存中)
┌─────────────────────────────────────────────────────┐
│  n          = 4096                                 │
│  n_avail    = (已弃用)                             │
│  get_idx    = 0x0001 (消费者下一个要读的槽)         │
│  put_idx    = 0x0003 (生产者下一个要写的槽)         │
│  user       = 0                                    │
├─────────────────────────────────────────────────────┤
│  elems[0]  = union elem { .pa = 0x12340000 }     │ ← 已被消费
│  elems[1]  = union elem { .pa = 0x12341000 }     │ ← 已被消费
│  elems[2]  = union elem { .pa = 0x12342000 }     │ ← 当前可读 ← get_idx
│  elems[3]  = union elem { .pa = 0x12343000 }     │ ← 已写入 ← put_idx
│  ...                                                │
│  elems[4095] = ...                                │
└─────────────────────────────────────────────────────┘
```

---

## 3. 核心操作

### 3.1 elem_ring_put (生产者写入)

**文件**: `elem_ring.h:135`

```c
static inline int elem_ring_put(struct elem_ring *er, union elem e)
{
    uint32_t next = (er->put_idx + 1) % er->n;  // 下一个位置

    // 检查是否满 (满时 next == get_idx)
    if (next == er->get_idx) return -ENOSPC;

    // Step 1: 写入数据到槽
    er->elems[er->put_idx] = e;

    // Step 2: 内存屏障 (确保数据写入完成)
    dmb(ish);

    // Step 3: 更新 put_idx (消费者现在可以看到新数据)
    er->put_idx = next;

    return 0;
}
```

**关键点**:
1. **先写数据，后写索引**
2. `dmb(ish)` 确保数据写入在索引更新之前完成

### 3.2 elem_ring_get (消费者读取)

**文件**: `elem_ring.h:110`

```c
static inline union elem elem_ring_get(struct elem_ring *er)
{
    union elem e = { 0 };

    // 检查是否空 (空时 get_idx == put_idx)
    if (er->get_idx == er->put_idx) return e;

    // Step 1: 读取数据 (从 get_idx 位置)
    e = er->elems[er->get_idx];

    // Step 2: 内存屏障 (确保数据读取在索引更新之前)
    dmb(ish);

    // Step 3: 更新 get_idx (生产者现在可以重用该槽)
    er->get_idx = (er->get_idx + 1) % er->n;

    return e;
}
```

### 3.3 状态检查

**文件**: `elem_ring.h:82-108`

```c
// 检查是否满
static inline int elm_ring_is_full(struct elem_ring *er)
{
    uint32_t get_idx = er->get_idx;
    uint32_t put_idx = er->put_idx;
    return ((put_idx + 1) % er->n == get_idx) ? 1 : 0;
}

// 检查是否空
static inline int elm_ring_is_empty(struct elem_ring *er)
{
    return (er->put_idx == er->get_idx) ? 1 : 0;
}

// 获取可用元素数量
static inline int elm_ring_avail_size(struct elem_ring *er)
{
    if (er->put_idx >= er->get_idx) {
        return (er->put_idx - er->get_idx);      // 未环绕
    } else {
        return (er->put_idx - er->get_idx + er->n); // 环绕
    }
}
```

---

## 4. 内存屏障机制

### 4.1 ARM 内存屏障

**文件**: `elem_ring.h:6-7`

```c
#define dsb(opt) do { asm volatile("dsb " # opt ::: "memory"); } while (0)
#define dmb(opt) do { asm volatile("dmb " # opt ::: "memory"); } while (0)
```

| 指令 | 全称 | 作用 |
|------|------|------|
| `dmb ish` | Data Memory Barrier, Inner Shareable | 确保屏障前的内存访问在屏障后的内存访问之前完成 |
| `dsb ish` | Data Synchronization Barrier | 确保所有内存访问在同步点之前完成 |

### 4.2 为什么需要内存屏障

**问题场景**:

```
生产者 (CPU 0)                    消费者 (CPU 1)
───────────                        ───────────
1. er->elems[put] = data    ← 可能被重排到
2. er->put_idx = new_idx         │
                                  │ 消费者看到 put_idx 更新
                                  │ 但 elems[put] 还是旧数据！
```

**解决方案**: `dmb ish` 确保数据写入在索引更新之前被消费者看到

### 4.3 环形索引的内存序

```
生产者视角:
┌──────────────────────────────────────────────────────────────┐
│  写入 elems[put_idx]  →  dmb(ish)  →  put_idx++          │
│         ↑                      ↑                ↑          │
│    数据可见              屏障完成         索引更新         │
└──────────────────────────────────────────────────────────────┘

消费者视角:
┌──────────────────────────────────────────────────────────────┐
│  读取 get_idx  →  读取 elems[get_idx]  →  dmb(ish)  →  get_idx++  │
│      ↑                    ↑                   ↑                ↑
│   索引旧值              数据读取            屏障完成         索引更新
└──────────────────────────────────────────────────────────────┘
```

---

## 5. 边界条件处理

### 5.1 满的情况

```
当 (put_idx + 1) % n == get_idx 时，队列满

处理:
- elem_ring_put 返回 -ENOSPC
- 调用者需要处理错误 (丢弃/等待/重试)
```

### 5.2 空的情况

```
当 get_idx == put_idx 时，队列空

处理:
- elem_ring_get 返回 {0}
- 调用者应该检查返回值
```

### 5.3 环绕 (Wrap-around)

```
索引使用模运算: (idx + 1) % n

示例: n = 8
put_idx: 7 → 0 → 1 → 2 → ...
         └────┘└────┘└────┘
         环绕      环绕      环绕
```

### 5.4 潜在问题: 假共享

**问题**:
```
struct elem_ring {
    uint32_t    n;
    uint32_t    n_avail;
    volatile uint32_t get_idx;  ← 被消费者更新
    volatile uint32_t put_idx;  ← 被生产者更新
    uint32_t    user;
    union elem  elems[0];       ← 被生产者写入，被消费者读取
};
```

`get_idx` 和 `put_idx` 在**同一 cache line**，当一个核心更新时，会 invalidate 另一个核心的 cache line。

**影响**:
- 消费者更新 `get_idx` → 生产者的 `put_idx` 读取被invalidate
- 生产者需要重新读取 `put_idx`

**优化建议** (代码注释中提到):
> have read and write in different cacheline will help in multiple core situation

---

## 6. 在 NSv 中的使用

### 6.1 四个 Ring 的用途

| Ring | 方向 | 生产者 | 消费者 | 传递内容 |
|------|------|--------|--------|----------|
| `empty_rx_buf_ring` | NIC ← NSv | NSv (预填充) | NIC | 可用 RX buffer PA |
| `used_rx_buf_ring` | NIC → NSv | NIC | NSv | 已接收数据的 buffer PA |
| `pending_tx_buf_ring` | NIC ← NSv | NSv | NIC | 待发送数据的 buffer PA |
| `used_tx_buf_ring` | NIC → NSv | NIC | NSv | 已发送完成的 buffer PA |

### 6.2 RX path 示例

**main.c:4986-4990** (`nic_rx_thread`):

```c
while (1) {
    union elem e = elem_ring_get(used_rx_buf_ring);  // 消费者
    if (e.pa) {
        LOCK_TCPIP_CORE();
        rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
        UNLOCK_TCPIP_CORE();
    } else {
        NET_PERF_STATS_INC(nic_rx_dones);
        break;
    }
}
```

### 6.3 TX path 示例

**main.c:3874-3891** (`ethif_link_output`):

```c
// 放入 pending_tx_buf_ring (生产者)
ret = elem_ring_put(pending_tx_buf_ring, e);  // e.pa = buffer 物理地址

// 通知 NIC
if (was_empty || is_full || ...) {
    sel4_signal(nic_tx_ntfn);  // 异步通知，不阻塞
}
```

---

## 7. 性能特征

### 7.1 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|------------|------|
| `elem_ring_put` | O(1) | 固定次数操作 |
| `elem_ring_get` | O(1) | 固定次数操作 |
| `elm_ring_avail_size` | O(1) | 固定次数操作 |

### 7.2 空间复杂度

```
空间 = sizeof(struct elem_ring) + n * sizeof(union elem)
     = 16 bytes + n * 8 bytes
     = 16 + 8n bytes

示例: n = 4096
空间 = 16 + 4096 * 8 = 32,784 bytes ≈ 32 KB
```

### 7.3 吞吐瓶颈

```
瓶颈分析:
- 每个 packet 需要一次 elem_ring_put (TX) + 一次 elem_ring_get (RX)
- elem_ring 操作本身 ~10-20ns
- 主要瓶颈在 NIC DMA 速度和 seL4 IPC 延迟
```

---

## 8. 与其他模块的关系

### 8.1 上游调用者

| 模块 | 函数 | 调用目的 |
|------|------|----------|
| **refill_dma_buf_if_needed** | `elem_ring_put` | 预填充 RX buffers |
| **ethif_link_output** | `elem_ring_put` | 提供 TX buffer 给 NIC |
| **nic_rx_thread** | `elem_ring_get` | 获取 RX 完成的数据 |
| **free_complete_tx_packet_pbuf** | `elem_ring_get` | 获取 TX 完成的 buffer |

### 8.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **CMA** | `cma_pa_to_va` | PA→VA 转换 |
| **seL4 IPC** | `sel4_signal` | 通知 NIC |
| **NIC Driver** | DMA | 通过 PA 访问 buffer |

---

## 9. 总结

### 9.1 核心设计

```
无锁 + 单生产者/单消费者 + 内存屏障
```

### 9.2 关键保证

1. **数据-索引一致性**: `dmb(ish)` 确保先写数据，后更新索引
2. **无锁设计**: 利用单生产者/单消费者约束，避免锁开销
3. **零拷贝**: 传递的是 buffer 指针，不是数据本身

### 9.3 限制

1. **不支持多消费者**: 多个消费者会导致 data race
2. **不支持多生产者**: 多个生产者会导致 data race
3. **假共享**: `get_idx` 和 `put_idx` 在同一 cache line

### 9.4 性能

- **单操作延迟**: ~10-20ns
- **吞吐限制**: 取决于 seL4 IPC 和 NIC DMA 速度
- **最优场景**: 高吞吐、连续数据流
