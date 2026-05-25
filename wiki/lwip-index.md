---
type: index
tags: [linux, lwip, network, embedded]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP Core Network Protocol — Module Index

> SafeOS NSv 网络栈中 lwIP 核心协议分析 (~28 篇文档)

## Entity Pages

### 网络接口层 (netif/L2)
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-netif]] | struct netif 结构、链表管理 (netif_list)、client_data 机制 |
| [[entities/linux/lwip/lwip-netif-add]] | netif_add 注册流程、netif_get_by_index 查找、编号分配 O(n²) |
| [[entities/linux/lwip/lwip-ethernet-input]] | L2→L3 入口 ethernet_input、VLAN tag 解析、LWIP_ARP_FILTER_NETIF |
| [[entities/linux/lwip/lwip-ethernet-output]] | L3→L2 封装 ethernet_output、VLAN tag 插入 (LWIP_HOOK_VLAN_SET) |

### 内存管理层
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-pbuf]] | pbuf 结构、pool/heap/ROM/ref 分配、refcount 机制、链表不变量 |
| [[entities/linux/lwip/lwip-malloc]] | memp 内存池 (O(1))、mem 堆 (首次适应)、线程安全 (mutex/sys_arch_protect) |

### IPv4 层
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-ip4-input]] | IP header 解析、checksum 校验、LWFW ingress hook、netif 选择 |
| [[entities/linux/lwip/lwip-ip4-output]] | IP 封装、路由查找 ip4_route、LWFW egress hook、IP fragmentation |
| [[entities/linux/lwip/lwip-routing]] | 路由机制（无独立路由表）、ip4_route、默认网关、multicast routing |
| [[entities/linux/lwip/lwip-ip-fragmentation]] | IP 分片 (TX)、重组 (RX)、MTU 发现、重组队列超时 30s |

### TCP 层
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-tcp-input]] | TCP 状态机、segment demultiplex (O(n))、ooseq 队列、拥塞控制 |
| [[entities/linux/lwip/lwip-tcp-output]] | 拥塞窗口 (cwnd)、Nagle 算法、重传定时器 (RTO)、快速重传 |
| [[entities/linux/lwip/lwip-tcp-pcb]] | TCP PCB 结构、timer 管理 (fast/slow 25ms/50ms)、重传队列 |
| [[entities/linux/lwip/lwip-tcp-recv-queue]] | rcv_wnd、ooseq、zero-window probe、backlog 机制 |
| [[entities/linux/lwip/lwip-tcp-socket]] | listen/accept/connect/close、三次握手、四次挥手、TCP 状态转换 |
| [[entities/linux/lwip/lwip-tcpip-thread]] | tcpip_thread 实现、LWIP_TCPIP_CORE_LOCKING=1、mbox 机制 |

### UDP 层
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-udp-input]] | UDP header 解析、socket 匹配 (O(n))、broadcast/multicast 处理 |
| [[entities/linux/lwip/lwip-udp-output]] | UDP 封装、checksum pseudo-header、udp_output 函数 |
| [[entities/linux/lwip/lwip-udp-socket]] | UDP PCB 管理、bind/connect 流程、CONNECTED 标志 |

### 其他协议
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-igmp]] | IGMPv1/v2/v3、group management、多播组加入/离开、MAC 过滤器 |
| [[entities/linux/lwip/lwip-dhcp]] | DHCP client 状态机、地址获取、租约续约 (T1=50%, T2=87.5%) |

### VLAN 分发 (SafeOS 特供)
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-vlan-dispatch]] | lwIP vs Linux VLAN 分发机制对比、架构差异 |
| [[entities/linux/lwip/lwip-vlan-dispatch-deep]] | VLAN 分发深度分析：模块、函数、设计哲学 |
| [[entities/linux/lwip/lwip-vlan-hook]] | LWIP_HOOK_VLAN_CHECK (RX)、LWIP_HOOK_VLAN_SET (TX)、PCP 优先级 |
| [[entities/linux/lwip/lwip-vlan-implementation]] | IEEE 802.1Q 实现、VLAN netif 创建、YAML 配置解析 |
| [[entities/linux/lwip/lwip-vlan-parsing]] | VLAN Tag 结构 (TPID=0x8100/TCI)、VID/PCP 提取 |

### 系统集成
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-network-init]] | lwip_init → tcpip_init → netif_add → vlanif_setup 完整初始化流程 |

## Source Page

- [[sources/safeos-lwip-core]] — SafeOS lwIP Core Network Protocol Analysis (28 篇汇总)

## Related Indexes

- [[kernel-net-index]] — Linux 内核网络子系统 (Socket/sk_buff、Netfilter)
- [[kernel-protocols-index]] — 网络协议与物理层 (TCP/IP、PHY/MAC)
