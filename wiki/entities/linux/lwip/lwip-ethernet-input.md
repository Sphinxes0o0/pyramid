---
type: entity
tags: [linux, lwip, network, ethernet, vlan]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP ethernet_input Analysis

## 定义

`ethernet_input()` 是 lwIP 的 **L2→L3 分发入口**，负责解析 Ethernet Header、解析 VLAN Tag、选择正确的 netif、分发到上层协议。

## 调用链

```
rx_callback()
    │
    ▼
vnet_if.input(p, &vnet_if) = ethernet_input
    │
    ├─► 解析 ETH Header
    ├─► LWIP_ARP_FILTER_NETIF (VLAN-aware netif 选择)
    ├─► ETHARP_SUPPORT_VLAN (VLAN tag 解析)
    ├─► MAC_VLAN_FILTER (可选的安全检查)
    ├─► raw_afpacket_input (AF-PACKET 捕获)
    └─► switch(type) → ip4_input / etharp_input
```

## VLAN 分发核心 — lwip_arp_filter_netif_fn

```c
struct netif *lwip_arp_filter_netif_fn(void *_p, void *_netifIn, u16_t type) {
    switch (type) {
        case ETHTYPE_VLAN: {
            // 通过 VLAN ID 匹配找到 vlan_if[i]
            NETIF_FOREACH(netif) {
                u16_t vid = netif->vlanid & VLAN_ID_MASK;
                if (vid == (vlan_hdr->prio_vid & VLAN_ID_MASK)) {
                    return netif;  // 找到匹配的 VLAN netif!
                }
            }
            return NULL;  // 无匹配，丢弃
        }
        case ETHTYPE_IP: {
            // 通过 IP 地址 + vlanid==0 匹配
            NETIF_FOREACH(netif) {
                if (ip4_addr_cmp(&dst, &netif->ip_addr) &&
                    netif->vlanid == 0u) {
                    return netif;
                }
            }
            break;
        }
    }
    return netifIn;
}
```

## 分发决策矩阵

| Packet 类型 | 分发条件 | 返回的 netif |
|------------|----------|--------------|
| **VLAN-tagged** (VID=100) | `netif->vlanid == 100` | `vlan_if[100]` |
| **VLAN-tagged** (VID=200) | `netif->vlanid == 200` | `vlan_if[200]` |
| **非 VLAN** (IP=172.20.0.1) | `netif->ip_addr == dst && vlanid == 0` | `vnet_if` |

## VLAN Tag 解析 (IEEE 802.1Q)

```
Ethernet Header (14 bytes) + VLAN Tag (4 bytes)
┌─────────────────────────────────────┬──────────────────────────────┐
│ DST (6B) │ SRC (6B) │ Type=0x8100 │ PCP (3b) │ DEI (1b) │ VID (12b) │
└─────────────────────────────────────┴──────────────────────────────┘
```

## 关键设计点

1. **VLAN-aware netif 选择**: `LWIP_ARP_FILTER_NETIF` 在 VLAN 解析之前选择正确的 netif
2. **VLAN Tag 剥离**: `type = vlan->tpid` 移除 VLAN tag，暴露真正的 EtherType
3. **PCP 传递**: `p->priority = vlan->prio_vid >> 13` 将 802.1Q PCP 传递到 pbuf

## 相关概念
- [[entities/linux/safeos/safeos-lwip-lwfw-plan]]
- [[entities/linux/safeos/safeos-nsv]]
- [[entities/linux/lwfw/lwfw-vlan-interception-flow]]

- [[entities/linux/lwip/lwip-netif]] — netif 结构和链表管理
- [[entities/linux/lwip/lwip-ethernet-output]] — 对应的 L3→L2 封装
- [[entities/linux/lwip/lwip-vlan-dispatch]] — lwIP vs Linux VLAN 分发对比
- [[entities/linux/lwip/lwip-vlan-hook]] — LWIP_HOOK_VLAN_CHECK hook
- [[entities/linux/lwip/lwip-ip4-input]] — 分发到 L3 的入口
