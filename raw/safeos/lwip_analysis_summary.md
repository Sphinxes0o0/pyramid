# SafeOS lwIP + LWFW 分析文档汇总

> 更新时间: 2026/04/22
> 状态: 持续更新中

---

## 已完成文档清单

### Phase 1: 基础设施 (T-001 ~ T-003)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_cma_buffer_analysis.md` | T-001 | CMA (Contiguous Memory Area) 缓冲区分配、pbuf 映射、DMA 共享机制 | ✅ |
| `lwip_elem_ring_analysis.md` | T-002 | 无锁单生产者/单消费者环形缓冲区实现、内存屏障、边界条件 | ✅ |
| `lwip_sel4_ipc_analysis.md` | T-003 | seL4 通知机制 (notification)、endpoint 通信、badge 机制 | ✅ |

### Phase 2: lwIP 核心 (T-010 ~ T-052)

#### netif 层 (T-010 ~ T-013)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_netif_analysis.md` | T-010 | struct netif 所有字段详解、client_data 机制、netif_list/netif_default 链表管理 | ✅ |
| `lwip_netif_add_analysis.md` | T-011 | netif_add 注册流程、netif_get_by_index 查找、编号分配 | ✅ |
| `lwip_ethernet_input_analysis.md` | T-012 | Ethernet header 解析、VLAN tag 处理、L2→L3 分发 | ✅ |
| `lwip_ethernet_output_analysis.md` | T-013 | L3→L2 封装、VLAN tag 插入、AF-PACKET 捕获 | ✅ |

#### IP 层 (T-020 ~ T-024)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_ip4_input_analysis.md` | T-020 | IP header 解析、checksum 校验、netif 选择、socket 匹配 | ✅ |
| `lwip_ip4_output_analysis.md` | T-021 | IP 封装、路由查找、egress filter hook | ✅ |
| `lwip_routing_analysis.md` | T-022 | 路由表结构、默认网关、multicast routing、ip4_route | ✅ |

#### TCP 层 (T-030 ~ T-034)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_tcp_input_analysis.md` | T-030 | TCP 状态机、segment 重组、out-of-order 处理、拥塞控制 | ✅ |
| `lwip_tcp_output_analysis.md` | T-031 | 拥塞控制、慢启动、快重传、nagle 算法、segment 发送 | ✅ |
| `lwip_tcp_socket_analysis.md` | T-032 | TCP listen/accept/connect/close 流程、状态转换 | ✅ |
| `lwip_tcp_pcb_analysis.md` | T-033 | TCP PCB 结构、timer 管理、重传队列、拥塞状态 | ✅ |

#### UDP 层 (T-040 ~ T-043)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_udp_input_analysis.md` | T-040 | UDP header 解析、socket 匹配、broadcast/multicast 处理 | ✅ |
| `lwip_udp_output_analysis.md` | T-041 | UDP 封装、checksum 计算、udp_output 函数 | ✅ |
| `lwip_udp_socket_analysis.md` | T-042 | UDP PCB 管理、bind/connect 流程、socket 匹配 | ✅ |

#### RAW/IGMP (T-050, T-051)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_raw_socket_analysis.md` | T-050 | RAW socket 实现、AF-PACKET 绑定、raw_pcb 管理和回调 | ✅ |
| `lwip_igmp_analysis.md` | T-051 | IGMPv1/v2/v3 处理、group management、多播组加入/离开 | ✅ |

#### pbuf/内存 (T-112)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_pbuf_analysis.md` | T-112 | pbuf 结构、pool/heap 分配、refcount 机制、链表不变量 | ✅ |

#### VLAN/分发 (T-052, T-060~T-065)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_arp_filter_netif_fn_analysis.md` | T-052 | VLAN-aware netif 选择、ARP filter、packet 分发核心 | ✅ |
| `lwip_vlan_parsing_analysis.md` | T-060 | IEEE 802.1Q VLAN Tag 解析、TPID/TCI 结构、VID/PCP 提取 | ✅ |
| `lwip_vlan_hook_analysis.md` | T-062/T-063 | MAC_VLAN_FILTER hook、TX VLAN tag 插入 | ✅ |
| `lwip_bridgeif_analysis.md` | T-064 | 802.1D bridge 实现、FDB 学习/老化 | ✅ |
| `lwip_virt_brg_analysis.md` | T-065 | VIRT_BRG_SUPPORT 集成、port_input/port_output | ✅ |

