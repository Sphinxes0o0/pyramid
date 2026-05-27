---
type: index
tags: [safeos, nsv, lwip, network, embedded, seL4]
created: 2026-05-25
sources: [safeos-architecture]
---

# SafeOS — Module Index

> SafeOS NSv 网络栈架构与设计文档 (Batch G: 7 篇文档)

## Entity Pages

### NSv 网络架构
| Entity | Description |
|--------|-------------|
| [[entities/linux/safeos/safeos-nsv]] | NSv Network Server 深度分析：初始化序列、线程模型、CMA 内存架构、socket syscall、收包/发包路径、AF-PACKET 实现 |
| [[entities/linux/safeos/safeos-network-implementation]] | SafeOS 网络实现深度分析：CMA+DS-RING、elem_ring、收包/发包路径、协议支持、与 Linux 的本质差异 |
| [[entities/linux/safeos/safeos-packet-mmap]] | AF-PACKET + TPACKET 抓包实现：DSPACE 布局、TPACKET 帧格式、代码核心路径、与 Linux PACKET_MMAP 的核心区别、当前限制 |
| [[entities/linux/safeos/safeos-vdf-nids-relation]] | SafeOS 与 VDF nids 项目关系：仓库结构、适配条件、架构差异、nids 适配可行性 |
| [[entities/linux/safeos/safeos-abi-boundary]] | ABI 边界与内部头文件暴露：packet_mmap_info 暴露问题、public/private API 分离、稳定 ABI 层设计 |
| [[entities/linux/safeos/safeos-lwip-lwfw-plan]] | lwIP + LWFW 深度分析计划：9阶段 ~64任务、架构总览、模块分解、分析方法 |

## New Entity Pages (Batch G2: 2026-05-27)

| Entity | Description |
|--------|-------------|
| [[entities/linux/safeos/safeos-lwip-vlan]] | lwIP IEEE 802.1Q VLAN 实现：struct eth_vlan_hdr、netif->vlanid、PCP 优先级、LWIP_HOOK_VLAN_SET/CHECK |
| [[entities/linux/safeos/safeos-lwip-sel4-performance-boundary]] | seL4 + lwIP 性能边界：IPC 开销 ~150-710ns/packet、tcpip_thread 单线程瓶颈、4 核仅利用 ~25% |
| [[entities/linux/lwip/lwip-cma-elem-ring]] | CMA Buffer (96MB) 与 elem_ring 无锁队列：VA/PA 转换、DMA 共享、dmb(ish) 内存屏障 |
| [[entities/linux/safeos/safeos-lwfw-hotswap]] | LWFW 热切换：双缓冲架构、深拷贝持锁时间、P0 热重载窗口期无防护问题 |

## Source Pages

### Architecture
- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents (7 篇汇总)
- [[sources/safeos-plan]] — SafeOS lwIP + LWFW 深度分析计划 (9 阶段 ~64 任务)

### lwIP Core
- [[sources/safeos-lwip-analysis-summary]] — lwIP 深度分析文档汇总 (~28 Core + 19 Extensions)
- [[sources/safeos-lwip-vlan-implementation]] — lwIP VLAN IEEE 802.1Q 实现
- [[sources/safeos-lwip-vlan-dispatch]] — lwIP vs Linux VLAN 分发机制对比
- [[sources/safeos-lwip-firewall-analysis]] — lwIP 防火墙实现 (lwfw/lwct/cBPF 三层)

## Related Indexes

- [[lwip-index]] — lwIP 模块完整索引 (TCP/UDP/IP/netif/pbuf/VLAN)
- [[lwfw-index]] — LWFW 防火墙模块索引 (架构/过滤/LWCT/解析/IPC/Agent/VLAN/优化)
- [[vdf-index]] — VDF 模块索引 (evm-report/iot-gateway/oam-service/traffic-manager/data-collection)
