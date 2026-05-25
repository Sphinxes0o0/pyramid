---
type: entity
tags: [linux, lwip, network, vlan]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP VLAN Dispatch Deep Analysis

## 定义

VLAN packet 在 lwIP 中的完整处理流程深度分析：分发前 (DMA → elem_ring → rx_callback → ethernet_input)、分发点 (VLAN 解析、VLAN ID 匹配、netif 选择)、分发后 (ip4_input → UDP/TCP → socket)。

## VLAN 分发完整路径

```
┌─────────────────────────────────────────────────────────────┐
│                    VLAN Packet 分发前 (NIC → ethernet_input) │
├─────────────────────────────────────────────────────────────┤
│  物理网卡 DMA → elem_ring → nic_rx_thread → rx_callback     │
│      → vnet_if.input = ethernet_input                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    VLAN 分发点 (ethernet_input)             │
├─────────────────────────────────────────────────────────────┤
│  1. LWIP_ARP_FILTER_NETIF (line 121-127)                   │
│     └─► lwip_arp_filter_netif_fn()                         │
│           ├─► ETHTYPE_VLAN: 通过 netif->vlanid == VID 匹配 │
│           ├─► ETHTYPE_IP:   通过 IP 地址 + vlanid==0 匹配  │
│           └─► 返回正确的 netif (vnet_if 或 vlan_if[i])       │
│                                                                     │
│  2. ETHARP_SUPPORT_VLAN (line 133-168)                           │
│     └─► 解析 VLAN tag，更新 type 和 p->priority                   │
│                                                                     │
│  3. MAC_VLAN_FILTER (line 146-158) [可选]                        │
│     └─► 额外的 VLAN ID 安全检查                                    │
│                                                                     │
│  4. raw_afpacket_input() (line 202-205)                          │
│     └─► AF_PACKET socket 捕获                                      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    VLAN 分发后 (ip4_input → socket)          │
├─────────────────────────────────────────────────────────────┤
│  ip4_input(p, netif=vlan_if[i])                            │
│     └─► ip_data.current_input_netif = vlan_if[i]           │
│           ↓                                                  │
│  UDP/TCP: udp_input_local_match()                          │
│     └─► 检查 pcb->netif_idx == vlan_if[i]->index           │
│           ↓                                                  │
│  Socket 接收队列                                              │
└─────────────────────────────────────────────────────────────┘
```

## 设计哲学对比

### lwIP 的核心理念：简单、高效、嵌入式友好

| 设计选择 | 说明 |
|---------|------|
| **netif 平权设计** | 所有 netif 平等，VLAN 只是属性 |
| **IP 路由优先** | VLAN 分发服务于 IP 路由 |
| **单层分发** | 减少函数调用开销 |

### Linux 的核心理念：功能完整、兼容标准

| 设计选择 | 说明 |
|---------|------|
| **设备堆叠模型** | VLAN device 像真实设备一样工作 |
| **VLAN ID 精确匹配** | 确保严格的隔离 |
| **多层处理** | rx_handler 链提供灵活性 |

## 两层过滤机制

**第一层 (`LWIP_ARP_FILTER_NETIF`)**：在 VLAN 解析 **之前** 执行，提前确定正确的 netif

**第二层 (`MAC_VLAN_FILTER`)**：可选的额外安全检查，确保 VLAN ID 精确匹配

## elem_ring 无锁设计

```c
// elem_ring.h - 单生产者/单消费者无锁环形缓冲区
struct elem_ring {
    uint32_t n;
    volatile uint32_t get_idx;  // 消费者索引
    volatile uint32_t put_idx;  // 生产者索引
    union elem elems[0];
};

// 生产者 (NIC Driver): elem_ring_put()
// 消费者 (nic_rx_thread): elem_ring_get()
```

## 关键代码路径

| 功能 | 文件 | 行号 | 函数 |
|------|------|------|------|
| NIC RX 线程 | `main.c` | 4961 | `nic_rx_thread()` |
| RX 回调 | `main.c` | 4781 | `rx_callback()` |
| VLAN 分发入口 | `ethernet.c` | 89 | `ethernet_input()` |
| VLAN netif 选择 | `ethernet.c` | 459 | `lwip_arp_filter_netif_fn()` |
| VLAN tag 解析 | `ethernet.c` | 133 | `if (type == ETHTYPE_VLAN)` |
| IP 层入口 | `ip4.c` | 468 | `ip4_input()` |
| UDP 匹配 | `udp.c` | 151 | `udp_input_local_match()` |

## 相关概念

- [[entities/linux/lwip/lwip-vlan-dispatch]] — VLAN 分发机制概述
- [[entities/linux/lwip/lwip-ethernet-input]] — ethernet_input 完整源码分析
- [[entities/linux/lwip/lwip-ip4-input]] — ip4_input 中的 netif 匹配
- [[entities/linux/lwip/lwip-udp-input]] — udp_input_local_match netif_idx 检查
