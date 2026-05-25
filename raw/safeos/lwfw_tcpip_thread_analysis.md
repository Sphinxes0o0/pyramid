# LWFW 与 tcpip_thread — T-082

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: Filter 执行上下文、锁机制、LWFW 与协议栈线程交互

---

## 1. 概述

LWFW (Lightweight Firewall) 过滤器通过 hook 点集成到 lwIP 协议栈，其执行上下文取决于：
1. **RX (Ingress)**: 包从 NIC 接收后的处理路径
2. **TX (Egress)**: 应用发送包的处理路径

---

## 2. 锁机制

### 2.1 LOCK_TCPIP_CORE 定义

**文件**: `include/lwip/tcpip.h:52-64`

```c
#if LWIP_TCPIP_CORE_LOCKING
extern sys_mutex_t lock_tcpip_core;
#define LOCK_TCPIP_CORE()     sys_mutex_lock(&lock_tcpip_core)
#define UNLOCK_TCPIP_CORE()   sys_mutex_unlock(&lock_tcpip_core)
#else
#define LOCK_TCPIP_CORE()
#define UNLOCK_TCPIP_CORE()
#endif
```

### 2.2 SafeOS 中的锁使用

在 SafeOS 中，`nic_rx_thread` 显式使用锁来保护 RX 处理：

**文件**: `os-framework/servers/net/src/main.c:4984-4995`

```c
if (badge == 0) {
    while (1) {
        union elem e = elem_ring_get(used_rx_buf_ring);
        if (e.pa) {
            LOCK_TCPIP_CORE();
            rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
            UNLOCK_TCPIP_CORE();
        } else {
            NET_PERF_STATS_INC(nic_rx_dones);
            break;
        }
    }
}
```

---

## 3. RX: Ingress Filter 执行上下文

### 3.1 RX 路径

```
NIC DMA 接收
    │
    ▼
nic_rx_thread (seL4_Recv)
    │
    ▼
elem_ring_get(used_rx_buf_ring)  // 无锁获取 RX buffer
    │
    ▼
LOCK_TCPIP_CORE()                // 获取全局锁
    │
    ▼
rx_callback(p)                  // 触发协议栈处理
    │
    ├─► ethernet_input(p, netif)
    │     │
    │     ├─► raw_afpacket_input()  // AF-PACKET 捕获
    │     │
    │     └─► ip4_input(p, netif)
    │           │
    │           └─► lwfw_p->ops->ingress_filter(p, inp)
    │                 └─► 检查包、修改、决策
    │
    ▼
UNLOCK_TCPIP_CORE()              // 释放全局锁
```

### 3.2 Ingress Filter 调用点

**文件**: `core/ipv4/ip4.c:740-790`

```c
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
#if LWFW_TEST_LATENCY
    in_count++;
    freq = raw_read_cntfrq_el0();
    t_start = raw_read_pcnt_el0();
#endif
    if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
      pbuf_free(p);
      return ERR_OK;
    }
#if LWFW_TEST_LATENCY
    t_end = raw_read_pcnt_el0();
    delta_ns = (t_end - t_start) * 1000000000 / freq;
    if (in_count % 1000 == 0) {
      printf("LWFW_TEST_LATENCY INPUT: %lu ns for ip4_filter_dispatch_incoming\n", delta_ns);
    }
#endif
  }
#endif
```

### 3.3 执行上下文总结

| 特性 | RX Ingress Filter |
|------|-------------------|
| **执行线程** | `nic_rx_thread` |
| **持有锁** | `LOCK_TCPIP_CORE()` |
| **上下文类型** | 中断上下文 (NIC 中断触发) |
| **包来源** | 从 RX ring buffer 获取 |
| **SafeOS 特有** | 是，显式锁保护 |

---

## 4. TX: Egress Filter 执行上下文

### 4.1 TX 路径

```
Application Thread
    │
    ▼
sys_net_sendto() / socket API
    │
    ▼
tcp_output() / udp_output()
    │
    ▼
ip4_output(p, src, dest, ttl, tos, proto)
    │
    ▼
ip4_output_if()
    │
    ├─► 添加 IP Header
    ├─► 计算 IP Checksum
    │
    ▼
lwfw_p->ops->egress_filter(p, netif)  // 第 1104 行
    │
    ▼
netif->output(netif, p, dest)         // 发送到 NIC
```

### 4.2 Egress Filter 调用点

**文件**: `core/ipv4/ip4.c:1096-1122`

