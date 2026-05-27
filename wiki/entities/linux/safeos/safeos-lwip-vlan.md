---
type: entity
tags: [safeos, lwip, vlan, 802.1q, netif, qos]
created: 2026-05-27
sources: [safeos-lwip-vlan-implementation, safeos-lwip-vlan-dispatch]
---

# SafeOS lwIP VLAN 实现

## 定义

IEEE 802.1Q VLAN 在 SafeOS lwIP 中的完整实现，包括 VLAN Tag 解析/插入、per-netif VLAN ID、LWIP_HOOK_VLAN_SET/CHECK hooks、多 VLAN netif 支持。

## 核心机制

### RX — VLAN Tag 解析

```
ethernet_input()
  ├─ ETHTYPE_VLAN (0x8100) 判断
  ├─ lwip_hook_vlan_check_fn() — VID 匹配检查
  ├─ p->priority = PCP (提取到 pbuf)
  └─ type = vlan->tpid (还原真实 EtherType)
```

### TX — VLAN Tag 插入

```
ethernet_output()
  └─ LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type)
        ├─ netif->vlanid == NO_VLANID → return -1 (不插入)
        └─ return (pbuf->priority << 13) | netif->vlanid
```

## 关键数据结构

| 结构 | 位置 | 说明 |
|------|------|------|
| `struct eth_vlan_hdr` | `lwip/prot/ethernet.h:121` | TPID + TCI (PCP+VID+DEI) |
| `netif->vlanid` | `lwip/netif.h:441` | 本 netif 的 VLAN ID |
| `pbuf->priority` | `lwip/pbuf.h` | 0-7，对应 802.1Q PCP |

## VLAN 分发 vs Linux

| 方面 | SafeOS lwIP | Linux |
|------|-------------|-------|
| **分发机制** | `LWIP_ARP_FILTER_NETIF` + IP 地址匹配 | `netdev_rx_handler` 精确 VID 匹配 |
| **VLAN netif 本质** | 独立 netif，独立 IP | 虚拟 net_device 堆叠 |
| **AF_PACKET 绑定** | 无法绑定到 VLAN netif | 正常工作 |

## 关键文件

- `ethernet.c` — VLAN tag 解析/插入、hooks
- `vlanif.c` — VLAN netif 创建/初始化
- `conf_parser.c` — YAML 配置解析

## 相关概念

- [[entities/linux/lwip/lwip-sel4-performance-boundary]] — seL4 + lwIP 性能边界
- [[entities/linux/safeos/safeos-nsv]] — NSv 网络服务器
- [[lwip-index]] — lwIP 模块索引

## 来源详情

- [[sources/safeos-lwip-vlan-implementation]] — VLAN 实现深度分析
- [[sources/safeos-lwip-vlan-dispatch]] — lwIP vs Linux VLAN 分发对比
