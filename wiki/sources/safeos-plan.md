---
type: source
source-type: github
created: 2026-05-27
title: "SafeOS lwIP + LWFW 深度分析计划"
date: 2026-04-22
size: small
path: raw/safeos/docs/plan.md
summary: "SafeOS lwIP+LWFW深度分析9阶段计划：T-001~T-114共64个任务，覆盖基础设施/lwIP核心/LWFW/NSv虚拟化/Socket API/系统集成，优先级P0~P2"
tags: [safeos, lwip, lwfw, seL4, plan, analysis]
sources: []
---

# SafeOS lwIP + LWFW 深度分析计划

> 文档版本: 1.0 | 创建日期: 2026/04/22

## 分析范围

1. **lwIP** — seL4 上运行的轻量级 IP 协议栈
2. **LWFW** — Lightweight Firewall，轻量级防火墙/包过滤器

---

## 9 阶段任务分解

### Phase 1: 基础设施 (T-001 ~ T-003)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-001 | CMA Buffer | CMA 缓冲区分配、pbuf 映射、DMA 共享 | P0 |
| T-002 | elem_ring | 无锁单生产者/单消费者环形缓冲区 | P0 |
| T-003 | seL4 IPC | 通知机制、endpoint 通信 | P1 |

### Phase 2: lwIP 核心 (T-010 ~ T-065)

#### netif 层 (T-010 ~ T-013)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-010 | netif 结构 | `struct netif` 所有字段详解 | P0 |
| T-011 | netif_add | netif 注册到全局链表 | P1 |
| T-012 | ethernet_input | Ethernet header 解析、VLAN tag 处理 | P0 |
| T-013 | ethernet_output | L3→L2 封装、VLAN tag 插入 | P0 |

#### IP 层 (T-020 ~ T-024)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-020 | ip4_input | IP header 解析、checksum 校验 | P0 |
| T-021 | ip4_output | IP 封装、路由查找、egress filter hook | P0 |
| T-022 | Routing | 路由表结构、default gateway | P1 |
| T-023 | Fragmentation | IP 分片重组 | P2 |
| T-024 | ip4_frag | 分片重组队列管理 | P2 |

#### TCP 层 (T-030 ~ T-034)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-030 | tcp_input | TCP 状态机、segment 重组 | P0 |
| T-031 | tcp_output | 拥塞控制、慢启动、快重传 | P0 |
| T-032 | tcp_socket | listen/accept/connect/close 流程 | P1 |
| T-033 | tcp_pcb | TCP PCB 结构、timer 管理 | P1 |
| T-034 | tcp_in_q | TCP 接收队列、backlog、zero-window | P2 |

#### UDP 层 (T-040 ~ T-043)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-040 | udp_input | UDP header 解析、socket 匹配 | P0 |
| T-041 | udp_output | UDP 封装、checksum 计算 | P1 |
| T-042 | udp_socket | UDP PCB 管理 | P1 |
| T-043 | DHCP | DHCP client 实现 | P2 |

#### RAW/IGMP (T-050 ~ T-052)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-050 | raw_pcb | RAW socket 实现、AF-PACKET 绑定 | P1 |
| T-051 | igmp_input | IGMPv1/v2/v3 处理 | P1 |
| T-052 | lwip_arp_filter_netif_fn | VLAN-aware netif 选择 | P0 |

#### VLAN/桥接 (T-060 ~ T-065)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-060 | VLAN 解析 | 802.1Q TPID/TCI 解析 | P0 |
| T-061 | VLAN 分发 | VLAN ID → netif 映射 | P0 |
| T-062 | lwip_hook_vlan_check | MAC_VLAN_FILTER hook | P1 |
| T-063 | lwip_hook_vlan_set | TX VLAN tag 插入 | P1 |
| T-064 | bridgeif | 802.1D bridge 实现 | P1 |
| T-065 | bridge_port | VIRT_BRG_SUPPORT 集成 | P1 |

### Phase 4: LWFW (T-070 ~ T-083)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-070 | LWFW 架构 | 模块划分、filter chain | P0 |
| T-071 | lwfw_ct | Connection tracking 表、状态机 | P0 |
| T-072 | lwfw_classify | Packet classification、5-tuple | P1 |
| T-073 | lwfw_stats | 统计计数 | P2 |
| T-080 | LWIP_HOOK_IP4_INPUT | Ingress filter hook | P0 |
| T-081 | LWIP_HOOK_IP4_OUTPUT | Egress filter hook | P0 |
| T-082 | LWFW 与 tcpip_thread | Filter 执行上下文 | P1 |
| T-083 | lwfw_config | YAML 配置解析 | P1 |

### Phase 5: NSv 虚拟化 (T-090 ~ T-093)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-090 | NSv event_loop | 主事件循环、select/poll | P0 |
| T-091 | packet_mmap | AF-PACKET mmap 实现 | P0 |
| T-092 | ipc-if | VNET_OVER_IPC_SUPPORT | P1 |
| T-093 | VMM/bridge 集成 | VIRT_BRG_SUPPORT | P1 |

### Phase 5: Socket API (T-100 ~ T-105)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-100 | sys_net_socket | socket 创建 | P1 |
| T-101 | sys_net_bind | bind 实现 | P1 |
| T-102 | sys_net_sendto/recvfrom | 数据传输、shm 优化 | P0 |
| T-103 | sys_net_connect | connect 流程 | P1 |
| T-104 | sys_net_accept | accept 队列 | P1 |
| T-105 | sys_net_ctl | netstat/ifconfig/lwfwcfg | P1 |

### Phase 6: 系统集成 (T-110 ~ T-114)

| 任务 ID | 模块 | 描述 | 优先级 |
|---------|------|------|--------|
| T-110 | tcpip_thread | 协议栈线程、mbox 机制 | P0 |
| T-111 | tcpip_inpkt | 带 netif 参数的 input | P1 |
| T-112 | pbuf 管理层 | pbuf 结构、pool/heap | P0 |
| T-113 | lwip_malloc | 内存池初始化 | P1 |
| T-114 | 网络初始化流程 | lwip_init → tcpip_init → netif_add | P0 |

---

## 优先级说明

| 优先级 | 含义 | 任务数量 |
|--------|------|----------|
| **P0** | 必须深度分析，核心路径 | ~25 个 |
| **P1** | 重要，分析主要逻辑 | ~25 个 |
| **P2** | 可选，性能相关 | ~15 个 |

---

## 分析方法

每模块分析模板:
1. 概述 — 功能定位、在架构中的位置、关键数据结构
2. 源码分析 — 函数签名、函数体、调用关系图
3. 数据结构 — 结构体字段详解、初始化流程
4. 边界条件 — 错误处理、资源清理、竞态条件
5. 性能特征 — 时间/空间复杂度、锁/无锁分析
6. 与其他模块的关系 — 上游调用者、下游被调用者、Hook 点
7. 配置方式 — 编译选项、运行时配置

---

## 相关页面

- [[sources/safeos-lwip-analysis-summary]] — 文档汇总
- [[lwip-index]] — lwIP 模块索引
- [[lwfw-index]] — LWFW 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引