### Phase 4: LWFW (T-070 ~ T-083)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwfw_analysis/lwfw_architecture.md` | T-070 | LWFW 模块划分、filter chain、hook 点 | ✅ |
| `lwfw_lwct_analysis.md` | T-071 | LWCT 连接追踪、状态机、哈希表、GC 线程 | ✅ |
| `lwfw_classification_analysis.md` | T-072 | Packet classification、5-tuple 匹配、规则匹配流程 | ✅ |
| `lwfw_stats_analysis.md` | T-073 | 统计计数、byte/packet counter | ✅ |
| `lwfw_tcpip_thread_analysis.md` | T-082 | Filter 执行上下文、锁机制 | ✅ |
| `lwip_lwfw_filter_hooks_analysis.md` | T-080/T-081 | LWFW Ingress/Egress filter hooks 在 ip4_input/ip4_output 中的集成点 | ✅ |

### Phase 5: NSv/Socket API (T-090)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_nsv_event_loop_analysis.md` | T-090 | NSv 主事件循环、select/poll 实现、socket 事件分发 | ✅ |
| `lwip_packet_mmap_analysis.md` | T-091 | AF-PACKET mmap 实现、ring buffer、零拷贝机制 | ✅ |
| `lwip_ipcif_analysis.md` | T-092 | VNET_OVER_IPC_SUPPORT、VM 通信 | ✅ |
| `lwip_virt_brg_analysis.md` | T-093 | VIRT_BRG_SUPPORT 与 hypervisor 交互 | ✅ |

### Phase 5: Socket API (T-100~T-105)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_sys_net_socket_api_analysis.md` | T-100~T-104 | Socket 创建/bind/listen/accept/connect/close | ✅ |
| `lwip_sys_net_send_recv_analysis.md` | T-102 | sys_net_sendto/recvfrom 数据传输、shm 优化 | ✅ |
| `lwip_sys_net_ctl_analysis.md` | T-105 | netstat/ifconfig/lwfwcfg 等控制命令 | ✅ |

### 系统集成 (T-110~T-114)

| 文档 | 任务 ID | 描述 | 完成度 |
|------|---------|------|--------|
| `lwip_tcpip_thread_analysis.md` | T-110 | 协议栈线程、mbox 机制、LWIP_TCPIP_CORE_LOCKING | ✅ |
| `lwip_network_init_analysis.md` | T-114 | lwip_init → tcpip_init → netif_add → vlanif_setup 初始化流程 | ✅ |

---

## 核心分析文档 (深度分析)

### 网络栈整体分析

| 文档 | 描述 |
|------|------|
| `lwip_vlan_dispatch_analysis.md` | VLAN packet 分发机制 vs Linux |
| `lwip_vlan_dispatch_deep_analysis.md` | VLAN 分发深入分析：模块、函数、设计哲学 |
| `lwip_sel4_performance_boundary.md` | seL4 + lwIP 性能边界分析 |
| `lwip_firewall_analysis.md` | lwIP 防火墙分析 |
| `lwip_ip_fragmentation_analysis.md` | IP 分片与重组分析 |
| `lwip_tcp_recv_queue_analysis.md` | TCP 接收队列、backlog、zero-window |
| `lwip_dhcp_analysis.md` | DHCP client 实现、地址获取、renewal |

### LWFW 详细文档

| 文档 | 描述 |
|------|------|
| `docs/lwfw_analysis/lwfw_architecture.md` | LWFW 整体架构深度分析 |
| `docs/lwfw_analysis/lwfw_core_filtering.md` | LWFW 核心过滤流程 |
| `docs/lwfw_analysis/lwfw_data_structure.md` | LWFW 数据结构 |
| `docs/lwfw_analysis/lwfw_filter_flow.md` | LWFW 过滤流程 |
| `docs/lwfw_analysis/lwfw_optimization.md` | LWFW 优化建议 |

---

## 核心调用链总结

### RX (接收) 路径

