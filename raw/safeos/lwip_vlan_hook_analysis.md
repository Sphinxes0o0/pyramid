# VLAN Hook 函数分析 — T-062/T-063

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: MAC_VLAN_FILTER hook 实现 (T-062)、TX VLAN tag 插入 hook (T-063)

---

## 1. 概述

lwIP 提供两个 VLAN 相关 hooks：
1. **RX Hook**: `LWIP_HOOK_VLAN_CHECK` / `lwip_hook_vlan_check_fn` — RX 时检查 VLAN ID 是否匹配
2. **TX Hook**: `LWIP_HOOK_VLAN_SET` / `lwip_hook_vlan_set_fn` — TX 时决定是否插入 VLAN tag

---

## 2. RX Hook: lwip_hook_vlan_check_fn

### 2.1 调用位置

**文件**: `ethernet.c:146-159`

```c
#ifdef MAC_VLAN_FILTER
#if defined(LWIP_HOOK_VLAN_CHECK) || defined(ETHARP_VLAN_CHECK) || defined(ETHARP_VLAN_CHECK_FN)
#ifdef LWIP_HOOK_VLAN_CHECK
    if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
#elif defined(ETHARP_VLAN_CHECK_FN)
    if (!ETHARP_VLAN_CHECK_FN(ethhdr, vlan)) {
#elif defined(ETHARP_VLAN_CHECK)
    if (VLAN_ID(vlan) != ETHARP_VLAN_CHECK) {
#endif
      /* silently ignore this packet: not for our VLAN */
      pbuf_free(p);
      return ERR_OK;
    }
#endif
#endif
```

### 2.2 函数实现

**文件**: `ethernet.c:398-421`

```c
#if ETHARP_SUPPORT_VLAN
int lwip_hook_vlan_check_fn(void *netif, void *eth_hdr, void *vlan_hdr)
{
    struct netif *_netif = (struct netif *)netif;
    struct eth_hdr *_eth_hdr = (struct eth_hdr *)eth_hdr;
    struct eth_vlan_hdr *_vlan_hdr = (struct eth_vlan_hdr *)vlan_hdr;

    // 检查 EtherType 是否为 VLAN (0x8100)
    if (_eth_hdr->type != PP_HTONS(ETHTYPE_VLAN)) {
        return 0;
    }

    if (netif_is_up(_netif)) {
        // 提取 netif 的 VLAN ID
        u16_t vid = PP_HTONS(_netif->vlanid) & PP_HTONS(VLAN_ID_MASK);

        // 提取 packet 的 VLAN ID
        if (vid == (_vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK))) {
            LWIP_DEBUGF(ETHARP_DEBUG, ("lwip_hook_vlan_check_fn: netif %c%c vlanid %d\n",
                _netif->name[0], _netif->name[1], _netif->vlanid));
            return 1;  // 匹配
        }
    }

    return 0;  // 不匹配
}
#endif
```

### 2.3 关键设计点

| 字段 | 说明 |
|------|------|
| `netif->vlanid` | netif 的 VLAN ID |
| `VLAN_ID_MASK` | 0x0FFF (取低 12 位) |
| `vlan_hdr->prio_vid` | 高 3 位为 PCP，低 12 位为 VID |

### 2.4 VLAN ID 提取逻辑

```c
// netif 的 VLAN ID (小端序)
u16_t vid = PP_HTONS(_netif->vlanid) & PP_HTONS(VLAN_ID_MASK);

// packet 的 VLAN ID
u16_t pkt_vid = _vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK);

// 比较
if (vid == pkt_vid)  // 匹配
```

---

## 3. TX Hook: lwip_hook_vlan_set_fn

### 3.1 调用位置

**文件**: `ethernet.c:339-354`

```c
#if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
  s32_t vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
  if (vlan_prio_vid >= 0) {
    /* VLAN is supported and a vlan was set, so allocate room for it */
    if (pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR)) {
      goto pbuf_header_failed;
    }
    {
      struct eth_vlan_hdr *vlan = (struct eth_vlan_hdr *)(((u8_t *)p->payload) + SIZEOF_ETH_HDR);
      vlan->tpid = eth_type_be;  /* @todo? could @to an endian issue? */
      vlan->prio_vid = lwip_htons(vlan_prio_vid);
      eth_type_be = PP_HTONS(ETHTYPE_VLAN);
    }
  }
#endif
```

### 3.2 函数实现

**文件**: `ethernet.c:423-441`

```c
int lwip_hook_vlan_set_fn(void* netif, void* pbuf, const void* src, const void* dst, u16_t eth_type)
{
    struct netif *_netif = (struct netif *)netif;
    struct pbuf *_pbuf = (struct pbuf *)pbuf;
    struct eth_addr *_src = (struct eth_addr *)src;
    struct eth_addr *_dst = (struct eth_addr *)dst;

    u16_t vlan_id;

    // 检查是否为 NO_VLANID
    if (_netif->vlanid == NO_VLANID) {
        return -1;  // 不插入 VLAN tag
    }

    // 组合 VLAN ID 和 PCP
    vlan_id = _netif->vlanid;
    vlan_id = (((struct pbuf*)pbuf)->priority << 13) | vlan_id;

    return vlan_id;
}
```

