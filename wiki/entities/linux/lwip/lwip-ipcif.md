---
type: entity
tags: [linux, lwip, network, vm, ipc, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# IPCIF — VNET_OVER_IPC Network Interface

## 定义

IPCIF (IPC Network Interface) 允许虚拟机 (VM) 通过 seL4 IPC 与 NSv 通信，实现虚拟网络功能。使用共享内存 (CMA) 进行高速数据传输，seL4 IPC 进行控制平面通信。

## 双线程模型

```
ipcif_evt_loop()  ←── seL4 IPC (IPCF_NSV_NOTIFY_RX)
    │  (等待 VM 通知)
    ▼
work_thread()
    │  (处理工作)
    ▼
ipc_if_rx_proc() → netif->input() → ethernet_input()
```

## 核心流程

### ipcif_evt_loop — 事件循环
```c
while (1) {
    info = seL4_Recv(ipcif_ep, &badge);
    switch (label) {
        case IPCF_NSV_NOTIFY_RX:
            pa = sel4_get_mr(mr++);   // DMA buffer 物理地址
            size = sel4_get_mr(mr++);  // 大小
            work->cmd = VNET_PROCESS_RX;
            msg_box_post(&work_mbox, work);
            break;
    }
}
```

### ipc_if_rx_proc — RX 处理
```c
static int ipc_if_rx_proc(void *arg, int ch, void *buff, size_t size)
{
    // 1. 分配 ds_ring 内存
    ipc_buf = ds_ring_mem_alloc(ds_ring, size);

    // 2. 从共享内存复制数据
    ipcf_sram_byte_copy(ipc_buf, buff, size);

    // 3. 分配自定义 pbuf
    ipc_pbuf = LWIP_MEMPOOL_ALLOC(IPC_RX_POOL);
    p = pbuf_alloced_custom(PBUF_RAW, size, PBUF_REF,
                           &ipc_pbuf->pc, ipc_buf, size);

    // 4. 调用 netif->input 传递给 lwIP
    err = netif->input(p, netif);
}
```

## VM → NSv 数据流

```
VM                              NSv                          lwIP
 │                               │                            │
 │  DMA buffer 准备好            │                            │
 │──────────────────────────────►│                            │
 │  seL4 IPC (IPCF_NSV_NOTIFY_RX)│                            │
 │  + pa, size in mr             │                            │
 │                               ▼                            │
 │                         ipcif_evt_loop()                  │
 │                               │                            │
 │                               ▼                            │
 │                         work_thread()                     │
 │                               │                            │
 │                               ├─► ds_ring_mem_alloc()    │
 │                               ├─► ipcf_sram_byte_copy()  │
 │                               ├─► pbuf_alloced_custom()  │
 │                               │                            │
 │                               ▼                            │
 │                         netif->input(p, netif)            │
 │                               │                            │
 │                               ▼                            │
 │                         ethernet_input()                   │
```

## 与 VIRT_BRG 对比

| 特性 | IPCIF (VNET_OVER_IPC) | VIRT_BRG (VIRT_BRG_SUPPORT) |
|------|------------------------|------------------------------|
| 用途 | VM → NSv 通信 | VM 间桥接 + NSv 通信 |
| 通信方式 | seL4 IPC + 共享内存 | seL4 IPC + 共享内存 |
| 数据方向 | 单向 (VM → NSv) | 双向 (VM ↔ VM, VM ↔ NSv) |
| 协议栈集成 | 作为 netif 集成 | 作为 bridge port |
| 典型场景 | 虚拟网络接口 | 虚拟机间网络桥接 |

## 相关概念

- [[entities/linux/lwip/lwip-virt-brg]] — VIRT_BRG 支持
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 机制
- [[entities/linux/lwip/lwip-cma-buffer]] — CMA 共享内存

## 来源详情

- [[sources/safeos-lwip-extensions]]
