---
type: synthesis
tags: [nids, safeos, lwip, lwfw, architecture]
created: 2026-05-26
---

# NIDS-SafeOS 全栈架构综述

## 模块索引

- [[safeos-index]] — SafeOS 运行时
- [[lwip-index]] — lwIP 网络栈
- [[lwfw-index]] — LWFW 防火墙
- [[nids-master-index]] — NIDS 检测引擎

## 架构全景

```
Hardware (NIC)
  → seL4 Microkernel
    → NIC Driver (AF_PACKET/PFE.VLAN1)
      → NSv (Network Server)
        ├── lwIP (TCP/IP stack)
        │   ├── ethernet_input → ip4_input
        │   └── VLAN dispatch (802.1Q)
        ├── LWFW (Firewall)
        │   ├── Hook injection (pbuf→LWFW)
        │   ├── Rule matching (ACL tree)
        │   └── LWCT (connection tracking)
        └── NIDS (via shared memory)
            ├── Capture pipeline
            ├── Detection engine
            └── Rule engine (Snort3-compatible)
```

## 数据流

**RX 路径**: NIC DMA → CMA Buffer → elem_ring → nic_rx_thread → ethernet_input → ip4_input → LWFW/LWCT → TCP/UDP → Application

**TX 路径**: App → event_loop → lwIP → ip4_output_if → LWFW → ethernet_output → NIC Driver

**NIDS 路径**: LWFW pbuf copy → shared memory (CMA) → NIDS capture → decode → detect → alert

## 性能边界

| 组件 | 延迟 | 吞吐 | 瓶颈 |
|------|------|------|------|
| seL4 IPC | ~2-5μs/call | — | 占延时 63-78% |
| lwIP tcpip_thread | — | 单核 ~1.2x scaling | 单线程模型 |
| LWFW rule matching | ~0.5μs/hook | — | 线性搜索(O(n)) |
| NIDS detection | ~8-15μs/pkt | ~2-5Gbps | Rule count scaling |
| seL4→Linux VM | ~10-20μs | — | VM boundary |

## 防御闭环

**当前**: NIDS 检测 → 告警 → (手动) → LWFW 阻断 (>10ms)

**Phase 1**: NIDS 检测 → shared memory → LWFW 阻 断 (<5ms)

**Phase 2**: NIDS inline → LWFW hook → 实时阻断 (<1ms)

## 瓶颈路线图

| 优先级 | 瓶颈 | 影响 | 缓解 |
|--------|------|------|------|
| P0 | seL4 IPC 频率 | 延迟天花板 | 批量 IPC、共享内存 |
| P1 | tcpip_thread 单线程 | 吞吐 scaling | 多线程 tcpip (lwIP 2.x) |
| P2 | LWFW 线性搜索 | 规则数上限 | Tree/hash 索引 |
| P2 | NIDS rule count | 检测延迟 | Rule 分组、pre-filter |

## 来源

- [[synthesis/safeos-lwip-deep-analysis]]
- [[synthesis/safeos-source-analysis]]
- [[synthesis/nids-current-architecture]]
- [[synthesis/nids-vs-snort3-summary]]
- [[synthesis/nids-gap-analysis-roadmap]]
