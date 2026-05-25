---
type: entity
tags: [linux, lwip, network, sel4, safeos, integration]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP on seL4 — Function-Level Analysis

## 定义

SafeOS 中 lwIP 运行在 **NSv (Network Server)** 用户态进程中，通过 seL4 微内核与 NIC 驱动交互。lwIP 的 sys_arch 层针对 seL4 做了深度适配：信号量、互斥锁、mbox、时间接口均基于 seL4 IPC 实现。

## NSv 进程结构

```
┌─────────────────────────────────────────────────────────────┐
│  NSv 进程                                                   │
│  线程:                                                       │
│  ├─ event_loop()     ← 处理 App 的 BSD socket 请求         │
│  ├─ nic_rx_thread()  ← 从 NIC 驱动接收数据包               │
│  ├─ tcpip_thread()   ← lwIP 内部协议处理线程               │
│  └─ select_thread()  ← 阻塞 socket 的事件等待               │
│  关键数据结构:                                                │
│  ├─ vnet_if (struct netif)  ← lwIP 虚拟网卡               │
│  ├─ CMA (96MB)              ← 与 NIC 共享的连续内存        │
│  └─ elem_ring x4             ← 无锁环形缓冲 (TX/RX 队列)    │
└─────────────────────────────────────────────────────────────┘
```

## 初始化路径

```
main() → net_resources_init()
       → init_ds_ring()  // CMA + elem_ring
       → create_nic_thread()  // seL4 IPC endpoint 注册
       → tcpip_init(NULL, NULL)  // lwIP 初始化
       → netif_add(&vnet_if, ...)  // 添加虚拟网卡
       → event_loop()  // 进入主事件循环
```

## 收包路径 (RX)

```
NIC 驱动
  │
  │ DMA 接收数据包到物理内存 (CMA 区域)
  │ elem_ring_put(used_rx_buf_ring, e)
  │ sel4_signal(nic_rx_ntfn)
  ▼
nic_rx_thread() → elem_ring_get(used_rx_buf_ring)
    → cma_pa_to_va() → rx_callback()
      → LOCK_TCPIP_CORE()
      → vnet_if.input(p, &vnet_if) = ethernet_input()
        → raw_afpacket_input()  // AF-PACKET 捕获
        → ip4_input()
          → LWFW ingress_filter
          → tcp_input() / udp_input()
      → UNLOCK_TCPIP_CORE()
```

## 发包路径 (TX)

```
App → sys_net_sendto() → seL4 IPC
  ▼
event_loop() → sys_sendto_nb()
  → lwip_sendto() → netconn_sendto()
    → tcp_output() / udp_output() → ip4_output_if()
      → LWFW egress_filter
      → netif->output = etharp_output
        → ethernet_output()
          → netif->linkoutput = ethif_link_output()
            → elem_ring_put(pending_tx_buf_ring, e)
            → sel4_signal(nic_tx_ntfn)  // 异步通知
```

## seL4 适配层 (sys_arch_sel4.c)

| 接口 | seL4 实现 |
|------|-----------|
| 信号量 | seL4 notification (P/V 操作) |
| 互斥锁 | seL4 mtx 对象 |
| mbox | 消息队列 (tcpip 线程通信) |
| 内存分配 | ds_ring_mem_alloc (CMA 区域) |
| 时间接口 | time_get_ns() / NS_IN_MS |

## LWIP_TCPIP_CORE_LOCKING

```c
#define LOCK_TCPIP_CORE()   sys_mutex_lock(&lock_tcpip_core)
#define UNLOCK_TCPIP_CORE() sys_mutex_unlock(&lock_tcpip_core)
```

所有 RX packet 在 `nic_rx_thread` 中获取锁后处理，TX packet 在 `tcpip_thread` 中处理，形成单线程瓶颈。

## 相关概念

- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 机制详解
- [[entities/linux/lwip/lwip-sel4-performance-boundary]] — 性能边界分析
- [[entities/linux/lwip/lwip-sel4-interaction]] — 整体交互架构
- [[entities/linux/lwip/lwip-cma-buffer]] — CMA 缓冲区
- [[entities/linux/lwip/lwip-elem-ring]] — 无锁队列

## 来源详情

- [[sources/safeos-lwip-extensions]]
