# VLAN 解析分析 — T-060

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: IEEE 802.1Q VLAN Tag 解析、TPID/TCI 结构、VID/PCP 提取

---

## 1. 概述

VLAN (Virtual Local Area Network) 解析是 lwIP 处理 **VLAN-tagged packet** 的关键步骤：

1. 识别 VLAN-tagged packet (通过 EtherType = 0x8100)
2. 提取 VLAN Tag 中的信息 (VID, PCP)
3. 更新 pbuf 的相关字段

### 1.1 VLAN Tag 结构 (IEEE 802.1Q)

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
│   TPID (2B) = 0x8100    │   TCI (2B) = PCP(3b) + DEI(1b) + VID(12b)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 函数源码分析

### 2.1 ethernet_input 中的 VLAN 解析

**文件**: `external/lwip_ds_mcu/src/netif/ethernet.c:133-168`

```c
#if ETHARP_SUPPORT_VLAN
if (type == PP_HTONS(ETHTYPE_VLAN)) {  // 0x8100
    // ============================================
    // Step 1: 定位 VLAN Header
    // ============================================
    struct eth_vlan_hdr *vlan = (struct eth_vlan_hdr *)(((char *)ethhdr) + SIZEOF_ETH_HDR);
    next_hdr_offset = SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR;  // = 18 bytes

    // ============================================
    // Step 2: 长度检查
    // ============================================
    if (p->len <= SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR) {
        goto free_and_return;  // packet 太短
    }

    // ============================================
    // Step 3: VLAN 安全检查 (可选)
    // ============================================
    #ifdef MAC_VLAN_FILTER
    #ifdef LWIP_HOOK_VLAN_CHECK
    if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
        pbuf_free(p);
        return ERR_OK;  // 不是本 VLAN，silently ignore
    }
    #elif defined(ETHARP_VLAN_CHECK_FN)
    if (!ETHARP_VLAN_CHECK_FN(ethhdr, vlan)) {
        pbuf_free(p);
        return ERR_OK;
    }
    #elif defined(ETHARP_VLAN_CHECK)
    if (VLAN_ID(vlan) != ETHARP_VLAN_CHECK) {
        pbuf_free(p);
        return ERR_OK;
    }
    #endif
    #endif

    // ============================================
    // Step 4: 提取真正的 EtherType
    // ============================================
    type = vlan->tpid;  // 从 VLAN Header 提取原始 EtherType

    // ============================================
    // Step 5: 提取 PCP 到 pbuf->priority
    // ============================================
    p->priority = PP_HTONS(vlan->prio_vid) >> 13;  // 高 3 位是 PCP
}
#endif
```

---

## 3. VLAN Header 结构

**文件**: `external/lwip_ds_mcu/src/include/lwip/prot/ethernet.h:121-134`

```c
struct eth_vlan_hdr {
    u16_t prio_vid;  // Bit[15:13] = PCP, Bit[12] = DEI, Bit[11:0] = VID
    u16_t tpid;      // Tag Protocol Identifier (= 0x8100)
};

#define SIZEOF_VLAN_HDR 4

// VID 提取: 取低 12 位
#define VLAN_ID(vlan_hdr) (lwip_htons((vlan_hdr)->prio_vid) & 0xFFF)
```

### 3.1 TCI (Tag Control Information) 结构

```
┌─────────────────────────────────────────────┐
│  PCP (3 bits)  │  DEI (1 bit)  │  VID (12 bits) │
├─────────────────────────────────────────────┤
│  Priority Code Point                       │  VLAN ID          │
│  (0-7, 802.1p priority)                   │  (0-4095)        │
└─────────────────────────────────────────────┘
```

| 字段 | 长度 | 说明 | 用途 |
|------|------|------|------|
| **PCP** | 3 bits | Priority Code Point | 802.1p QoS 优先级 |
| **DEI** | 1 bit | Drop Eligibility Indicator | 帧是否可丢弃 |
| **VID** | 12 bits | VLAN Identifier | 虚拟局域网 ID (0-4095) |

---

## 4. 解析流程图

```
收到 Ethernet Frame
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ether_header:                                                      │
│    - dst_mac (6B)                                                  │
│    - src_mac (6B)                                                  │
│    - type (2B):                                                     │
│        ├─ 0x0800 → IPv4                                            │
│        ├─ 0x0806 → ARP                                             │
│        └─ 0x8100 → VLAN-tagged                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
            type == 0x8100?                 type != 0x8100
                    │                               │
                    ▼                               │
            ┌───────────────────┐                   │
            │  VLAN-tagged packet │                   │
            │  next_hdr_offset = 18                   │
            │  type = vlan->tpid   │                   │
            │  p->priority = PCP     │                   │
            └───────────────────┘                   │
                    │                               │
                    ▼                               ▼
            switch(type)                   switch(type)
                │                               │
                ├─► IPv4  → ip4_input()           ├─► IPv4  → ip4_input()
                ├─► ARP   → etharp_input()        ├─► ARP   → etharp_input()
                └─► ...                          └─► ...
```

