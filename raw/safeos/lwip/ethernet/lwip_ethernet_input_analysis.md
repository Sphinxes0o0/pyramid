# ethernet_input 分析 — T-012

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: ethernet_input 函数：Ethernet header 解析、VLAN tag 处理、L2→L3 分发

---

## 1. 概述

`ethernet_input()` 是 lwIP 的 **L2→L3 分发入口**，负责：
1. 解析 Ethernet Header
2. 处理 VLAN Tag
3. 选择正确的 netif
4. 分发到上层协议 (IP/ARP/...)

### 1.1 调用链

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

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/netif/ethernet.c:89`

```c
ethernet_input(struct pbuf *p, struct netif *netif)
{
    struct eth_hdr *ethhdr;
    u16_t type;

    // ============================================
    // Step 1: 解析 Ethernet Header
    // ============================================
    ethhdr = (struct eth_hdr *)p->payload;
    type = ethhdr->type;  // 原始 EtherType

    // ============================================
    // Step 2: [VLAN分发关键] LWIP_ARP_FILTER_NETIF
    // ============================================
#if LWIP_ARP_FILTER_NETIF
    netif = LWIP_ARP_FILTER_NETIF_FN(p, netif, lwip_htons(type));
    if(NULL == netif) {
        goto free_and_return;
    }
#endif

    // ============================================
    // Step 3: 设置 pbuf 的 if_idx
    // ============================================
    if (p->if_idx == NETIF_NO_INDEX) {
        p->if_idx = netif_get_index(netif);
    }

    // ============================================
    // Step 4: [VLAN 解析] ETHARP_SUPPORT_VLAN
    // ============================================
#if ETHARP_SUPPORT_VLAN
    if (type == PP_HTONS(ETHTYPE_VLAN)) {  // 0x8100
        struct eth_vlan_hdr *vlan = (struct eth_vlan_hdr *)
            (((char *)ethhdr) + SIZEOF_ETH_HDR);
        next_hdr_offset = SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR;  // = 18 bytes

        // MAC_VLAN_FILTER: 可选的 VLAN ID 安全检查
#ifdef MAC_VLAN_FILTER
        if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
            pbuf_free(p);
            return ERR_OK;
        }
#endif

        type = vlan->tpid;  // 提取真正的 EtherType
        p->priority = PP_HTONS(vlan->prio_vid) >> 13;  // 提取 PCP
    }
#endif

    // ============================================
    // Step 5: 多播/广播标志
    // ============================================
    if (ethhdr->dest.addr[0] & 1) {
        // 标记 PBUF_FLAG_LLMCAST 或 PBUF_FLAG_LLBCAST
    }

    // ============================================
    // Step 6: AF-PACKET 捕获 (在 VLAN 解析之后!)
    // ============================================
#if LWIP_IPV4 && LWIP_ARP
    raw_afpacket_input(p, netif, type);
#endif

    // ============================================
    // Step 7: 分发到上层协议
    // ============================================
    switch (type) {
        case ETHTYPE_IP:
            pbuf_remove_header(p, next_hdr_offset);
            ip4_input(p, netif);
            break;
        case ETHTYPE_ARP:
            etharp_input(p, netif);
            break;
        // ...
    }
}
```

---

## 3. VLAN 解析详解

### 3.1 IEEE 802.1Q VLAN Tag

```
Ethernet Header (14 bytes) + VLAN Tag (4 bytes)
┌─────────────────────────────────────┬──────────────────────────────┐
│ Destination MAC (6B)                 │ Source MAC (6B)            │
├─────────────────────────────────────┴──────────────────────────────┤
│ TPID (2B) = 0x8100                   │ TCI (2B)                    │
├─────────────────────────────────────┼──────────────────────────────┤
│ PCP (3 bits) │ DEI (1 bit) │ VID (12 bits)                      │
└─────────────────────────────────────┴──────────────────────────────┘
```

### 3.2 VLAN Header 结构

```c
// lwip/prot/ethernet.h
struct eth_vlan_hdr {
    u16_t prio_vid;  // Bit[15:13] = PCP, Bit[12] = DEI, Bit[11:0] = VID
    u16_t tpid;      // Tag Protocol Identifier (= 0x8100)
};

#define VLAN_ID(vlan_hdr) (lwip_htons((vlan_hdr)->prio_vid) & 0xFFF)
```

---

## 4. lwip_arp_filter_netif_fn — VLAN 分发核心

**文件**: `ethernet.c:459-517`

```c
struct netif *lwip_arp_filter_netif_fn(void *_p, void *_netifIn, u16_t type)
{
    struct pbuf *p = (struct pbuf *)_p;
    struct netif *netifIn = (struct netif *)_netifIn;
    struct netif *netif = NULL;