```
NIC DMA
    │
    ▼
rx_callback (main.c)
    │
    ▼
elem_ring_get(used_rx_buf_ring)
    │
    ▼
LOCK_TCPIP_CORE()
    │
    ▼
rx_callback → ethernet_input(p, &vnet_if)
    │
    ├─► LWIP_ARP_FILTER_NETIF → lwip_arp_filter_netif_fn
    │     └─► 选择 netif (vnet_if 或 vlan_if[i])
    │
    ├─► ETHARP_SUPPORT_VLAN → 解析 VLAN Tag
    │     └─► type = vlan->tpid, p->priority = PCP
    │
    ├─► raw_afpacket_input → AF-PACKET 捕获
    │
    └─► switch(type)
          ├─► ETHTYPE_IP → ip4_input(p, netif)
          │     │
          │     ├─► LWIP_HOOK_IP4_INPUT
          │     │
          │     ├─► LWFW ingress_filter
          │     │
          │     └─► tcp_input() / udp_input()
          │
          └─► ETHTYPE_ARP → etharp_input()
UNLOCK_TCPIP_CORE()
```

### TX (发送) 路径

```
Application
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
    ├─► ip4_route_src() → 路由查找，找 netif
    │
    ▼
ip4_output_if()
    │
    ├─► 添加 IP Header
    ├─► 计算 IP Checksum
    ├─► LWFW egress_filter
    ├─► [分片处理]
    └─► netif->output(netif, p, dest)
          │
          └─► etharp_output()
                │
                ▼
              ethernet_output(netif, p, src, dst, ETHTYPE_IP)
                    │
                    ├─► [可选] LWIP_HOOK_VLAN_SET → 插入 VLAN Tag
                    ├─► 添加 Ethernet Header
                    ├─► raw_afpacket_output → AF-PACKET 捕获
                    └─► netif->linkoutput(netif, p)
                          │
                          ▼
                        ethif_link_output()
                          │
                          ▼
                        elem_ring_put(pending_tx_buf_ring)
                        sel4_signal(nic_tx_ntfn)
```

---

## 待完成任务

### Phase 2: lwIP 核心

| 任务 ID | 描述 | 优先级 |
|---------|------|--------|
| T-011 | netif_add - netif 注册到全局链表、netif_get_by_index | P1 | ✅ |
| T-022 | Routing - 路由表结构、default gateway、multicast routing | P1 | ✅ |
| T-023 | Fragmentation - IP 分片重组、mtu 发现 | P2 | ✅ |
| T-024 | ip4_frag - 分片重组队列管理、超时处理 | P2 | ✅ |
| T-031 | tcp_output - 拥塞控制、慢启动、快重传、nagle 算法 | P0 | ✅ |
| T-032 | tcp_socket - listen/accept/connect/close 流程 | P1 | ✅ |
| T-033 | tcp_pcb - TCP PCB 结构、timer 管理、重传队列 | P1 | ✅ |
| T-034 | tcp_in_q - TCP 接收队列、backlog、zero-window | P2 | ✅ |
| T-041 | udp_output - UDP 封装、checksum 计算 | P1 | ✅ |
| T-042 | udp_socket - UDP PCB 管理、bind/connect 流程 | P1 | ✅ |
| T-043 | DHCP - DHCP client 实现、地址获取、renewal | P2 | ✅ |
| T-050 | raw_pcb - RAW socket 实现、AF-PACKET 绑定 | P1 | ✅ |
| T-051 | igmp_input - IGMPv1/v2/v3 处理、group management | P1 | ✅ |

### Phase 3: VLAN/Bridge

| 任务 ID | 描述 | 优先级 |
|---------|------|--------|
| T-061 | VLAN 分发 - VLAN ID → netif 映射机制 | P0 | ✅ |
| T-062 | lwip_hook_vlan_check - MAC_VLAN_FILTER hook 实现 | P1 | ✅ |
| T-063 | lwip_hook_vlan_set - TX VLAN tag 插入 hook | P1 | ✅ |
| T-064 | bridgeif - 802.1D bridge 实现、FDB 学习/老化 | P1 | ✅ |
| T-065 | bridge_port - VIRT_BRG_SUPPORT 集成、port_input/port_output | P1 | ✅ |

### Phase 4: LWFW

