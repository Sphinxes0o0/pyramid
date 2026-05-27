---
type: source
source-type: github
created: 2026-05-27
title: "SafeOS lwIP 深度分析文档汇总"
date: 2026-04-22
size: medium
path: raw/safeos/docs/lwip_analysis_summary.md
summary: "SafeOS lwIP + LWFW 完整分析文档清单：9阶段计划、80+篇分析、任务ID T-001~T-114、核心调用链 RX/TX 路径"
tags: [safeos, lwip, lwfw, seL4, network, cma, elem-ring, vlan, packet-mmap, firewall]
sources: []
---

# SafeOS lwIP 深度分析文档汇总

> 更新时间: 2026/04/22 | 状态: 持续更新中

## 文档清单

本文档是 SafeOS lwIP + LWFW 深度分析的**总索引**，覆盖 9 个阶段、80+ 篇分析文档。

### Phase 1: 基础设施 (T-001 ~ T-003)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_cma_buffer_analysis.md` | T-001 | CMA 缓冲区分配、pbuf 映射、DMA 共享 | ✅ |
| `lwip_elem_ring_analysis.md` | T-002 | 无锁单生产者/单消费者环形缓冲区 | ✅ |
| `lwip_sel4_ipc_analysis.md` | T-003 | seL4 通知机制、endpoint 通信 | ✅ |

### Phase 2: lwIP 核心 (T-010 ~ T-065)

| 文档 | 任务 ID | 描述 |
|------|---------|------|
| `lwip_netif_analysis.md` | T-010 | struct netif 所有字段详解 |
| `lwip_netif_add_analysis.md` | T-011 | netif_add 注册流程 |
| `lwip_ethernet_input_analysis.md` | T-012 | Ethernet header 解析、VLAN tag 处理 |
| `lwip_ethernet_output_analysis.md` | T-013 | L3→L2 封装、VLAN tag 插入 |
| `lwip_ip4_input_analysis.md` | T-020 | IP header 解析、checksum 校验 |
| `lwip_ip4_output_analysis.md` | T-021 | IP 封装、路由查找 |
| `lwip_routing_analysis.md` | T-022 | 路由表结构、默认网关 |
| `lwip_tcp_input_analysis.md` | T-030 | TCP 状态机、segment 重组 |
| `lwip_tcp_output_analysis.md` | T-031 | 拥塞控制、慢启动 |
| `lwip_tcp_socket_analysis.md` | T-032 | TCP listen/accept/connect/close |
| `lwip_tcp_pcb_analysis.md` | T-033 | TCP PCB 结构、timer 管理 |
| `lwip_udp_input_analysis.md` | T-040 | UDP header 解析、socket 匹配 |
| `lwip_udp_output_analysis.md` | T-041 | UDP 封装、checksum |
| `lwip_udp_socket_analysis.md` | T-042 | UDP PCB 管理 |
| `lwip_raw_socket_analysis.md` | T-050 | RAW socket 实现、AF-PACKET 绑定 |
| `lwip_igmp_analysis.md` | T-051 | IGMPv1/v2/v3 处理 |
| `lwip_arp_filter_netif_fn_analysis.md` | T-052 | VLAN-aware netif 选择 |
| `lwip_pbuf_analysis.md` | T-112 | pbuf 结构、pool/heap 分配 |
| `lwip_vlan_parsing_analysis.md` | T-060 | IEEE 802.1Q VLAN Tag 解析 |
| `lwip_vlan_hook_analysis.md` | T-062/T-063 | MAC_VLAN_FILTER hook |
| `lwip_bridgeif_analysis.md` | T-064 | 802.1D bridge 实现 |
| `lwip_virt_brg_analysis.md` | T-065 | VIRT_BRG_SUPPORT 集成 |

### Phase 4: LWFW (T-070 ~ T-083)

