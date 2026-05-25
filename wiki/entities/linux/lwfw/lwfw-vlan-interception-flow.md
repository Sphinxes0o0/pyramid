---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW VLAN Interception Flow

## 定义

LWFW 在 **Ingress 路径** (ip4_input) 支持 VLAN Tag 解析与匹配，通过 `LWFW_ADVANCED_FUNC_L2` 编译选项启用 IEEE 802.1Q VLAN ID 过滤。

## 完整拦截流程

```
数据包进入 ip4_input()
         │
         ▼
┌─────────────────────────────────────┐
│  ip4.c:743-759                     │
│  lwfw_p->ops->ingress_filter(p, inp)│
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:802                         │
│  ip4_filter_dispatch_incoming()     │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:361-366                     │
│  lwfw_pkt_l2_info_constructor()    │
│  - 从 Ethernet 头提取 VLAN Tag       │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:383-437                     │
│  check_lwfw_l2_info()               │
│  - 匹配 rule->l2.vlan              │
└─────────────────────────────────────┘
```

## VLAN 提取 (L2 解析)

```c
// lwfw.c:232-259
inline static void lwfw_pkt_l2_info_constructor(...) {
  if (dir == LWFW_IN_TABLE) {
#if ETHARP_SUPPORT_VLAN
    struct eth_vlan_hdr *eth_vlan_hdr = NULL;
    if (eth_hdr->type == PP_HTONS(ETHTYPE_VLAN)) {
      eth_vlan_hdr = (struct eth_vlan_hdr *)((void*)eth_hdr + ETHER_HDR_LEN);
      l2->vlan = VLAN_ID(eth_vlan_hdr);  // 提取 VLAN ID
    } else {
      l2->vlan = 0;
    }
#endif
  }
}
```

## VLAN 匹配

```c
// lwfw.c:392
if ((rule->flags & LWFW_RULE_FLAGS_VLAN) &&
    rule_l2_info->vlan != packet_info->vlan)
  goto out;  // 不匹配，跳出
```

## 编译选项要求

```c
#define LWFW_ADVANCED_FUNC_L2 1  // 启用 L2 (VLAN/MAC) 过滤
#define NIO_LWIP_LWFW 1           // 启用 lwfw 功能
```

## 限制说明

| 限制项 | 说明 |
|--------|------|
| 仅 Ingress 方向 | Egress 方向无法使用 VLAN 条件过滤 |
| VLAN 精确匹配 | 只能匹配单个 VLAN ID，不支持范围 |
| 需编译选项 | 必须启用 `LWFW_ADVANCED_FUNC_L2` |

## 相关概念

- [[entities/linux/lwfw/lwfw-vlan-isolation-guide]] — VLAN 隔离配置指南
- [[entities/linux/lwfw/lwfw-filter-flow]] — 完整过滤流程
- [[entities/linux/lwfw/lwfw-core-filtering]] — L2 匹配逻辑
- [[entities/linux/lwip/lwip-vlan-parsing]] — lwIP VLAN tag 解析
