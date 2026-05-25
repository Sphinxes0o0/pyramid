---
type: source
source-type: github
created: 2026-05-25
title: "SafeOS Architecture & Design Documents"
date: 2026-04-22
size: medium
path: raw/safeos/
summary: "SafeOS NSv 网络栈架构设计文档集：9阶段分析计划、NSv深度分析、网络实现、AF-PACKET/TPACKET设计、VDF nids关系"
tags: [safeos, nsv, lwip, af-packet, tpacket, dspace, sel4, vdf]
sources: []
---

# SafeOS Architecture & Design Documents

## 核心内容

本批次包含 7 篇 SafeOS 架构与设计文档，涵盖 SafeOS NSv 网络栈的整体架构、初始化序列、内存架构、AF-PACKET/TPACKET 抓包实现、以及与 VDF nids 项目的适配关系。

### 文档组成

| 文档 | 主要内容 |
|------|----------|
| `architecture_notes.md` | SafeOS lwIP + LWFW 深度分析 9 阶段计划 |
| `plan.md` | 同上，完整任务分解表 (T-001~T-114) |
| `NSv_analysis.md` | NSv 深度分析：初始化序列、线程模型、CMA 内存架构、socket syscall、收包/发包路径、AF-PACKET 实现 |
| `network_implementation_analysis.md` | SafeOS 网络实现分析：整体架构、CMA+DS-RING、收包/发包路径、协议支持、与 Linux 的本质差异 |
| `packet_mmap_design.md` | AF-PACKET + TPACKET 抓包设计文档：架构图、DSPACE 布局、TPACKET 帧格式、代码核心路径 |
| `af_packet_mmap_summary.md` | AF-PACKET 实现总结与优化建议：当前架构评估、可优化点、nids 适配可行性 |
| `memory/safeos_vdf_nids.md` | SafeOS 与 VDF nids 项目关系：仓库结构、适配条件、架构差异 |

---

## 关键架构概念

### NSv 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     APP (net-cap / tcpdump)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │ seL4 IPC (sys_net_*)
┌────────────────────────────▼────────────────────────────────────┐
│                     NSv (Network Server — lwIP)                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │App Event    │  │ NIC RX       │  │ lwIP TCPIP Thread    │  │
│  │Loop         │  │ Thread       │  │                      │  │
│  │(socket API) │  │ (RX path)    │  │ TCP/UDP/RAW/IP       │  │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                      │               │
│  ┌──────▼────────────────▼──────────────────────▼───────────┐  │
│  │              vnet_if (struct netif)                      │  │
│  │         netif->output = etharp_output                      │  │
│  │         netif->linkoutput = ethif_link_output            │  │
│  └─────────────────────────────┬──────────────────────────────┘  │
└────────────────────────────────┼───────────────────────────────┘
                                 │ elem_ring (CMA shared memory)
┌────────────────────────────────▼───────────────────────────────┐
│                  NIC Driver (separate process)                  │
└───────────────────────────────────────────────────────────────┘
```

### NSv 初始化序列

```
main()
  ├─ net_resources_init()      // 堆/互斥锁/哈希表
  ├─ init_ds_ring()           // CMA 分配 + ds_ring 创建 + NIC 授权
  ├─ create_nic_thread()      // nic_rx_thread 创建
  ├─ tcpip_init()             // lwIP 核心初始化
  ├─ netif_add(&vnet_if)    // 注册虚拟网卡
  └─ event_loop()            // 主事件循环