---

## 5. PCP (Priority Code Point)

### 5.1 802.1p Priority

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

### 5.2 PCP 提取

```c
// 从 pbuf->priority 访问 PCP
p->priority = PP_HTONS(vlan->prio_vid) >> 13;
```

在 SafeOS 中，`p->priority` 用于 **TX 方向**的 VLAN PCP 设置。

---

## 6. lwip_hook_vlan_check_fn

**文件**: `external/lwip_ds_mcu/src/netif/ethernet.c:423-441`

```c
int lwip_hook_vlan_check_fn(void *netif, void *eth_hdr, void *vlan_hdr)
{
    struct netif *_netif = (struct netif *)netif;
    struct eth_hdr *_ethhdr = (struct eth_hdr *)eth_hdr;
    struct eth_vlan_hdr *_vlan_hdr = (struct eth_vlan_hdr *)vlan_hdr;

    // 检查 netif 是否配置了 VLAN
    if (_netif->vlanid == NO_VLANID) {
        return 0;  // 此 netif 不处理 VLAN，不匹配
    }

    // 比较 VID
    if ((_vlan_hdr->prio_vid & VLAN_ID_MASK) == (_netif->vlanid & VLAN_ID_MASK)) {
        return 1;  // VID 匹配
    }

    return 0;  // VID 不匹配
}
```

---

## 7. 与 ethernet_output 的关系

### 7.1 TX VLAN 插入

**文件**: `external/lwip_ds_mcu/src/netif/ethernet.c:333-395`

```c
ethernet_output(netif, p, src, dst, eth_type)
{
    struct eth_hdr *ethhdr;
    u16_t eth_type_be = lwip_htons(eth_type);

    // ============================================
    // [可选] VLAN Tag 插入
    // ============================================
    #if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
    s32_t vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
    if (vlan_prio_vid >= 0) {
        // 需要插入 VLAN Tag
        pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);
        vlanhdr = (struct eth_vlan_hdr *)(((u8_t *)p->payload) + SIZEOF_ETH_HDR);

        vlanhdr->tpid = eth_type_be;      // 保存原始 EtherType
        vlanhdr->prio_vid = lwip_htons((u16_t)vlan_prio_vid);

        eth_type_be = PP_HTONS(ETHTYPE_VLAN);  // 改为 VLAN EtherType
    }
    #endif

    // Ethernet Header 填充...
}
```

### 7.2 lwip_hook_vlan_set_fn

```c
int lwip_hook_vlan_set_fn(void *netif, void *pbuf,
                          const void *src, const void *dst, u16_t eth_type)
{
    struct netif *_netif = (struct netif *)netif;

    if (_netif->vlanid == NO_VLANID) {
        return -1;  // 不插入 VLAN Tag
    }

    // 组合 PCP (高 3 位) + VID (低 12 位)
    u16_t vlan_id = _netif->vlanid;
    vlan_id = ((_pbuf->priority << 13) | vlan_id);  // pbuf->priority 提供 PCP

    return vlan_id;  // 返回 >= 0，触发 VLAN Tag 插入
}
```

---

## 8. 总结

### 8.1 VLAN 解析核心

```
RX (ethernet_input):
1. 检查 EtherType == 0x8100
2. 提取 vlan->tpid 作为真正的 EtherType
3. 提取 vlan->prio_vid >> 13 作为 PCP (存入 p->priority)
4. VID 匹配在 lwip_arp_filter_netif_fn 中进行

TX (ethernet_output):
1. 检查 netif->vlanid 是否需要插入
2. 如果需要，pbuf_add_header 插入 VLAN Tag
3. 填充 tpid = 原始 EtherType
4. 填充 prio_vid = (PCP << 13) | VID
```

### 8.2 关键设计

1. **TPID**: 固定值 0x8100，表示 VLAN-tagged frame
2. **TCI**: 包含 PCP (优先级) 和 VID (VLAN ID)
3. **PCP 传递**: RX 时存入 `p->priority`，TX 时取出设置 VLAN PCP
4. **VID 匹配**: 在 `lwip_arp_filter_netif_fn` 中通过 NETIF_FOREACH 匹配

### 8.3 SafeOS 特供

无明显特供修改，VLAN 解析保持标准 lwIP IEEE 802.1Q 实现。