```c
#ifdef NIO_LWIP_LWFW
  {
    if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
#if LWFW_TEST_LATENCY
      out_count++;
      freq = raw_read_cntfrq_el0();
      t_start = raw_read_pcnt_el0();
#endif
      err_t ret = lwfw_p->ops->egress_filter(p, netif);
      if (ret != ERR_OK) {
        MIB2_STATS_INC(mib2.ipoutdiscards);
        IP_STATS_INC(ip.drop);
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipoutlwfwdrops);
        return ERR_FW;
      }
#if LWFW_TEST_LATENCY
      t_end = raw_read_pcnt_el0();
      delta_ns = (t_end - t_start) * 1000000000 / freq;
      if (out_count % 1000 == 0) {
        printf("LWFW_TEST_LATENCY OUTPUT: %lu ns for ip4_filter_dispatch_outgoing\n", delta_ns);
      }
#endif
    }
  }
#endif
```

### 4.3 执行上下文总结

| 特性 | TX Egress Filter |
|------|------------------|
| **执行线程** | 应用线程 (调用 socket API 的线程) |
| **持有锁** | 无 (直接调用) |
| **上下文类型** | 进程上下文 |
| **包来源** | 应用生成 |
| **SafeOS 特有** | 否，与标准 lwIP 相同 |

---

## 5. 与 Linux 的对比

### 5.1 Linux Netfilter

Linux 使用 Netfilter hook 点：

```bash
# iptables 规则示例
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT
```

| 特性 | SafeOS LWFW | Linux Netfilter |
|------|-------------|-----------------|
| **hook 位置** | ip4_input / ip4_output | PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING |
| **执行上下文** | 单一线程 | 多个 hook 点 (不同上下文) |
| **锁机制** | 全局 mutex | RCU/per-CPU |
| **连接追踪** | LWCT (独立模块) | conntrack (内核集成) |

### 5.2 架构差异

```
Linux:
  NIC Interrupt → softirq (ksoftirqd) → netfilter hook → socket buffer

SafeOS:
  NIC Interrupt → nic_rx_thread (seL4 IPC) → LOCK → lwfw ingress_filter → ip4_input
```

---

## 6. 连接追踪 LWCT 交互

### 6.1 LWCT Hook 位置

LWCT (LightWeight Connection Tracking) 在 LWFW 之前执行：

**文件**: `core/ipv4/ip4.c:739`

```c
#ifdef NIO_LWIP_LWCT
  if (lwct_enable)
    lwct_main_hook(p, LWFW_DIR_RX);
#endif

#ifdef NIO_LWIP_LWFW
  // ... ingress_filter ...
#endif
```

### 6.2 执行顺序

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

---

## 7. 关键设计

### 7.1 为什么 RX 需要锁而 TX 不需要？

**RX (需要锁)**:
- `nic_rx_thread` 是单一消费者
- `ethernet_input` → `ip4_input` 会遍历和修改共享数据结构 (netif_list, raw_pcbs, udp_pcbs, tcp_pcbs)
- 锁保护这些链表的并发访问

**TX (不需要锁)**:
- 每个 TCP/UDP connection 的 PCB 只有 owner 线程访问
- 应用线程直接调用 `tcp_output`/`udp_output`，不需要经过 `tcpip_thread`
- 实际上 TCP TX 操作本身是线程安全的 (基于 PCB 状态机)

### 7.2 Zero-Copy 考虑

SafeOS 使用 CMA buffer 和 elem_ring 实现 zero-copy DMA：
- RX: NIC DMA → CMA buffer → 直接传递给协议栈 (pbuf 引用)
- TX: 应用数据 → pbuf → NIC DMA

LWFW filter 在这两个路径上都直接操作 pbuf，无需复制。

### 7.3 性能优化机会

1. **Ingress Filter 并行化**: 当前 ingress_filter 在全局锁下执行，无法并行
2. **Batch 处理**: 多个包可以在单次锁持有期间处理
3. **RCU 优化**: 考虑使用 RCU 替代 mutex 提升读取并发性

---

## 8. 总结

### 8.1 Filter 执行上下文

```
┌─────────────────────────────────────────────────────────────────┐
│                        RX Path                                  │
│  nic_rx_thread → LOCK → rx_callback → ingress_filter → ip4_input│
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        TX Path                                  │
│  Application → tcp_output/udp_output → egress_filter → netif->output │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 锁使用对比

| 路径 | Filter | 锁 | 上下文 |
|------|--------|-----|--------|
| **RX** | ingress_filter | LOCK_TCPIP_CORE | 中断 (nic_rx_thread) |
| **TX** | egress_filter | 无 | 进程 (应用线程) |

### 8.3 关键点

1. **RX 路径**: `nic_rx_thread` 在持有 `LOCK_TCPIP_CORE()` 时调用 `ingress_filter`
2. **TX 路径**: `egress_filter` 在应用线程上下文中直接调用，无锁保护
3. **LWCT 优先**: LWCT 在 LWFW 之前执行，更新连接状态
4. **SafeOS 特有**: 与标准 lwIP 不同，SafeOS 使用显式锁保护 RX 处理
