---
type: entity
tags: [linux, lwip, network, vlan]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP VLAN Implementation Analysis

## 定义

SafeOS 中 lwIP 的 VLAN 支持基于 **IEEE 802.1Q** 标准，实现了两层功能：lwIP 核心 (802.1Q VLAN Tag 解析/插入、VLAN ID 检查) 和 VLAN 接口层 (虚拟 VLAN netif 创建、配置、路由)。

## IEEE 802.1Q VLAN 数据结构

### VLAN Tag 结构

```
Ethernet Header (14 bytes)
┌─────────────────────────────────────┬──────────────────────────────┐
│ Destination MAC (6B)               │ Source MAC (6B)              │
├─────────────────────────────────────┴──────────────────────────────┤
│ TPID (2B) = 0x8100                   │ TCI (2B)                    │
├─────────────────────────────────────┼──────────────────────────────┤
│ PCP (3 bits) │ DEI (1 bit) │ VID (12 bits)                       │
└─────────────────────────────────────┴──────────────────────────────┘
```

### lwIP VLAN 数据结构

```c
struct eth_vlan_hdr {
    u16_t prio_vid;  // Bit[15:13] = PCP, Bit[12] = DEI, Bit[11:0] = VID
    u16_t tpid;       // Tag Protocol Identifier (= 0x8100 for 802.1Q)
};

#define SIZEOF_VLAN_HDR  4
#define VLAN_ID(vlan_hdr) (lwip_htons((vlan_hdr)->prio_vid) & 0xFFF)
```

### netif->vlanid 字段

```c
struct netif {
    // ... 其他字段 ...
#if ETHARP_SUPPORT_VLAN
    u16_t vlanid;    // 本 netif 所属的 VLAN ID (0 = 不属于任何 VLAN)
#endif
    // ... 其他字段 ...
};
```

## VLAN 接口层 (vlanif.c)

### vlanif_init — VLAN netif 初始化

```c
err_t vlanif_init(struct netif *netif) {
    int conf_idx = *((int *)netif->state);

    netif->name[0] = vlan_conf[conf_idx].ifName[0];
    netif->name[1] = vlan_conf[conf_idx].ifName[1];

    netif->output = etharp_output;

    // 关键: linkoutput 调用物理网口的 linkoutput
    netif->linkoutput = low_level_output;

    // 设置 VLAN ID
    netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10);

    netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP |
                   NETIF_FLAG_LINK_UP | NETIF_FLAG_IGMP | NETIF_FLAG_ETHERNET;
    return ERR_OK;
}
```

### low_level_output — VLAN → 物理网卡

```c
static err_t low_level_output(struct netif *netif, struct pbuf *p) {
    if (physical_netif != NULL) {
        // 所有 VLAN 包的输出都经过物理网卡
        // VLAN Tag 的插入由 lwip_hook_vlan_set_fn() 完成
        physical_netif->linkoutput(physical_netif, p);
    }
    return ERR_OK;
}
```

## VLAN 与物理网卡的对应关系

```
┌─────────────────────────────────────────────────────────────┐
│  vnet_if (物理网卡, netif->vlanid = 0)                     │
│  - name = "ei"                                             │
│  - linkoutput = ethif_link_output                           │
│  - 处理不带 VLAN Tag 的以太网帧                              │
└─────────────────────────────────────────────────────────────┘
                              ↑
                              │ VLAN 包 (已插入 VLAN Tag)
                              │
┌─────────────────────────────────────────────────────────────┐
│  vlan_if[0] (VLAN 网卡, netif->vlanid = 100)              │
│  - name = "vl0"                                            │
│  - linkoutput = low_level_output                           │
│  - 所有包输出到 physical_netif (vnet_if)                   │
└─────────────────────────────────────────────────────────────┘
```

## 支持的功能和限制

| 功能 | 状态 | 说明 |
|------|------|------|
| VLAN Tag 解析 (RX) | ✅ | 支持 ETHTYPE_VLAN (0x8100) |
| VLAN Tag 插入 (TX) | ✅ | 通过 `LWIP_HOOK_VLAN_SET` |
| per-netif VLAN ID | ✅ | `netif->vlanid` 字段 |
| VLAN ID 过滤 (RX) | ✅ | `lwip_hook_vlan_check_fn()` |
| 802.1Q PCP 优先级 | ✅ | `pbuf->priority` ↔ VLAN TCI |
| 多 VLAN netif | ✅ | 物理网卡 + 多个 VLAN 网卡 |
| QinQ (双重 VLAN Tag) | ❌ | 不支持嵌套 VLAN |

## 相关概念

- [[entities/linux/lwip/lwip-vlan-parsing]] — VLAN Tag 结构和 VID/PCP 提取
- [[entities/linux/lwip/lwip-vlan-hook]] — LWIP_HOOK_VLAN_SET 和 LWIP_HOOK_VLAN_CHECK
- [[entities/linux/lwip/lwip-vlan-dispatch]] — VLAN 分发机制
- [[entities/linux/lwip/lwip-netif-add]] — netif_add 注册流程