```

### CMA + elem_ring 内存架构

- **CMA_SIZE**: 96MB (0x6000000)
- **4 个 elem_ring**: empty_rx_buf_ring, used_rx_buf_ring, pending_tx_buf_ring, used_tx_buf_ring
- **内存屏障**: ARM `dmb(ish)` 保证读写顺序

### AF-PACKET + TPACKET 关键参数

| 常量 | 值 |
|------|-----|
| `DEFAULT_TP_FRAME_SIZE` | 2048 |
| `DEFAULT_TP_FRAME_NR` | 1024 |
| `DEFAULT_TP_BLOCK_SIZE` | 4096 |
| `DEFAULT_DSPACE_SIZE` | 0x400000 (4MB) |

### 与 Linux 的本质差异

| 方面 | Linux | SafeOS |
|------|-------|--------|
| 网络栈位置 | 内核态 | 用户态 (NSv进程) |
| NIC驱动 | 内核模块 | 独立用户态进程 |
| 内存共享 | mmap + 页表 | DSPACE + CMA |
| socket实现 | 内核文件描述符 | lwIP netconn → 用户态模拟 |
| 零拷贝 | sendfile/AF_XDP | NIC→CMA→pbuf→App，两次拷贝 |
| 通知机制 | 内核poll() | lwIP API_EVENT() → select() |

### AF-PACKET 实现评估

| 方面 | 评价 |
|------|------|
| 架构合理性 | ⚠️ 中等 — 自定义 DSPACE 方案可行，但架构边界不清晰 |
| API 稳定性 | ❌ 差 — 内部结构暴露给 App，无稳定 ABI 层 |
| 代码复用 | ❌ 差 — packet_mmap.c 代码重复 |
| 功能完整性 | ⚠️ 基本可用 — TPACKET_V1 RX Ring 可工作，但缺高级特性 |
| 可维护性 | ⚠️ 中等 — 文档分散，缺少测试 |
| 对外适配 | ❌ 差 — nids 等外部 App 无法直接使用 |

---

## 关键引用

### NSv 核心文件

- `os-framework/servers/net/src/main.c` — NSv 主入口、event_loop、socket 处理、TX/RX 路径
- `os-framework/servers/net/src/packet_mmap.c` — AF-PACKET TPACKET ring 设置、tpacket_recv 回调
- `os-framework/servers/net/include/nsv/nsv.h` — NSv 常量、宏定义
- `os-framework/servers/net/include/nsv/packet_mmap.h` — packet_mmap_info 结构体、常量

### 内存基础设施

- `libs/os_libs/libcore/src/ds_ring.c` — DS-RING 实现
- `libs/os_libs/libcore/src/ringbuffer.c` — ringbuf 读写实现
- `libs/os_libs/libcore/include/core/elem_ring.h` — elem_ring 定义

### lwIP 集成

- `external/lwip_ds_mcu/src/api/sockets.c` — BSD socket → lwIP 映射
- `external/lwip_ds_mcu/src/netif/ethernet.c` — ethernet_input/output
- `external/lwip_ds_mcu/src/core/raw.c` — raw_afpacket_input/output
- `libs/util_libs/liblwip/src/sys_arch_sel4.c` — seL4 sys_arch 适配层

---

## 相关页面

### NSv 网络架构
- [[entities/linux/safeos/safeos-nsv]] — NSv 深度分析
- [[entities/linux/safeos/safeos-network-implementation]] — SafeOS 网络实现
- [[entities/linux/safeos/safeos-packet-mmap]] — AF-PACKET + TPACKET 抓包实现
- [[entities/linux/safeos/safeos-vdf-nids-relation]] — SafeOS 与 VDF nids 关系

### lwIP 相关
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 内存管理
- [[entities/linux/lwip/lwip-raw-socket]] — RAW socket / AF-PACKET 绑定
- [[entities/linux/lwip/lwip-packet-mmap]] — lwIP packet_mmap 实现
- [[entities/linux/lwip/lwip-nsv-event-loop]] — NSv 事件循环
- [[entities/linux/lwip/lwip-sel4-function]] — seL4 上 lwIP 函数调用链
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 通信机制

### LWFW 防火墙
- [[entities/linux/lwip/lwip-lwfw-filter-hooks]] — LWFW filter hooks 集成点
- [[entities/linux/lwip/lwip-firewall]] — LWFW 无状态包过滤

### 模块索引
- [[lwip-index]] — lwIP 模块完整索引
- [[lwfw-index]] — LWFW 防火墙模块索引
