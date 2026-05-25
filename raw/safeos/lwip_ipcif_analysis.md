# IPCIF (VNET_OVER_IPC) 分析 — T-092

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: VNET_OVER_IPC_SUPPORT、VM 通信、seL4 IPC 共享内存

---

## 1. 概述

IPCIF (IPC Network Interface) 允许虚拟机 (VM) 通过 seL4 IPC 与 NSv 通信，实现虚拟网络功能。

**主要特性**：
- 基于 seL4 IPC 进行控制平面通信
- 使用共享内存 (CMA) 进行高速数据传输
- 支持 VM → NSv 的数据包接收

---

## 2. 核心数据结构

### 2.1 IPCIF 配置

```c
// ipc-if.c
netif_custom_t ipcif_cfg = {
    .num         = 0,
    .hwaddr      = { 0x70, 0xb3, 0xd5, 0x20, 0x03, 0x01 },
    .has_dhcp    = false,
    .has_auto_ip = false,
    .has_IPv6    = false,
    .hostname    = "s32g",
    .name        = { 'i', '1' }  // "i1"
};
```

### 2.2 IPCIF 私有数据

```c
struct ipc_if_priv {
    struct netif *netif;      // lwIP netif
    int ch;                    // Channel ID
    // ...
};
```

### 2.3 DMA 缓冲区

```c
struct ipcf_dma_buf {
    paddr_t pa;              // 物理地址
    size_t size;              // 大小
    vaddr_t va;              // 虚拟地址
};
```

### 2.4 工作队列

```c
#define VNET_PROCESS_RX    (1000)
#define VNET_IPCF_RESTART  (1001)

struct ipc_vnet_work {
    int cmd;                      // 命令类型
    struct ipcf_dma_buf dma_buf;  // DMA 缓冲区
};
```

---

## 3. 内存布局

### 3.1 CMA 区域

```c
#define CMA_SIZE        0x400000ul  // 4MB

static struct cma       cma = { 0 };         // CMA 区域
static struct ds_ring   *ds_ring = 0;        // DS Ring

static dspace_t         ds = 0;              // 主 dspace
static dspace_t         tx_ds = 0;           // TX dspace
static dspace_t         rx_ds = 0;           // RX dspace
```

### 3.2 共享内存布局

```
CMA (4MB)
├─ TX Buffer (2MB)
└─ RX Buffer (2MB)
```

---

## 4. 核心流程

### 4.1 ipcif_setup — 初始化

```c
int ipcif_setup(void)
{
    // 1. 分配 CMA 区域
    cma = cma_alloc(CMA_SIZE);

    // 2. 创建 ds_ring
    ds_ring = ds_ring_create(&cma, ...);

    // 3. 创建 IPCF dspace
    tx_ds = sys_dspace_create(...);
    rx_ds = sys_dspace_create(...);

    // 4. 映射共享内存
    map_ipcf_driver_shm();

    // 5. 创建事件循环线程
    sys_thread_create(ipcif_evt_loop, ...);

    // 6. 创建工作线程
    sys_thread_create(work_thread, ...);

    return 0;
}
```

### 4.2 ipcif_evt_loop — 事件循环

**文件**: `ipc-if.c:774-830`

```c
static void *ipcif_evt_loop(void *arg)
{
    sel4_cptr ipcif_ep = (sel4_cptr)arg;

    while (1) {
        // 等待 IPCF 通知
        info = seL4_Recv(ipcif_ep, &badge);
        label = sel4_msg_info_get_label(info);

        switch (label) {
            case IPCF_NSV_NOTIFY_RX:
                // 从 message registers 获取 DMA buffer 信息
                pa = sel4_get_mr(mr++);
                size = sel4_get_mr(mr++);

                // 创建工作项
                work = malloc(sizeof(struct ipc_vnet_work));
                work->dma_buf.pa = pa;
                work->dma_buf.va = rx_shm_base + pa;
                work->cmd = VNET_PROCESS_RX;

                // 放入工作队列
                msg_box_post(&work_mbox, work);
                break;

            case SYS_DEV_DRV_RESTART:
                // 处理驱动重启
                vnet_mmap_state = WAIT_STATE;
                work->cmd = VNET_IPCF_RESTART;
                msg_box_post(&work_mbox, work);
                break;
        }
    }
}
```

### 4.3 work_thread — 工作线程

**文件**: `ipc-if.c:832-864`

```c
static void *work_thread(void *arg)
{
    while (1) {
        // 从工作队列获取任务
        work = msg_box_fetch(&work_mbox);

        switch (work->cmd) {
            case VNET_IPCF_RESTART:
                // 重新映射驱动共享内存
                map_ipcf_driver_shm();
                vnet_mmap_state = READY_STATE;
                break;

            case VNET_PROCESS_RX:
                // 处理接收数据
                ipc_if_rx_proc(NULL, SOA_FRAMEWORK_DATA_CHANNEL_ID,
                             (void *)dma_buf->va, dma_buf->size);
                break;
        }
        free(work);
    }
}
```

### 4.4 ipc_if_rx_proc — RX 处理

**文件**: `ipc-if.c:378-448`