    switch (type) {
        // ========================================
        // VLAN-tagged Packet: 通过 VLAN ID 匹配
        // ========================================
        case ETHTYPE_VLAN: {
            NETIF_FOREACH(netif) {
                if (netif_is_up(netif)) {
                    struct eth_vlan_hdr *vlan_hdr =
                        (struct eth_vlan_hdr *)(((char *)ethhdr) + SIZEOF_ETH_HDR);
                    u16_t vid = netif->vlanid & VLAN_ID_MASK;
                    if (vid == (vlan_hdr->prio_vid & VLAN_ID_MASK)) {
                        return netif;  // 找到匹配的 VLAN netif!
                    }
                }
            }
            return NULL;  // 无匹配，丢弃
        }

        // ========================================
        // 非 VLAN Packet: 通过 IP 地址 + vlanid==0 匹配
        // ========================================
        case ETHTYPE_IP: {
            ip_addr_copy_from_ip4(dst, iphdr->dest);
            NETIF_FOREACH(netif) {
                if (netif_is_up(netif) &&
                    ip4_addr_cmp(&dst, &netif->ip_addr) &&  // IP 匹配
                    netif->vlanid == 0u) {                   // 非 VLAN netif
                    return netif;
                }
            }
            break;
        }

        case ETHTYPE_ARP:
            // 类似 IP 的逻辑
            break;
    }

    return netifIn;
}
```

---

## 5. 分发决策矩阵

| Packet 类型 | 分发条件 | 返回的 netif |
|------------|----------|--------------|
| **VLAN-tagged** (VID=100) | `netif->vlanid == 100` | `vlan_if[100]` |
| **VLAN-tagged** (VID=200) | `netif->vlanid == 200` | `vlan_if[200]` |
| **非 VLAN** (IP=172.20.0.1) | `netif->ip_addr == 172.20.0.1 && netif->vlanid == 0` | `vnet_if` |
| **VLAN** 但无匹配 | 所有 `netif->vlanid != packet VID` | `NULL` (丢弃) |

---

## 6. 关键设计点

### 6.1 两层 VLAN 过滤

| 层次 | 机制 | 位置 |
|------|------|------|
| **第一层** | `LWIP_ARP_FILTER_NETIF` | Step 2 (line 121-127) |
| **第二层** | `MAC_VLAN_FILTER` | Step 4 (line 146-158) |

### 6.2 netif vs VLAN netif

- **vnet_if**: `input = ethernet_input`, `vlanid = 0`
- **vlan_if[i]**: `input = tcpip_input` (从未被直接调用!), `vlanid = 100/200/...`

### 6.3 p->if_idx 设置

```c
if (p->if_idx == NETIF_NO_INDEX) {
    p->if_idx = netif_get_index(netif);  // 记录接收的 netif
}
```

这个值在 UDP/TCP socket 绑定检查中使用 (`udp_input_local_match`).

---

## 7. 与其他模块的关系

### 7.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **rx_callback** | `vnet_if.input(p, &vnet_if)` | 物理网口 RX |

### 7.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **lwip_arp_filter_netif_fn** | VLAN-aware netif 选择 | 核心分发逻辑 |
| **ip4_input** | IP 层入口 | 分发 IP packet |
| **etharp_input** | ARP 层入口 | 分发 ARP packet |
| **raw_afpacket_input** | AF-PACKET 捕获 | socket 捕获 |

---

## 8. 总结

### 8.1 ethernet_input 的核心作用

```
接收 packet
    │
    ├─► 解析 ETH Header
    │
    ├─► [VLAN 分发] 通过 VLAN ID 找到 vlan_if[i]
    │     或通过 IP 地址 + vlanid==0 找到 vnet_if
    │
    ├─► 解析 VLAN Tag (如果存在)
    │     - 更新 type (移除 VLAN tag 后的 EtherType)
    │     - 更新 p->priority (PCP)
    │
    ├─► AF-PACKET 捕获
    │
    └─► 分发到上层协议
          - ETHTYPE_IP → ip4_input
          - ETHTYPE_ARP → etharp_input
```

### 8.2 关键设计

1. **VLAN-aware netif 选择**: `LWIP_ARP_FILTER_NETIF` 在 VLAN 解析之前选择正确的 netif
2. **VLAN Tag 剥离**: `type = vlan->tpid` 移除 VLAN tag，暴露真正的 EtherType
3. **PCP 传递**: `p->priority = vlan->prio_vid >> 13` 将 802.1Q PCP 传递到 pbuf

### 8.3 VLAN 分发特点

- **VLAN netif 的 input 函数从未被直接调用**
- **所有 packet 都经过 vnet_if.input = ethernet_input**
- **VLAN 分发靠 `LWIP_ARP_FILTER_NETIF` 实现**
