---
type: entity
tags: [linux, lwip, network, vlan]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP VLAN Hook Analysis

## 定义

lwIP 提供两个 VLAN 相关 hooks：
- **RX Hook**: `LWIP_HOOK_VLAN_CHECK` / `lwip_hook_vlan_check_fn` — RX 时检查 VLAN ID 是否匹配
- **TX Hook**: `LWIP_HOOK_VLAN_SET` / `lwip_hook_vlan_set_fn` — TX 时决定是否插入 VLAN tag

## RX Hook: lwip_hook_vlan_check_fn

### 调用位置

```c
#ifdef MAC_VLAN_FILTER
#ifdef LWIP_HOOK_VLAN_CHECK
if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
    pbuf_free(p);
    return ERR_OK;  // 不是本 VLAN，silently ignore
}
#endif
#endif
```

### 函数实现

```c
int lwip_hook_vlan_check_fn(void *netif, void *eth_hdr, void *vlan_hdr) {
    struct netif *_netif = (struct netif *)netif;
    struct eth_vlan_hdr *_vlan_hdr = (struct eth_vlan_hdr *)vlan_hdr;

    if (netif_is_up(_netif)) {
        // 提取 netif 的 VLAN ID
        u16_t vid = PP_HTONS(_netif->vlanid) & PP_HTONS(VLAN_ID_MASK);
        // 提取 packet 的 VLAN ID
        u16_t pkt_vid = _vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK);

        if (vid == pkt_vid) {
            return 1;  // 匹配
        }
    }
    return 0;  // 不匹配
}
```

## TX Hook: lwip_hook_vlan_set_fn

### 调用位置

```c
#if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
s32_t vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
if (vlan_prio_vid >= 0) {
    pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);
    vlan->tpid = eth_type_be;
    vlan->prio_vid = lwip_htons(vlan_prio_vid);
    eth_type_be = PP_HTONS(ETHTYPE_VLAN);
}
#endif
```

### 函数实现

```c
int lwip_hook_vlan_set_fn(void *netif, void *pbuf, const void *src, const void *dst, u16_t eth_type) {
    struct netif *_netif = (struct netif *)netif;
    struct pbuf *_pbuf = (struct pbuf *)pbuf;

    if (_netif->vlanid == NO_VLANID) {
        return -1;  // 不插入 VLAN tag
    }

    // 组合 VLAN ID 和 PCP
    u16_t vlan_id = _netif->vlanid;
    vlan_id = ((_pbuf->priority << 13) | vlan_id);

    return vlan_id;
}
```

### 返回值含义

| 返回值 | 含义 |
|--------|------|
| **-1** | 不插入 VLAN tag |
| **>= 0** | 插入 VLAN tag，值为 (PCP << 13) \| VID |

## pbuf->priority 与 PCP

### PCP (Priority Code Point)

| PCP 值 | 优先级 | 用途 |
|--------|--------|------|
| 0 | Best Effort | 普通流量 |
| 1 | Background | 低优先级 |
| 4 | Video | < 100ms 延迟 |
| 5 | Voice | < 10ms 延迟 |
| 6 | Internetwork Control | |
| 7 | Network Control | 最高优先级 |

### PCP 提取/设置

```c
// RX 方向 (ethernet_input)
p->priority = PP_HTONS(vlan->prio_vid) >> 13;  // 提取 PCP

// TX 方向 (lwip_hook_vlan_set_fn)
vlan_id = (pbuf->priority << 13) | netif->vlanid;  // 组合 PCP + VID
```

## SafeOS 配置

| 网口类型 | vlanid | lwip_hook_vlan_set_fn 返回 | 行为 |
|----------|--------|---------------------------|------|
| 物理网口 | 0 (NO_VLANID) | -1 | 不插入 VLAN tag |
| VLAN 网口 | 100 | (pbuf->priority << 13) \| 100 | 插入 VLAN tag |

## 相关概念

- [[entities/linux/lwip/lwip-vlan-parsing]] — VLAN Tag 结构 TPID/TCI/VID/PCP
- [[entities/linux/lwip/lwip-vlan-implementation]] — IEEE 802.1Q 实现、VLAN netif 创建
- [[entities/linux/lwip/lwip-ethernet-input]] — RX Hook 调用位置
- [[entities/linux/lwip/lwip-ethernet-output]] — TX Hook 调用位置