| 任务 ID | 描述 | 优先级 | 状态 |
|---------|------|--------|------|
| T-070 | LWFW 架构概述 - LWFW 模块划分、filter chain、hook 点 | P0 | ✅ |
| T-071 | lwfw_ct - Connection tracking 表、状态机 (NEW/ESTABLISHED) | P0 | ✅ |
| T-072 | lwfw_classify - Packet classification、5-tuple 匹配 | P1 | ✅ |
| T-073 | lwfw_stats - 统计计数、byte/packet counter | P2 | ✅ |
| T-082 | LWFW 与 tcpip_thread - Filter 执行上下文、锁机制 | P1 | ✅ |
| T-083 | lwfw_config - YAML 配置解析、rule 加载 | P1 | ✅ |

### Phase 5: Socket API / NSv

| 任务 ID | 描述 | 优先级 |
|---------|------|--------|
| T-090 | NSv event_loop - 主事件循环、select/poll 实现 | P0 | ✅ |
| T-091 | packet_mmap - AF-PACKET mmap 实现、ring buffer | P0 | ✅ |
| T-092 | ipc-if - VNET_OVER_IPC_SUPPORT、VM 通信 | P1 | ✅ |
| T-093 | VMM/bridge 集成 - VIRT_BRG_SUPPORT 与 hypervisor 交互 | P1 | ✅ |
| T-100 | sys_net_socket - socket 创建、协议族、类型 | P1 | ✅ |
| T-101 | sys_net_bind - bind 实现、port 分配 | P1 | ✅ |
| T-102 | sys_net_sendto/recvfrom - 数据传输、shm 优化 | P0 | ✅ |
| T-103 | sys_net_connect - connect 流程、TCP 握手触发 | P1 | ✅ |
| T-104 | sys_net_accept - accept 队列、backlog | P1 | ✅ |
| T-105 | sys_net_ctl - netstat/ifconfig/lwfwcfg 等控制命令 | P1 | ✅ |

### Phase 6: 内存/pbuf

| 任务 ID | 描述 | 优先级 |
|---------|------|--------|
| T-112 | pbuf 管理层 - pbuf 结构、pool/heap/malloc、refcount | P0 | ✅ |
| T-113 | lwip_malloc - 内存池初始化、分配策略 | P0 | ✅ |
| T-114 | 网络初始化流程 - lwip_init → tcpip_init → netif_add → vlanif_setup | P0 | ✅ |

---

## 架构图

### lwIP + LWFW 在 SafeOS 中的位置

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Application Layer                                 │
│                        (iperf, ping, lwfwcfg)                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          NSv Network Server                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Socket API Layer                                │   │
│  │         sys_net_socket / sys_net_bind / sys_net_sendto             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      LWFW (Lightweight Firewall)                    │   │
│  │         ingress_filter / egress_filter / connection tracking          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      lwIP Protocol Stack                            │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │   │
│  │  │   TCP   │ │   UDP   │ │  Raw    │ │  IGMP   │ │   DNS   │     │   │
│  │  │ tcp_in  │ │ udp_in  │ │  pcb    │ │ igmp_in │ │         │     │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐     │   │
│  │  │              IP Layer (ipv4)                                │     │   │
│  │  │   ip4_input ────► LWFW ingress_filter                      │     │   │
│  │  │   ip4_output ◄─── LWFW egress_filter                      │     │   │
│  │  └─────────────────────────────────────────────────────────────┘     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐     │   │
│  │  │              L2 Layer (ethernet/arp/vlan)                 │     │   │
│  │  │   ethernet_input ───► lwip_arp_filter_netif_fn (VLAN)      │     │   │
│  │  │   ethernet_output ◄── lwip_hook_vlan_set (VLAN)          │     │   │
│  │  └─────────────────────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      NSv Network Virtualization                      │   │
│  │   packet_mmap / AF-PACKET / VIRT_BRG / VNET_OVER_IPC                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    DMA / elem_ring Layer                            │   │
│  │              CMA buffer / lock-free ring buffer                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         seL4 Microkernel                                   │
│                         (IPC, memory, threads)                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NIC Driver (PFE)                                   │
│                         (DMA, interrupts, RX/TX)                          │
└─────────────────────────────────────────────────────────────────────────────┘
```
