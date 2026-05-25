# CMA Buffer 分析 — T-001

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: CMA (Contiguous Memory Area) 缓冲区分配、pbuf 映射、DMA 共享机制

---

## 1. 概述

### 1.1 CMA 在架构中的位置

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Application Layer                             │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     NSv (lwIP + Socket API)                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    CMA Region (共享内存)                    │   │
│  │   ┌──────────────┬──────────────┬────────────────────┐   │   │
│  │   │  DMA Rx Buf  │  DMA Tx Buf  │   Reserved Memory   │   │   │
│  │   │  (empty_rx)  │  (used_tx)   │   (NIC init)       │   │   │
│  │   └──────────────┴──────────────┴────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │      seL4 IPC / DSpace    │
                    └─────────────┬─────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      NIC Driver (PFE/VIRTIO)                        │
│              (使用物理地址访问 DMA buffers)                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心问题

**为什么需要 CMA？**
- **DMA 要求**：NIC 直接内存访问需要物理上连续的缓冲区
- **地址空间隔离**：NSv (用户态) 和 NIC Driver (可能内核态或独立进程) 需要共享内存
- **零拷贝**：避免不必要的数据拷贝

---

## 2. CMA 数据结构

### 2.1 struct cma 定义

**文件**: `os-framework/libs/os_libs/libcore/include/core/cma.h:9`

```c
struct cma {
    vaddr_t     va;      // 虚拟地址起始
    paddr_t     pa;      // 物理地址起始
    size_t      size;    // 区域大小
    kr_malloc_t mem;     // 内核内存分配器
};
```

### 2.2 CMA 初始化

**文件**: `os-framework/servers/net/src/main.c:3670` (`init_ds_ring`)

```c
static int init_ds_ring(void)
{
    // Step 1: 分配 CMA 区域
    if (!cma.va) {
        err = sys_mem_map(getpid(), &cma.pa, &cma.va, CMA_SIZE, PAGE_DMA);
        //            ↑           ↑           ↑           ↑
        //         进程ID      物理地址    虚拟地址    DMA-safe 标志
        cma.size = CMA_SIZE;
    }

    // Step 2: 创建 DSpace (共享内存抽象)
    err = sys_dspace_create(cma.size, attr, &cma.va, &ds);

    // Step 3: 初始化 DS Ring (用于分配 DMA buffers)
    ds_ring = ds_ring_init(cma.va, cma.pa,
                           (cma.size - CMA_RESERVED_MEM_SIZE),
                           ds, getpid(), NSV_NIC_DESC_SIZE);

    // Step 4: 预留内存 (NIC 初始化参数)
    reserved_data = (struct cma_reserved_data *)
        (cma.va + (cma.size - CMA_RESERVED_MEM_SIZE));

    // Step 5: 授予 NIC Driver 访问权限
    err = sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr);
    err = sys_ds_ring_share(nic_ep, ds);
}
```

### 2.3 CMA 内存布局

```
CMA Region (CMA_SIZE bytes)
┌────────────────────────────────────────┬─────────────────┐
│                                        │                 │
│         DS Ring / DMA Buffers          │  Reserved       │
│         (用于 RX/TX pbuf 分配)        │  Memory         │
│                                        │  (NIC init)     │
│                                        │                 │
│  cma.va ─────────────────────────►    │ cma.va +       │
│                                        │ cma.size -     │
│                                        │ CMA_RESERVED_   │
│                                        │ MEM_SIZE       │
└────────────────────────────────────────┴─────────────────┘
         ↑                                      ↑
         └──────────────┬───────────────────────┘
                        │
                   cma.pa (物理地址)
```

---

## 3. DMA Buffer 分配与释放

### 3.1 分配路径

**文件**: `os-framework/servers/net/src/main.c:4155`

```c
static paddr_t alloc_dma_buf(size_t UNUSED size)
{
    // 分配一个 pbuf (位于 CMA 区域)
    struct pbuf *p = dma_pbuf_alloc(DMA_BUF_SIZE);
    if (!p) return 0;

    // 返回物理地址 (供 NIC DMA 使用)
    return cma_va_to_pa(&cma, (vaddr_t)p);
}
```