```c
static int ipc_if_rx_proc(void *arg, int ch, void *buff, size_t size)
{
    // 1. 分配 ds_ring 内存
    spin_lock(&mem_lock);
    ipc_buf = ds_ring_mem_alloc(ds_ring, size);
    spin_unlock(&mem_lock);

    // 2. 从共享内存复制数据
    ipcf_sram_byte_copy(ipc_buf, buff, size);

    // 3. 分配自定义 pbuf
    ipc_pbuf = LWIP_MEMPOOL_ALLOC(IPC_RX_POOL);
    p = pbuf_alloced_custom(PBUF_RAW, size, PBUF_REF,
                           &ipc_pbuf->pc, ipc_buf, size);

    // 4. 调用 netif->input 传递给 lwIP
    err = netif->input(p, netif);

    // 5. 释放共享内存 buffer
    sys_file_ioctl(ipcf_ep, 0, IPCF_IOCTL_SHM_RELEASE_BUF, ...);

    return 0;
}
```

---

## 5. 完整数据流

### 5.1 VM → NSv (接收)

```
VM                              NSv                          lwIP
 │                               │                            │
 │  DMA buffer 准备好            │                            │
 │──────────────────────────────►│                            │
 │  seL4 IPC (IPCF_NSV_NOTIFY_RX)│                            │
 │  + pa, size in mr             │                            │
 │                               │                            │
 │                               ▼                            │
 │                         ipcif_evt_loop()                  │
 │                               │                            │
 │                               ▼                            │
 │                         ipc_if_rx_proc()                   │
 │                               │                            │
 │                               ├─► ds_ring_mem_alloc()      │
 │                               ├─► ipcf_sram_byte_copy()    │
 │                               ├─► pbuf_alloced_custom()    │
 │                               │                            │
 │                               ▼                            │
 │                         netif->input(p, netif)            │
 │                               │                            │
 │                               ▼                            │
 │                         ipc_if.input = ethernet_input      │
 │                               │                            │
 │                               ▼                            │
 │                         ethernet_input()                   │
 │                               │                            │
 │                               ├─► VLAN 处理                │
 │                               ├─► LWFW ingress_filter     │
 │                               │                            │
 │                               ▼                            │
 │                         UDP/TCP/input()                   │
 │                               │                            │
 ▼                               ▼                            ▼
```

### 5.2 内存复制路径

```
RX Path:
  DMA Buffer (物理地址)
      │
      │ ipcf_sram_byte_copy()
      ▼
  ds_ring (CMA 区域)
      │
      │ pbuf_alloced_custom()
      ▼
  pbuf (PBUF_REF)
      │
      │ netif->input()
      ▼
  lwIP 协议栈
```

---

## 6. 与 VIRT_BRG 对比

| 特性 | IPCIF (VNET_OVER_IPC) | VIRT_BRG (VIRT_BRG_SUPPORT) |
|------|------------------------|------------------------------|
| **用途** | VM → NSv 通信 | VM 间桥接 + NSv 通信 |
| **通信方式** | seL4 IPC + 共享内存 | seL4 IPC + 共享内存 |
| **数据方向** | 单向 (VM → NSv) | 双向 (VM ↔ VM, VM ↔ NSv) |
| **协议栈** | 作为 netif 集成 | 作为 bridge port |
| **典型场景** | 虚拟网络接口 | 虚拟机间网络桥接 |

---

## 7. netif 配置

### 7.1 ipc_if_init

```c
err_t ipc_if_init(struct netif *netif)
{
    netif->name[0] = 'i';
    netif->name[1] = '1';

    netif->output = etharp_output;
    netif->output_ip6 = ethip6_output;
    netif->linkoutput = ipc_if_low_level_transmit;

    netif->mtu = MTU_SIZE;  // 1500
    netif->hwaddr_len = NETIF_MAX_HWADDR_LEN;
    netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP |
                    NETIF_FLAG_IGMP;
    return ERR_OK;
}
```

---

## 8. 安全考虑

### 8.1 状态检查

```c
sync_mutex_lock(&vnet_mmap_state_lock);
if (vnet_mmap_state != READY_STATE) {
    sync_mutex_unlock(&vnet_mmap_state_lock);
    goto vnet_mmap_state_err;
}
ipcf_sram_byte_copy(ipc_buf, buff, size);
sync_mutex_unlock(&vnet_mmap_state_lock);
```

### 8.2 内存分配保护

```c
// 使用自旋锁保护 ds_ring 分配
spin_lock(&mem_lock);
ipc_buf = ds_ring_mem_alloc(ds_ring, size);
spin_unlock(&mem_lock);
```

---

## 9. 总结

### 9.1 关键设计

1. **双线程模型**：事件循环 + 工作线程分离
2. **共享内存**：使用 CMA 实现高速数据传输
3. **自定义 pbuf**：避免数据拷贝，直接使用共享内存
4. **工作队列**：异步处理，提高响应性

### 9.2 数据流

```
VM DMA → seL4 IPC 通知 → ipcif_evt_loop → work_thread
    → ipc_if_rx_proc → ds_ring → pbuf → netif->input
    → ethernet_input → lwIP 协议栈
```

### 9.3 与 Linux 对比

| 特性 | SafeOS IPCIF | Linux virtio-net |
|------|--------------|------------------|
| **控制平面** | seL4 IPC | virtio-MMIO |
| **数据平面** | CMA 共享内存 | VRING |
| **驱动模型** | IPCF driver | virtio_pci driver |
| **集成方式** | netif | net_device |
