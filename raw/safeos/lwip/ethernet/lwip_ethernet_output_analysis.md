# ethernet_output 分析 — T-013

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: ethernet_output 函数：L3→L2 封装、VLAN tag 插入、AF-PACKET 输出

---

## 1. 概述

`ethernet_output()` 是 lwIP 的 **L3→L2 封装出口**，负责：
1. 添加 Ethernet Header
2. 可选插入 VLAN Tag
3. AF-PACKET 输出捕获
4. 调用 `netif->linkoutput` 发送到链路

### 1.1 调用链

```
App send()
    │
    ▼
socket API
    │
    ▼
TCP/UDP output
    │
    ▼
etharp_output() / ethip6_output()
    │
    ▼
ethernet_output(netif, p, src, dst, ETHTYPE_IP)
    │
    ├─► [VLAN 插入] LWIP_HOOK_VLAN_SET
    │     └─► pbuf_add_header(ETH_HDR + VLAN_HDR)
    │         └─► 填充 VLAN tag
    │
    ├─► raw_afpacket_output()  // AF-PACKET 捕获
    │
    └─► netif->linkoutput(netif, p)
          └─► ethif_link_output() → NIC DMA
```

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/netif/ethernet.c:333`

```c
ethernet_output(struct netif *netif, struct pbuf *p,
                const struct eth_addr *src,
                const struct eth_addr *dst,
                u16_t eth_type)
{
    struct eth_hdr *ethhdr;
    u16_t eth_type_be = lwip_htons(eth_type);

    // ============================================
    // Step 1: [可选] VLAN Tag 插入
    // ============================================
    #if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
    s32_t vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
    if (vlan_prio_vid >= 0) {
        // 需要插入 VLAN Tag
        pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);  // +4 bytes
        vlanhdr = (struct eth_vlan_hdr *)(((u8_t *)p->payload) + SIZEOF_ETH_HDR);
        vlanhdr->tpid = eth_type_be;      // 保存原始 EtherType
        vlanhdr->prio_vid = lwip_htons((u16_t)vlan_prio_vid);  // PCP + VID
        eth_type_be = PP_HTONS(ETHTYPE_VLAN);  // 改为 VLAN EtherType
    }
    #endif

    // ============================================
    // Step 2: 添加 Ethernet Header
    // ============================================
    pbuf_add_header(p, SIZEOF_ETH_HDR);  // +14 bytes
    ethhdr = (struct eth_hdr *)p->payload;
    ethhdr->type = eth_type_be;
    SMEMCPY(&ethhdr->dest, dst, ETH_HWADDR_LEN);
    SMEMCPY(&ethhdr->src,  src, ETH_HWADDR_LEN);

    // ============================================
    // Step 3: AF-PACKET 输出捕获
    // ============================================
    #if LWIP_IPV4 && LWIP_ARP
    raw_afpacket_output(p, netif);
    #endif

    // ============================================
    // Step 4: 发送到链路
    // ============================================
    return netif->linkoutput(netif, p);
}
```

---

## 3. VLAN Tag 插入详解

### 3.1 lwip_hook_vlan_set_fn

**文件**: `ethernet.c:423-441`

```c
int lwip_hook_vlan_set_fn(void *netif, void *pbuf,
                          const void *src, const void *dst, u16_t eth_type)
{
    struct netif *_netif = (struct netif *)netif;
    struct pbuf *_pbuf = (struct pbuf *)pbuf;

    // vlanid == 0 表示此 netif 不带 VLAN Tag
    if (_netif->vlanid == NO_VLANID) {
        return -1;  // 不插入 VLAN Tag
    }

    // 组合 PCP (高 3 位) + VID (低 12 位)
    u16_t vlan_id = _netif->vlanid;
    vlan_id = ((_pbuf->priority << 13) | vlan_id);  // pbuf->priority 提供 PCP

    return vlan_id;  // 返回 >= 0，触发 VLAN Tag 插入
}
```

### 3.2 VLAN Tag 结构

```
VLAN Tag 插入后:
┌─────────────────────────────────────┬──────────────────────────────────────┐
│ Ethernet Header (14 bytes)          │ VLAN Tag (4 bytes)                    │
├─────────────────────────────────────┼──────────────────────────────────────┤
│ DST (6B) │ SRC (6B) │ Type=0x8100 │ PCP (3b) │ DEI (1b) │ VID (12b) │
└─────────────────────────────────────┴──────────────────────────────────────┘
                                     ↑
                                     └── 从 vlanid 和 pbuf->priority 计算
```

---

## 4. SafeOS 中的 linkoutput

### 4.1 vnet_if.linkoutput

```c
// main.c:4720
netif->linkoutput = ethif_link_output;

// ethif_link_output (main.c:3788)
err_t ethif_link_output(struct netif *netif, struct pbuf *q)
{
    // 放入 pending_tx_buf_ring
    elem_ring_put(pending_tx_buf_ring, e);
    // 通知 NIC
    sel4_signal(nic_tx_ntfn);
    return ERR_OK;
}
```

### 4.2 vlan_if[i].linkoutput

```c
// vlanif.c:110
netif->linkoutput = low_level_output;

// low_level_output (vlanif.c:52)
static err_t low_level_output(struct netif *netif, struct pbuf *p)
{
    if (physical_netif != NULL) {
        physical_netif->linkoutput(physical_netif, p);  // 调用 vnet_if.linkoutput
    }
    return ERR_OK;
}
```

### 4.3 VLAN netif TX 流程

```
vlan_if[i].linkoutput = low_level_output
    │
    └─► physical_netif->linkoutput(physical_netif, p)
            │
            └─► ethif_link_output()
                    │
                    ├─► elem_ring_put(pending_tx_buf_ring)
                    └─► sel4_signal(nic_tx_ntfn)
```

---

## 5. 与其他模块的关系

### 5.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **etharp_output** | `ethernet_output()` | ARP 输出 |
| **ip4_output_if** | `ethernet_output()` | IP 输出 |

### 5.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **low_level_output** | VLAN netif 的 linkoutput | 调用物理网口 |
| **ethif_link_output** | 物理网口的 linkoutput | 发送到 NIC |

---

## 6. 总结

### 6.1 ethernet_output 的核心作用

```
收到 IP packet
    │
    ├─► [可选] 插入 VLAN Tag
    │     - 检查 netif->vlanid
    │     - pbuf->priority → PCP
    │     - netif->vlanid → VID
    │
    ├─► 添加 Ethernet Header
    │     - src MAC
    │     - dst MAC
    │     - Type (IP/ARP/VLAN)
    │
    ├─► AF-PACKET 输出捕获
    │
    └─► 调用 netif->linkoutput
          └─► 发送到物理网卡
```

### 6.2 关键设计

1. **VLAN 插入是可选的**: 由 `LWIP_HOOK_VLAN_SET` hook 控制
2. **VLAN netif 复用物理网口的 linkoutput**: `low_level_output` 直接调用 `physical_netif->linkoutput`
3. **VLAN tag 在 ethernet_output 中插入**: 而不是更低层
