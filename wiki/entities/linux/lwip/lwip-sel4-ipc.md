---
type: entity
tags: [linux, lwip, network, sel4, ipc, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# seL4 IPC Mechanism — lwIP on seL4

## 定义

seL4 IPC 是 SafeOS NSv 中 NSv 与 NIC 驱动、Hypervisor 之间通信的基础机制。核心包括 **Notification** (异步通知) 和 **Endpoint** (同步通信) 两种 IPC 类型。

## seL4 IPC 类型与延迟

| IPC 类型 | 用途 | 延迟 |
|----------|------|------|
| **seL4_Signal** | 单向通知 (NIC → NSv) | ~50-200ns |
| **seL4_Recv** | 阻塞接收 (NSv → NIC) | ~100-500ns |
| **seL4_Call** | 同步调用 (NSv ↔ NIC) | ~200-1000ns |
| **seL4_Send** | 同步发送 | ~100-400ns |
| **seL4_Reply** | 回复 (配合 Call) | ~50-150ns |

## Notification 机制

### TX 通知 (ethif_link_output)
```c
err_t ethif_link_output(struct netif *netif, struct pbuf *q)
{
    ret = elem_ring_put(pending_tx_buf_ring, e);
    if (was_empty || is_full || ...) {
        sel4_signal(nic_tx_ntfn);  // 异步，不阻塞
    }
    return ERR_OK;
}
```

### RX 接收 (nic_rx_thread)
```c
while (1) {
    info = seL4_Recv(nsv_nic_ep, &badge);  // 阻塞等待
    if (badge == 0) {
        // 正常的 RX 数据
        while (1) {
            elem = elem_ring_get(used_rx_buf_ring);
            if (e.pa) {
                LOCK_TCPIP_CORE();
                rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
                UNLOCK_TCPIP_CORE();
            } else break;
        }
    }
}
```

## Badge 机制

Badge 是 32-bit 值，用于识别消息发送者：
```c
info = seL4_Recv(nsv_nic_ep, &badge);

if (badge == NET_PM_NIC_RX_BADGE && label == NET_PM_SUSPEND) {
    // NIC 收到暂停命令
} else if (badge == 0) {
    // 正常的 RX 数据
}
```

## NIC RX/TX 完整流程

### RX 路径
```
NIC Driver                              NSv (lwIP)
─────────────                            ───────────
1. DMA 完成，写入 empty_rx_buf
2. elem_ring_put(used_rx_buf, e)
3. sel4_signal(nic_rx_ntfn)  ──────► seL4_Recv(nsv_nic_ep, &badge)
                                          │
                                          ▼
                                     elem_ring_get(used_rx_buf)
                                          │
                                          ▼
                                     cma_pa_to_va(e.pa)
                                          │
                                          ▼
                                     rx_callback(pbuf)
```

### TX 路径
```
NSv (lwIP)                              NIC Driver
─────────────                            ───────────
1. ethernet_output(pbuf)
2. ethif_link_output()
3. elem_ring_put(pending_tx_buf, e)
4. sel4_signal(nic_tx_ntfn)  ─────────► 收到 notification
                                              │
                                              ▼
                                         elem_ring_get(pending_tx)
                                              │
                                              ▼
                                         DMA 读取 buffer
```

## IPC 开销对网络栈的影响

- **单 packet IPC 开销**: ~150-700ns per packet (RX)
- **主要瓶颈**: 单 tcpip_thread 处理 + seL4 IPC
- **优化方向**: 批处理、异步化

## 相关概念

- [[entities/linux/lwip/lwip-sel4-function]] — lwIP on seL4 整体分析
- [[entities/linux/lwip/lwip-sel4-performance-boundary]] — 性能边界分析
- [[entities/linux/lwip/lwip-cma-buffer]] — CMA 缓冲区
- [[entities/linux/lwip/lwip-elem-ring]] — 无锁队列

## 来源详情

- [[sources/safeos-lwip-extensions]]
