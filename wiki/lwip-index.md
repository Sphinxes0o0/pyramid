---
type: index
tags: [linux, lwip, network, embedded]
created: 2026-05-25
sources: [safeos-lwip-core, safeos-lwip-extensions]
---

# lwIP — Module Index

> SafeOS NSv 网络栈中 lwIP 协议分析 (~47 篇文档: ~28 Core + 19 Extensions)

## Entity Pages

### 网络接口层 (netif/L2)
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-netif]] | struct netif 结构、链表管理 (netif_list)、client_data 机制 |
| [[entities/linux/lwip/lwip-netif-add]] | netif_add 注册流程、netif_get_by_index 查找、编号分配 O(n²) |
| [[entities/linux/lwip/lwip-ethernet-input]] | L2→L3 入口 ethernet_input、VLAN tag 解析、LWIP_ARP_FILTER_NETIF |
| [[entities/linux/lwip/lwip-ethernet-output]] | L3→L2 封装 ethernet_output、VLAN tag 插入 (LWIP_HOOK_VLAN_SET) |
| [[entities/linux/lwip/lwip-arp-filter-netif-fn]] | VLAN-aware netif 选择、两阶段分发、VID 精确匹配 |

### Bridge / VIRT 网桥
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-bridgeif]] | 802.1D MAC Bridge 实现、FDB 学习/老化、port_input/port_output |
| [[entities/linux/lwip/lwip-virt-brg]] | VIRT_BRG_SUPPORT 与 hypervisor 交互、ethif_link_output_overload |

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

## DMA / CMA / elem_ring 基础设施
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-cma-buffer]] | CMA (Contiguous Memory Area) 缓冲区分配、pbuf 映射、DMA 共享机制 |
| [[entities/linux/lwip/lwip-elem-ring]] | 无锁单生产者/单消费者环形缓冲区、ARM dmb/dsb 内存屏障 |
| [[entities/linux/lwip/lwip-cma-elem-ring]] | CMA (96MB) 与 elem_ring 无锁队列：VA/PA 转换、DMA 共享、dmb(ish) 内存屏障、NSv/NIC 共享 |

## LWFW 防火墙
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-firewall]] | LWFW 无状态包过滤、三层安全架构 (lwfw/lwct/cBPF)、规则匹配 |
| [[entities/linux/lwip/lwip-lwfw-filter-hooks]] | LWFW Ingress/Egress filter hooks 在 ip4_input/ip4_output 中的精确集成点 |

## RAW / packet_mmap / AF-PACKET
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-raw-socket]] | RAW socket 实现、AF-PACKET 绑定、raw_pcb 管理、cBPF 过滤 |
| [[entities/linux/lwip/lwip-packet-mmap]] | AF-PACKET mmap 实现、TPACKET_V1、ring buffer 零拷贝 |

## NSv Socket API
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-nsv-event-loop]] | NSv 主事件循环、select/poll 实现、socket 事件分发、双线程模型 |
| [[entities/linux/lwip/lwip-sys-net-socket-api]] | BSD Socket API：socket/bind/listen/accept/connect/close |
| [[entities/linux/lwip/lwip-sys-net-send-recv]] | sys_net_sendto/recvfrom 数据传输、共享内存优化 |
| [[entities/linux/lwip/lwip-sys-net-ctl]] | netstat/ifconfig/lwfwcfg 等控制命令 |

## VNET_OVER_IPC / IPCIF
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-ipcif]] | VNET_OVER_IPC_SUPPORT、VM 通信、seL4 IPC + 共享内存 |

## seL4 微内核集成
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-sel4-function]] | lwIP 在 seL4 上运行的函数级深度分析：完整调用链 |
| [[entities/linux/lwip/lwip-sel4-interaction]] | lwIP 与 seL4 物理网卡/VLAN/Hypervisor 交互深度分析 |
| [[entities/linux/lwip/lwip-sel4-ipc]] | seL4 notification/endpoint 通信、badge 机制、IPC 延迟 |
| [[entities/linux/lwip/lwip-sel4-performance-boundary]] | seL4 + lwIP 性能边界分析 (~3x 单核性能损失) |

## 概览
| Entity | Description |
|--------|-------------|
| [[entities/linux/lwip/lwip-analysis-summary]] | SafeOS lwIP Extensions 分析文档汇总 |

## Source Pages

- [[sources/safeos-lwip-core]] — SafeOS lwIP Core Network Protocol Analysis (~28 篇汇总)
- [[sources/safeos-lwip-extensions]] — SafeOS lwIP Extensions & Integration (19 篇汇总)

## Related Indexes

- [[kernel-net-index]] — Linux 内核网络子系统 (Socket/sk_buff、Netfilter)
- [[kernel-protocols-index]] — 网络协议与物理层 (TCP/IP、PHY/MAC)
