---
type: source
source-type: github
created: 2026-05-27
title: "SafeOS lwIP VLAN 实现深度分析"
date: 2026-04-22
size: medium
path: raw/safeos/docs/lwip_vlan_implementation.md
summary: "IEEE 802.1Q VLAN在lwIP中的完整实现：struct eth_vlan_hdr、netif->vlanid、PCP优先级、LWIP_HOOK_VLAN_SET/CHECK、vlanif_init、多VLAN支持、YAML配置"
tags: [safeos, lwip, vlan, 802.1q, pcp, qos, netif, yaml]
sources: []
---

# SafeOS lwIP VLAN 实现深度分析

> 文档版本: 1.0 | 更新日期: 2026/04/22

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
// struct eth_vlan_hdr (lwip/prot/ethernet.h:121)
struct eth_vlan_hdr {
    u16_t prio_vid;   // Bit[15:13] = PCP, Bit[12] = DEI, Bit[11:0] = VID
    u16_t tpid;       // = 0x8100 for 802.1Q
};

// netif->vlanid (lwip/netif.h:441)
#if ETHARP_SUPPORT_VLAN
    u16_t vlanid;    // 本 netif 所属的 VLAN ID (0 = 不属于任何 VLAN)
#endif
```

### PCP 优先级映射

| PCP 值 | 优先级 | 典型用途 |
|--------|--------|----------|
| 0 | Best Effort | 普通数据 |
| 4 | Video | < 100ms 延迟 |
| 5 | Voice | < 10ms 延迟 |
| 6 | Internetwork Control | |
| 7 | Network Control | 最高优先级 |

---

## RX 路径 — VLAN Tag 解析

```c
// ethernet_input() (ethernet.c:133-169)
if (type == ETHTYPE_VLAN (0x8100)) {
    vlan = (struct eth_vlan_hdr *)(p->payload + SIZEOF_ETH_HDR);
    #ifdef MAC_VLAN_FILTER
        LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)
        // → lwip_hook_vlan_check_fn() 检查 VID 匹配
    #endif
    type = vlan->tpid;  // 还原真实 EtherType
    p->priority = (vlan->prio_vid >> 13);  // 提取 PCP
}
```

---

## TX 路径 — VLAN Tag 插入

```c
// ethernet_output() (ethernet.c:339-355)
#if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
    vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
    if (vlan_prio_vid >= 0) {
        pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);
        vlanhdr->tpid = eth_type_be;  // 保存原始 EtherType
        vlanhdr->prio_vid = lwip_htons(vlan_prio_vid);
        eth_type_be = PP_HTONS(ETHTYPE_VLAN);
    }
#endif

// lwip_hook_vlan_set_fn() (ethernet.c:423-441)
if (_netif->vlanid == NO_VLANID) return -1;  // 不插入 VLAN
return (pbuf->priority << 13) | netif->vlanid;  // PCP + VID
```

---

## VLAN 接口层 (vlanif.c)

### vlanif_init

```c
err_t vlanif_init(struct netif *netif)
{
    netif->output = etharp_output;
    netif->linkoutput = low_level_output;  // 指向物理网卡的 low_level_output
    netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10);
    netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP | ...;
}
```

### low_level_output

```c
static err_t low_level_output(struct netif *netif, struct pbuf *p) {
    if (physical_netif != NULL) {
        // 所有 VLAN 包都经过物理网卡
        // VLAN Tag 插入由 lwip_hook_vlan_set_fn() 完成
        physical_netif->linkoutput(physical_netif, p);
    }
}
```

---

## YAML 配置

```yaml
network:
  vlan:
    - ifName: "vl0"
      vid: "100"
      lowerIfs: "PFE"
      ipAddr: "192.168.100.1"
      netMask: "255.255.255.0"
  arp_table:
    - ifName: "vl0"
      addr: "192.168.100.10"
      mac: "00:11:22:33:44:55"
```

---

## 功能支持矩阵

| 功能 | 状态 | 说明 |
|------|------|------|
| VLAN Tag 解析 (RX) | ✅ | ETHTYPE_VLAN (0x8100) |
| VLAN Tag 插入 (TX) | ✅ | `LWIP_HOOK_VLAN_SET` |
| per-netif VLAN ID | ✅ | `netif->vlanid` |
| VLAN ID 过滤 (RX) | ✅ | `lwip_hook_vlan_check_fn()` |
| 802.1Q PCP 优先级 | ✅ | `pbuf->priority` ↔ VLAN TCI |
| 多 VLAN netif | ✅ | 物理网卡 + 多个 VLAN 网卡 |
| VLAN 静态 ARP 表 | ✅ | `etharp_add_static_entry()` |
| QinQ (双重 VLAN Tag) | ❌ | 不支持 |
| IPv6 VLAN | ❌ | `LWIP_IPV6 = 0` |

---

## 关键文件清单

| 文件 | 职责 |
|------|------|
| `ethernet.c` | VLAN Tag 解析/插入、hooks |
| `etharp.c` | VLAN multicast 输出 |
| `vlanif.c` | VLAN netif 创建/初始化 |
| `conf_parser.c` | YAML 配置解析 |
| `qos.c` | PFE VLAN Bridge、QoS 优先级 |

---

## 相关页面

- [[sources/safeos-lwip-vlan-dispatch]] — lwIP vs Linux VLAN 分发对比
- [[lwip-index]] — lwIP 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引