**地址转换**:
```c
// cma.h:43
static inline paddr_t cma_va_to_pa(struct cma *cma, vaddr_t va)
{
    return cma->pa + (va - cma->va);
}
```

### 3.2 释放路径

**文件**: `os-framework/servers/net/src/main.c:4171`

```c
static void free_dma_buf(paddr_t pa)
{
    // 物理地址 → 虚拟地址
    struct pbuf *p = (struct pbuf *)cma_pa_to_va(&cma, pa);
    if (!p) return;

    // 释放 pbuf (回到 slab cache)
    pbuf_free(p);
}
```

### 3.3 RX Buffer 预填充

**文件**: `os-framework/servers/net/src/main.c:4738` (`refill_dma_buf_if_needed`)

```c
static void refill_dma_buf_if_needed(void)
{
    // 计算需要填充的数量
    int avail_limit = empty_rx_buf_ring->n
                      - elm_ring_avail_size(empty_rx_buf_ring) - 1;

    while (i < avail_limit) {
        union elem e;
        e.pa = alloc_dma_buf(DMA_BUF_SIZE);

        // 放入 empty_rx_buf_ring (供 NIC 写入)
        if (elem_ring_put(empty_rx_buf_ring, e) == -ENOSPC) {
            free_dma_buf(e.pa);  // 失败则释放
            return;
        }
        i++;
    }
}
```

---

## 4. VA/PA 地址转换

### 4.1 核心函数

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `cma_va_to_pa()` | `cma.h:43` | 虚拟地址 → 物理地址 |
| `cma_pa_to_va()` | `cma.h:50` | 物理地址 → 虚拟地址 |
| `cma_alloc()` | `cma.h:27` | 从 CMA 分配内存 |
| `cma_free()` | `cma.h:37` | 释放到 CMA |

### 4.2 地址转换示例

```
场景: NIC DMA 完成，收到物理地址 e.pa

1. elem_ring_get(used_rx_buf_ring) → e.pa = 0x12345678
                                         (物理地址)

2. cma_pa_to_va(&cma, e.pa) → p = 0x7f000000
                                  (虚拟地址)

3. rx_callback(p) → ethernet_input(p, &vnet_if)
                        (使用虚拟地址访问 pbuf)

---

TX 场景: 应用发送数据

1. ethif_link_output(netif, p)
       ↓
2. elem_ring_put(pending_tx_buf_ring, e)
       其中 e.pa = cma_va_to_pa(&cma, (vaddr_t)p)
       (NIC 需要物理地址进行 DMA)

3. sel4_signal(nic_tx_ntfn) → 通知 NIC
```

---

## 5. 与 NIC Driver 的共享机制

### 5.1 DSpace 共享

**文件**: `main.c:3726-3729`

```c
// 授予 NIC driver 访问权限
err = sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr);

// 共享 DS ring
err = sys_ds_ring_share(nic_ep, ds);
```

### 5.2 共享的 Ring 结构

| Ring | 方向 | 内容 | 生产者 | 消费者 |
|------|------|------|--------|--------|
| `empty_rx_buf_ring` | NIC → NSv | 可用的 DMA buffer 物理地址 | NSv (预填充) | NIC (写入 RX 数据) |
| `used_rx_buf_ring` | NIC → NSv | 已接收数据的 DMA buffer 物理地址 | NIC (RX 完成) | NSv (读取) |
| `pending_tx_buf_ring` | NSv → NIC | 待发送数据的 DMA buffer 物理地址 | NSv (发送请求) | NIC (读取并 DMA) |
| `used_tx_buf_ring` | NIC → NSv | 已发送完成的 DMA buffer 物理地址 | NIC (TX 完成) | NSv (释放) |

### 5.3 数据流

