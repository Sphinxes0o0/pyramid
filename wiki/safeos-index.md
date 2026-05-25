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

## Source Pages

- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents (7 篇汇总)

## Related Indexes

- [[lwip-index]] — lwIP 模块完整索引 (TCP/UDP/IP/netif/pbuf/VLAN)
- [[lwfw-index]] — LWFW 防火墙模块索引 (架构/过滤/LWCT/解析/IPC/Agent/VLAN/优化)
