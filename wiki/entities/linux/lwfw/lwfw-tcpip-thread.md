---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW and tcpip_thread

## 定义

LWFW 过滤器通过 hook 点集成到 lwIP 协议栈，其执行上下文取决于 RX (Ingress) 或 TX (Egress) 路径。SafeOS 使用 `LWIP_TCPIP_CORE_LOCKING` 模式进行锁保护。

## RX: Ingress Filter 执行上下文

```
NIC DMA 接收
    │
    ▼
nic_rx_thread (seL4_Recv)
    │
    ▼
LOCK_TCPIP_CORE()                // 获取全局锁
    │
    ▼
rx_callback(p)
    │
    ├─► ethernet_input(p, netif)
    │     │
    │     └─► ip4_input(p, netif)
    │           │
    │           └─► lwfw_p->ops->ingress_filter(p, inp)
    │
    ▼
UNLOCK_TCPIP_CORE()              // 释放全局锁
```

### 执行上下文总结

| 特性 | RX Ingress Filter |
|------|-------------------|
| **执行线程** | `nic_rx_thread` |
| **持有锁** | `LOCK_TCPIP_CORE()` |
| **上下文类型** | 中断上下文 (NIC 中断触发) |
| **SafeOS 特有** | 是，显式锁保护 |

## TX: Egress Filter 执行上下文

```
Application Thread
    │
    ▼
tcp_output() / udp_output()
    │
    ▼
ip4_output(p, src, dest, ttl, tos, proto)
    │
    ▼
lwfw_p->ops->egress_filter(p, netif)  // 直接调用，无锁
    │
    ▼
netif->output(netif, p, dest)
```

### 执行上下文总结

| 特性 | TX Egress Filter |
|------|------------------|
| **执行线程** | 应用线程 (调用 socket API 的线程) |
| **持有锁** | 无 (直接调用) |
| **上下文类型** | 进程上下文 |

## 锁机制

```c
#if LWIP_TCPIP_CORE_LOCKING
#define LOCK_TCPIP_CORE()     sys_mutex_lock(&lock_tcpip_core)
#define UNLOCK_TCPIP_CORE()   sys_mutex_unlock(&lock_tcpip_core)
#else
#define LOCK_TCPIP_CORE()
#define UNLOCK_TCPIP_CORE()
#endif
```

## LWCT 交互顺序

```
RX:
  nic_rx_thread
    │
    ├─► LWCT (lwct_main_hook, DIR_RX)  // 连接状态更新
    │
    └─► LWFW (ingress_filter)          // 包过滤决策

TX:
  应用线程
    │
    ├─► LWCT (lwct_main_hook, DIR_TX)  // 连接状态更新
    │
    └─► LWFW (egress_filter)           // 包过滤决策
```

## 与 Linux 对比

| 特性 | SafeOS LWFW | Linux Netfilter |
|------|-------------|-----------------|
| hook 位置 | ip4_input / ip4_output | PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING |
| 执行上下文 | 单一线程 | 多个 hook 点 (不同上下文) |
| 锁机制 | 全局 mutex | RCU/per-CPU |
| 连接追踪 | LWCT (独立模块) | conntrack (内核集成) |

## 相关概念

- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread 详解
- [[entities/linux/lwfw/lwfw-hook-injection]] — Hook 注入点
- [[entities/linux/lwfw/lwfw-lwct-interaction]] — LWCT 交互机制
- [[entities/linux/lwfw/lwfw-core-filtering]] — 过滤引擎架构
