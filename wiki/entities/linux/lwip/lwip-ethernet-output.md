---
type: entity
tags: [linux, lwip, network, ethernet, vlan]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP ethernet_output Analysis

## 定义

`ethernet_output()` 是 lwIP 的 **L3→L2 封装出口**，负责添加 Ethernet Header、可选插入 VLAN Tag、AF-PACKET 输出捕获、调用 `netif->linkoutput` 发送到链路。

## 调用链

```
App send()
    │
    ▼
ethernet_output(netif, p, src, dst, ETHTYPE_IP)
    │
    ├─► [VLAN 插入] LWIP_HOOK_VLAN_SET → pbuf_add_header(ETH_HDR + VLAN_HDR)
    ├─► 添加 Ethernet Header (dst MAC, src MAC, Type)
    ├─► raw_afpacket_output()  // AF-PACKET 捕获
    └─► netif->linkoutput(netif, p)
          └─► ethif_link_output() → NIC DMA
```

## VLAN Tag 插入

```c
s32_t vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
if (vlan_prio_vid >= 0) {
    // 需要插入 VLAN Tag
    pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);  // +4 bytes
    vlanhdr->tpid = eth_type_be;      // 保存原始 EtherType
    vlanhdr->prio_vid = lwip_htons((u16_t)vlan_prio_vid);  // PCP + VID
    eth_type_be = PP_HTONS(ETHTYPE_VLAN);  // 改为 VLAN EtherType
}
```

## lwip_hook_vlan_set_fn

```c
int lwip_hook_vlan_set_fn(void *netif, void *pbuf, const void *src, const void *dst, u16_t eth_type) {
    if (_netif->vlanid == NO_VLANID) {
        return -1;  // 不插入 VLAN Tag
    }
    u16_t vlan_id = _netif->vlanid;
    vlan_id = (((struct pbuf*)pbuf)->priority << 13) | vlan_id;  // PCP + VID
    return vlan_id;  // 返回 >= 0，触发 VLAN Tag 插入
}
```

## SafeOS 中的 linkoutput

| netif 类型 | linkoutput | 说明 |
|------------|-----------|------|
| **vnet_if** | `ethif_link_output` | 发送到 NIC DMA |
| **vlan_if[i]** | `low_level_output` | 调用 `physical_netif->linkoutput` |

## 相关概念

- [[entities/linux/lwip/lwip-ethernet-input]] — 对应的 L2→L3 入口
- [[entities/linux/lwip/lwip-netif]] — netif 结构和 linkoutput 回调
- [[entities/linux/lwip/lwip-vlan-hook]] — LWIP_HOOK_VLAN_SET hook 详解
- [[entities/linux/lwip/lwip-pbuf]] — pbuf_add_header 用于添加 VLAN space
