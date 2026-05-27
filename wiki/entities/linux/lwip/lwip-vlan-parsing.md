---
type: entity
tags: [linux, lwip, network, vlan]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP VLAN Parsing Analysis

## 定义

VLAN 解析是 lwIP 处理 **VLAN-tagged packet** 的关键步骤：识别 VLAN-tagged packet (通过 EtherType = 0x8100)、提取 VLAN Tag 中的信息 (VID, PCP)、更新 pbuf 的相关字段。

## IEEE 802.1Q VLAN Tag 结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Ethernet Header (14 bytes)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  Destination MAC (6B) │   Source MAC (6B)  │  EtherType (2B)              │
└─────────────────────────────────────────────────────────────────────────────┘
                                                              │
                        ┌─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      VLAN Tag (4 bytes)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│   TPID (2B) = 0x8100    │   TCI (2B) = PCP(3b) + DEI(1b) + VID(12b)      │
└─────────────────────────────────────────────────────────────────────────────┘
```

## TCI (Tag Control Information) 结构

```
┌─────────────────────────────────────────────┐
│  PCP (3 bits)  │  DEI (1 bit)  │  VID (12 bits) │
├─────────────────────────────────────────────┤
│  Priority Code Point                       │  VLAN ID          │
│  (0-7, 802.1p priority)                   │  (0-4095)         │
└─────────────────────────────────────────────┘
```

| 字段 | 长度 | 说明 | 用途 |
|------|------|------|------|
| **PCP** | 3 bits | Priority Code Point | 802.1p QoS 优先级 |
| **DEI** | 1 bit | Drop Eligibility Indicator | 帧是否可丢弃 |
| **VID** | 12 bits | VLAN Identifier | 虚拟局域网 ID (0-4095) |

## ethernet_input 中的 VLAN 解析

```c
#if ETHARP_SUPPORT_VLAN
if (type == PP_HTONS(ETHTYPE_VLAN)) {  // 0x8100
    // Step 1: 定位 VLAN Header
    struct eth_vlan_hdr *vlan = (struct eth_vlan_hdr *)(((char *)ethhdr) + SIZEOF_ETH_HDR);
    next_hdr_offset = SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR;  // = 18 bytes

    // Step 2: 长度检查
    if (p->len <= SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR) {
        goto free_and_return;
    }

    // Step 3: VLAN 安全检查 (可选)
    #ifdef MAC_VLAN_FILTER
    if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
        pbuf_free(p);
        return ERR_OK;
    }
    #endif

    // Step 4: 提取真正的 EtherType
    type = vlan->tpid;

    // Step 5: 提取 PCP 到 pbuf->priority
    p->priority = PP_HTONS(vlan->prio_vid) >> 13;  // 高 3 位是 PCP
}
#endif
```

## VLAN Header 结构

```c
struct eth_vlan_hdr {
    u16_t prio_vid;  // Bit[15:13] = PCP, Bit[12] = DEI, Bit[11:0] = VID
    u16_t tpid;       // Tag Protocol Identifier (= 0x8100 for 802.1Q)
};

#define SIZEOF_VLAN_HDR 4

// VID 提取: 取低 12 位
#define VLAN_ID(vlan_hdr) (lwip_htons((vlan_hdr)->prio_vid) & 0xFFF)
```

## PCP (Priority Code Point)

| PCP 值 | 优先级 | 典型用途 |
|--------|--------|----------|
| 0 | Lowest | Background |
| 1 | Low | Best Effort |
| 2 | Medium | Excellent Effort |
| 3 | High | Critical Applications |
| 4 | High | Video (< 100ms latency) |
| 5 | High | Voice (< 100ms latency) |
| 6 | Highest | Internetwork Control |
| 7 | Highest | Network Control |

## VLAN 解析流程图

```
收到 Ethernet Frame
    │
    ▼
┌───────────────────────────────────────────────────────┐
│  ether_header:                                        │
│    - dst_mac (6B)                                     │
│    - src_mac (6B)                                     │
│    - type (2B):                                       │
│        ├─ 0x0800 → IPv4                              │
│        ├─ 0x0806 → ARP                               │
│        └─ 0x8100 → VLAN-tagged                        │
└───────────────────────────────────────────────────────┘
                    │
                    ▼
            type == 0x8100?
                    │
                    ▼
            ┌───────────────────┐
            │  VLAN-tagged      │
            │  next_hdr_offset=18│
            │  type = vlan->tpid │
            │  p->priority = PCP │
            └───────────────────┘
                    │
                    ▼
            switch(type)
                ├─► IPv4 → ip4_input()
                ├─► ARP → etharp_input()
                └─► ...
```

## 相关概念
- [[entities/linux/safeos/safeos-lwip-lwfw-plan]]
- [[entities/linux/lwfw/lwfw-core-filtering]]
- [[entities/linux/lwfw/lwfw-vlan-interception-flow]]

- [[entities/linux/lwip/lwip-ethernet-input]] — ethernet_input 完整流程
- [[entities/linux/lwip/lwip-vlan-hook]] — LWIP_HOOK_VLAN_CHECK 和 LWIP_HOOK_VLAN_SET
- [[entities/linux/lwip/lwip-vlan-implementation]] — IEEE 802.1Q 实现细节
- [[entities/linux/lwip/lwip-pbuf]] — pbuf->priority 字段用途
