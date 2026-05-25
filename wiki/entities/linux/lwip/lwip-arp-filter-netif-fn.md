---
type: entity
tags: [linux, lwip, network, vlan, netif, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP ARP Filter Netif Function

## 定义

`lwip_arp_filter_netif_fn()` 是 SafeOS lwIP 的 **VLAN-aware netif 选择函数**，负责根据 Ethernet Type 和 VID 将 packet 软分发到正确的 netif（vnet_if 或 vlan_if[i]）。

## 调用链

```
rx_callback
    │
    ▼
ethernet_input(p, &vnet_if)
    │
    ▼
LWIP_ARP_FILTER_NETIF(p, netif, type)
    │
    ▼
lwip_arp_filter_netif_fn(p, netif, type)
    │
    ├─► ETHTYPE_ARP → 通过 IP 地址 + vlanid==0 匹配
    ├─► ETHTYPE_IP  → 通过 IP 地址 + vlanid==0 匹配
    └─► ETHTYPE_VLAN → 通过 VLAN ID 匹配
```

## 核心分发逻辑

### ETHTYPE_ARP / ETHTYPE_IP
```c
NETIF_FOREACH(netif) {
    if (netif_is_up(netif) &&
        ip4_addr_cmp(&dst, &netif->ip_addr) &&  // IP 地址匹配
        netif->vlanid == 0u) {                  // ★ 关键: 非 VLAN netif
        return netif;
    }
}
```

### ETHTYPE_VLAN
```c
NETIF_FOREACH(netif) {
    u16_t vid = PP_HTONS(netif->vlanid) & VLAN_ID_MASK;
    if (vid == (vlan_hdr->prio_vid & VLAN_ID_MASK)) {
        return netif;  // VID 精确匹配
    }
}
```

## VLAN 分发决策矩阵

| Packet | EtherType | VID/IP | 分发到 |
|--------|-----------|--------|--------|
| 非VLAN, IP=172.20.0.1 | ETHTYPE_IP | 172.20.0.1 | vnet_if |
| VLAN VID=100 | ETHTYPE_VLAN | VID=100 | vlan_if[0] |
| VLAN VID=200 | ETHTYPE_VLAN | VID=200 | vlan_if[1] |
| VLAN VID=300 | ETHTYPE_VLAN | VID=300 | NULL (丢弃) |

## SafeOS 中的 netif 链表

```
netif_list
      │
      ▼
┌─────────────┐    next     ┌─────────────┐    next     ┌─────────────┐
│   vnet_if   │ ─────────► │  vlan_if[0] │ ─────────► │  vlan_if[1] │
│ (物理网口)   │            │  (VLAN 100) │            │  (VLAN 200) │
└─────────────┘            └─────────────┘            └─────────────┘
   .vlanid = 0                .vlanid = 100              .vlanid = 200
```

## 关键设计

1. **软分发**: 所有 packet 经过 vnet_if，通过函数指针选择真正处理者
2. **vlanid=0 标记**: 物理网口使用 vlanid=0，与 VLAN netif 区分
3. **两阶段分发**: `lwip_arp_filter_netif_fn` 先选 netif，ethernet_input 内部再解析 VLAN Tag

## 相关概念

- [[entities/linux/lwip/lwip-ethernet-input]] — 调用方
- [[entities/linux/lwip/lwip-vlan-dispatch]] — VLAN 分发机制
- [[entities/linux/lwip/lwip-vlan-implementation]] — vlanid 使用方式
- [[entities/linux/lwip/lwip-netif]] — netif 结构

## 来源详情

- [[sources/safeos-lwip-extensions]]
