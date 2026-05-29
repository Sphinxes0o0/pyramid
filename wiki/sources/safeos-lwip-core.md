---
type: source
source-type: github
created: 2026-05-25
title: "SafeOS lwIP Core Network Protocol Analysis"
author: "SafeOS Team"
date: 2026-04-22
size: large
path: raw/safeos/
summary: "SafeOS NSv 网络栈中 lwIP 核心协议分析文档 (~28 篇)，涵盖 netif/pbuf/内存管理、TCP/UDP/IGMP/DHCP 协议、VLAN 分发机制、LWFW 防火墙、seL4 IPC 集成"
tags: [lwip, network, embedded, safeos, tcpip]
---

# SafeOS lwIP Core Network Protocol Analysis

## 概述

SafeOS NSv 网络服务器使用 lwIP 作为核心协议栈，运行在 seL4 微内核之上。本文档汇总了 ~28 篇深度分析文档，覆盖 lwIP 各层实现细节。

## 核心主题

### 网络接口层 (netif)
- [[entities/linux/lwip/lwip-netif]] — struct netif 结构、链表管理、client_data 机制
- [[entities/linux/lwip/lwip-netif-add]] — netif_add 注册流程、netif_get_by_index 查找、编号分配 O(n²)
- [[entities/linux/lwip/lwip-ethernet-input]] — L2→L3 入口、VLAN tag 解析、LWIP_ARP_FILTER_NETIF 分发
- [[entities/linux/lwip/lwip-ethernet-output]] — L3→L2 封装、VLAN tag 插入、AF-PACKET 捕获

### 内存管理层
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 结构、pool/heap/ROM/ref 分配、refcount 机制、链表不变量
- [[entities/linux/lwip/lwip-malloc]] — memp 内存池、mem 堆、首次适应算法、线程安全

### IPv4 层
- [[entities/linux/lwip/lwip-ip4-input]] — IP header 解析、checksum 校验、netif 选择、LWFW ingress hook
- [[entities/linux/lwip/lwip-ip4-output]] — IP 封装、路由查找、LWFW egress hook、IP fragmentation
- [[entities/linux/lwip/lwip-routing]] — 路由机制（无独立路由表，netif 链表遍历）、ip4_route、默认网关
- [[entities/linux/lwip/lwip-ip-fragmentation]] — IP 分片重组、MTU 发现、重组队列超时

### TCP 层
- [[entities/linux/lwip/lwip-tcp-input]] — TCP 状态机、segment demultiplex、ooseq 队列、拥塞控制
- [[entities/linux/lwip/lwip-tcp-output]] — 拥塞窗口、Nagle 算法、重传定时器、快速重传
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 结构、timer 管理、重传队列、状态转换
- [[entities/linux/lwip/lwip-tcp-recv-queue]] — rcv_wnd、ooseq、zero-window、backlog 机制
- [[entities/linux/lwip/lwip-tcp-socket]] — listen/accept/connect/close 流程、三次握手、四次挥手
- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread 实现、LWIP_TCPIP_CORE_LOCKING、mbox 机制

### UDP 层
- [[entities/linux/lwip/lwip-udp-input]] — UDP header 解析、socket 匹配、broadcast/multicast
- [[entities/linux/lwip/lwip-udp-output]] — UDP 封装、checksum pseudo-header、udp_output
- [[entities/linux/lwip/lwip-udp-socket]] — UDP PCB 管理、bind/connect、socket 匹配规则

### 其他协议
- [[entities/linux/lwip/lwip-igmp]] — IGMPv1/v2/v3、group management、多播组加入/离开
- [[entities/linux/lwip/lwip-dhcp]] — DHCP client 状态机、地址获取、租约续约 T1/T2

### VLAN 分发 (SafeOS 特供)
- [[entities/linux/lwip/lwip-vlan-dispatch]] — lwIP vs Linux VLAN 分发机制对比
- [[entities/linux/lwip/lwip-vlan-dispatch-deep]] — VLAN 分发深度分析：模块、函数、设计哲学
- [[entities/linux/lwip/lwip-vlan-hook]] — LWIP_HOOK_VLAN_CHECK (RX)、LWIP_HOOK_VLAN_SET (TX)
- [[entities/linux/lwip/lwip-vlan-implementation]] — IEEE 802.1Q 实现、VLAN netif 创建、配置解析
- [[entities/linux/lwip/lwip-vlan-parsing]] — VLAN Tag 结构 (TPID/TCI)、VID/PCP 提取

### 系统集成
- [[entities/linux/lwip/lwip-network-init]] — lwip_init → tcpip_init → netif_add → vlanif_setup 完整初始化流程

## 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Application Layer                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          NSv Network Server                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      LWFW (Lightweight Firewall)                    │   │
│  │         ingress_filter / egress_filter / connection tracking          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      lwIP Protocol Stack                            │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │   │
│  │  │   TCP   │ │   UDP   │ │  Raw    │ │  IGMP   │ │   DNS   │     │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐     │   │
│  │  │              IP Layer (ipv4)                                │     │   │
│  │  │   ip4_input ────► LWFW ingress_filter                      │     │   │
│  │  │   ip4_output ◄─── LWFW egress_filter                      │     │   │
│  │  └─────────────────────────────────────────────────────────────┘     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐     │   │
│  │  │              L2 Layer (ethernet/arp/vlan)                 │     │   │
│  │  │   ethernet_input ───► lwip_arp_filter_netif_fn (VLAN)      │     │   │
│  │  │   ethernet_output ◄── lwip_hook_vlan_set (VLAN)           │     │   │
│  │  └─────────────────────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    DMA / elem_ring Layer                            │   │
│  │              CMA buffer / lock-free ring buffer                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         seL4 Microkernel                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 关键设计决策

### 1. LWIP_TCPIP_CORE_LOCKING = 1
RX packet 在 `nic_rx_thread` 中直接处理，无需通过 mbox 传递，延迟更低但单线程瓶颈。

### 2. VLAN 分发机制
所有 packet 经过 `vnet_if.input = ethernet_input`，通过 `LWIP_ARP_FILTER_NETIF` + VLAN ID 匹配找到正确的 vlan_if[i]。

### 3. 无独立路由表
lwIP 的路由直接编码在 netif 结构中 (ip_addr, netmask, gw)，通过 netif_list 遍历匹配。

### 4. O(n) PCB 查找
TCP/UDP PCB 查找都是链表遍历，无哈希表，适合连接数少的嵌入式场景。

## 相关页面

- [[entities/linux/kernel/index#networking]] — Linux 内核网络子系统
- [[entities/linux/kernel/index#network-protocols--physical-layer]] — 网络协议与物理层
- [[entities/linux/lwip/lwip-vlan-dispatch]] — 与 Linux VLAN 分发机制对比