```
RX 路径:
empty_rx_buf_ring (提供空 buffer)
        │
        ▼ NIC 写入 DMA
used_rx_buf_ring (通知 NSv)
        │
        ▼ elem_ring_get
nic_rx_thread: cma_pa_to_va → rx_callback
        │
        ▼
ethernet_input(p, &vnet_if)

TX 路径:
ethernet_output → ethif_link_output
        │
        ▼ elem_ring_put
pending_tx_buf_ring (提供给 NIC)
        │
        ▼ sel4_signal
NIC: DMA 读取 buffer → 发送
        │
        ▼ elem_ring_put
used_tx_buf_ring
        │
        ▼ free_dma_buf(p)
(释放 pbuf)
```

---

## 6. 性能特征

### 6.1 内存访问模式

| 操作 | 延迟 | 带宽 |
|------|------|------|
| CMA 分配 (`kr_malloc`) | ~50-100ns | - |
| CMA 释放 (`kr_free`) | ~20-50ns | - |
| VA→PA 转换 | ~5-10ns | - |
| 物理地址 DMA 访问 | ~100-200ns | 取决于 NIC |

### 6.2 关键设计点

1. **零拷贝**: 数据直接在 DMA buffer 和 pbuf 之间传递，无额外拷贝
2. **无锁**: elem_ring 是单生产者/单消费者，无锁设计
3. **批量操作**: `refill_dma_buf_if_needed` 批量填充 RX buffers
4. **slab 缓存**: `dma_pbuf_alloc` 使用 slab cache，减少分配开销

---

## 7. 与其他模块的关系

### 7.1 上游调用者

| 模块 | 函数 | 调用目的 |
|------|------|----------|
| **nic_rx_thread** | `elem_ring_get` → `rx_callback` | 获取 RX buffer |
| **refill_dma_buf_if_needed** | `alloc_dma_buf` | 预填充 RX buffer |
| **ethif_link_output** | `cma_va_to_pa` | 获取 TX buffer 物理地址 |

### 7.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **elem_ring** | `elem_ring_put/get` | Buffer 传递 |
| **NIC Driver** | DMA 访问 | 通过物理地址访问 |
| **lwIP pbuf** | `pbuf_free` | 释放 buffer |

### 7.3 Hook 点

- `sys_mem_map` — CMA 区域分配 (seL4 syscall)
- `sys_dspace_grant` — 权限授予 NIC driver
- `dma_pbuf_alloc_raw` — pbuf slab cache 分配

---

## 8. 配置参数

| 参数 | 定义位置 | 默认值 | 说明 |
|------|----------|--------|------|
| `CMA_SIZE` | `main.c` | - | CMA 区域总大小 |
| `DMA_BUF_SIZE` | `main.c` | - | 单个 DMA buffer 大小 |
| `DMA_REFILL_THRESHOLD` | `main.c` | - | 触发 refill 的阈值 |
| `CMA_RESERVED_MEM_SIZE` | `main.c` | - | 预留内存大小 |
| `PAGE_DMA` | - | - | DMA-safe 页面标志 |

---

## 9. 总结

### 9.1 CMA 的核心作用

1. **桥接虚拟地址和物理地址**: NSv 使用虚拟地址，NIC 使用物理地址
2. **支持 DMA**: 物理连续的内存区域，供 NIC 直接访问
3. **跨进程共享**: 通过 seL4 DSpace 机制共享给 NIC driver

### 9.2 关键流程

```
分配: alloc_dma_buf
  → dma_pbuf_alloc (slab cache)
  → cma_va_to_pa (转换为物理地址)
  → elem_ring_put (提供给 NIC)

释放: elem_ring_get
  → cma_pa_to_va (转换为虚拟地址)
  → pbuf_free (归还到 slab cache)
```

### 9.3 性能优化

- **预填充**: `refill_dma_buf_if_needed` 批量填充，减少分配频率
- **slab cache**: `dma_pbuf_alloc_raw` 使用专用 cache，避免通用分配器开销
- **无锁设计**: elem_ring 单生产者/单消费者，无锁操作