### 3.3 返回值含义

| 返回值 | 含义 |
|--------|------|
| **-1** | 不插入 VLAN tag |
| **>= 0** | 插入 VLAN tag，值为 (PCP << 13) \| VID |

### 3.4 VLAN Tag 插入流程

```
1. lwip_hook_vlan_set_fn() 返回 vlan_id = (PCP << 13) | VID
2. pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR)  // 前移 18 字节
3. 填充 VLAN header:
   - vlan->tpid = 原始 EtherType (如 0x0800 for IP)
   - vlan->prio_vid = lwip_htons(vlan_id)
4. 更新 EtherType: eth_type_be = 0x8100 (VLAN)
```

---

## 4. pbuf->priority 与 PCP

### 4.1 PCP (Priority Code Point)

| PCP 值 | 优先级 | 用途 |
|--------|--------|------|
| 0 | Best Effort | 普通流量 |
| 1 | Background | 低优先级 |
| 2 | Excellent Effort | |
| 3 | Controlled Load | |
| 4 | Reserved | |
| 5 | Reserved | |
| 6 | Reserved | |
| 7 | Reserved | |

### 4.2 pbuf->priority 设置

在 RX 方向 (ethernet.c:167)：
```c
p->priority = PP_HTONS(vlan->prio_vid) >> 13;  // 提取 PCP
```

在 TX 方向，priority 可以在协议栈上层设置（如 UDP/TCP socket）。

---

## 5. SafeOS 中的配置

### 5.1 物理网口 (vnet_if)

```c
// vnet_if.vlanid = 0 (NO_VLANID)
netif->vlanid = NO_VLANID;
```

### 5.2 VLAN 网口 (vlan_if[i])

```c
// vlan_if[i].vlanid = 配置值 (如 100, 200)
netif->vlanid = atoi(vlan_conf[conf_idx].vid);
```

### 5.3 TX 行为

| 网口类型 | vlanid | lwip_hook_vlan_set_fn 返回 | 行为 |
|----------|--------|---------------------------|------|
| 物理网口 | 0 (NO_VLANID) | -1 | 不插入 VLAN tag |
| VLAN 网口 | 100 | (pbuf->priority << 13) \| 100 | 插入 VLAN tag |

---

## 6. 与 Linux 对比

### 6.1 Linux VLAN 发送

```c
// drivers/net/vlan.c
static netdev_tx_t vlan_dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct vlan_dev_priv *vlan = vlan_dev_priv(dev);

    // 添加 VLAN tag
    vhdr->h_vlan_TCI = htons(vlan->vlan_id | vlan->priority);
    vhdr->h_vlan_proto = htons(ETH_P_8021Q);

    // 发送到物理设备
    return dev_queue_xmit(skb);
}
```

### 6.2 关键差异

| 特性 | lwIP (SafeOS) | Linux |
|------|---------------|-------|
| **VLAN ID 来源** | `netif->vlanid` | `vlan_dev_priv->vlan_id` |
| **PCP 来源** | `pbuf->priority` | `skb->priority` |
| **Hook 机制** | 可选的 hook 函数 | 专用 driver |
| **NO_VLANID 处理** | 返回 -1 | skb->vlan_tci = 0 |

---

## 7. 总结

### 7.1 RX 流程 (VLAN 检查)

```
NIC DMA
    │
    ▼
ethernet_input(p, &vnet_if)
    │
    ▼
解析 VLAN tag:
  type = ETHTYPE_VLAN (0x8100)
  vlan = ethhdr + 14 bytes
  p->priority = vlan->prio_vid >> 13
    │
    ▼
#ifdef MAC_VLAN_FILTER
  lwip_hook_vlan_check_fn(netif, ethhdr, vlan)
    │
    ├─► 检查 vnet_if.vlanid == packet VID
    └─► vnet_if.vlanid = 0 → 所有 VLAN packet 不匹配 → 丢弃！
#endif
```

### 7.2 TX 流程 (VLAN 插入)

```
App send()
    │
    ▼
ethernet_output(netif, p, ...)
    │
    ▼
LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type)
    │
    ├─► netif->vlanid == NO_VLANID? → return -1 (不插入)
    └─► netif->vlanid == 100? → return (pbuf->priority << 13) | 100
    │
    ▼
pbuf_add_header(p, 18)  // 腾出 VLAN tag 空间
    │
    ▼
填充 VLAN header:
  vlan->tpid = 原始 EtherType
  vlan->prio_vid = PCP | VID
  EtherType = 0x8100
```

### 7.3 关键设计

1. **VLAN ID 匹配**：使用 12 位 VID (0-4095)
2. **PCP 优先级**：从 pbuf->priority 提取
3. **NO_VLANID**：vlanid=0 表示不处理 VLAN
4. **TX 插入**：由 netif->vlanid 控制是否插入