| 文档 | 任务 ID | 描述 |
|------|---------|------|
| `lwfw_architecture.md` | T-070 | LWFW 模块划分、filter chain |
| `lwfw_lwct_analysis.md` | T-071 | LWCT 连接追踪、状态机 |
| `lwfw_classification_analysis.md` | T-072 | Packet classification、5-tuple |
| `lwfw_stats_analysis.md` | T-073 | 统计计数 |
| `lwip_lwfw_filter_hooks_analysis.md` | T-080/T-081 | LWFW Ingress/Egress filter hooks |

### Phase 5: NSv/Socket API (T-090 ~ T-105)

| 文档 | 任务 ID | 描述 |
|------|---------|------|
| `lwip_nsv_event_loop_analysis.md` | T-090 | NSv 主事件循环、select/poll |
| `lwip_packet_mmap_analysis.md` | T-091 | AF-PACKET mmap 实现 |
| `lwip_ipcif_analysis.md` | T-092 | VNET_OVER_IPC_SUPPORT |
| `lwip_sys_net_socket_api_analysis.md` | T-100~T-104 | Socket 创建/bind/listen/accept |
| `lwip_sys_net_send_recv_analysis.md` | T-102 | sys_net_sendto/recvfrom |
| `lwip_sys_net_ctl_analysis.md` | T-105 | netstat/ifconfig/lwfwcfg |
| `lwip_tcpip_thread_analysis.md` | T-110 | tcpip_thread、mbox 机制 |
| `lwip_network_init_analysis.md` | T-114 | lwip_init → tcpip_init → netif_add |

---

## 核心调用链

### RX 路径

```
NIC DMA
  │
  ▼
rx_callback() → elem_ring_get(used_rx_buf_ring)
  │
  ▼
LOCK_TCPIP_CORE()
  │
  ▼
ethernet_input() → LWIP_ARP_FILTER_NETIF → ip4_input()
  │
  ├─► ETHTYPE_IP → ip4_input() → LWFW ingress_filter → tcp_input()
  └─► ETHTYPE_ARP → etharp_input()
UNLOCK_TCPIP_CORE()
```

### TX 路径

```
App sendto()
  │
  ▼
tcp_output() / udp_output() → ip4_output()
  │
  ▼
LWFW egress_filter
  │
  ▼
netif->output = etharp_output() → ethernet_output()
  │
  ▼
LWIP_HOOK_VLAN_SET → netif->linkoutput()
  │
  ▼
elem_ring_put(pending_tx_buf_ring) → sel4_signal(nic_tx_ntfn)
```

---

## 架构图

```
┌─────────────────────────────────────────────┐
│  Application (iperf, ping, lwfwcfg)         │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│  NSv Network Server (lwIP)                  │
│  ┌───────────────────────────────────────┐  │
│  │ Socket API Layer (sys_net_*)          │  │
│  └───────────────────────────────────────┘  │
│                    │                          │
│  ┌───────────────────────────────────────┐  │
│  │ LWFW (ingress/egress filter)         │  │
│  └───────────────────────────────────────┘  │
│                    │                          │
│  ┌───────────────────────────────────────┐  │
│  │ TCP/UDP/Raw/IGMP/DNS                  │  │
│  │ IP Layer (ip4_input/output)            │  │
│  │ L2 Layer (ethernet/arp/vlan)          │  │
│  └───────────────────────────────────────┘  │
│                    │                          │
│  ┌───────────────────────────────────────┐  │
│  │ DMA / elem_ring Layer (CMA 96MB)      │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│  seL4 Microkernel (IPC, memory, threads)    │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│  NIC Driver (PFE)                          │
└─────────────────────────────────────────────┘
```

---

## 相关页面

### lwIP 核心
- [[sources/safeos-lwip-core]] — lwIP 协议栈分析 (~28篇)
- [[sources/safeos-lwip-extensions]] — lwIP 扩展集成 (19篇)
- [[lwip-index]] — lwIP 模块完整索引

### LWFW 防火墙
- [[sources/safeos-lwfw]] — LWFW 防火墙分析 (27篇)
- [[lwfw-index]] — LWFW 模块索引

### 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引
- [[lwip-index]] — lwIP 索引 (27 entities)
