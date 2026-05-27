---
type: source
source-type: github
created: 2026-05-27
title: "lwIP vs Linux VLAN Packet 分发机制对比分析"
date: 2026-04-22
size: medium
path: raw/safeos/docs/lwip_vlan_dispatch_analysis.md
summary: "lwIP与Linux内核VLAN packet分发机制深度对比：lwIP通过IP地址匹配+VLAN ID，Linux通过rx_handler精确匹配；LWIP_ARP_FILTER_NETIF是关键分发机制；VIRT_BRG模式下Bridge不理解VLAN tag"
tags: [safeos, lwip, vlan, netif, arp-filter, virt-brg, rx-handler, linux]
sources: []
---

# lwIP vs Linux VLAN Packet 分发机制对比分析

> 文档版本: 1.1 | 更新日期: 2026/04/22

## 核心差异

| 维度 | lwIP (SafeOS) | Linux |
|------|---------------|-------|
| **VLAN netif 本质** | 独立的 `struct netif`，拥有独立 IP | 虚拟 `net_device`，依赖物理设备 |
| **VLAN 识别方式** | IP 地址匹配 + 可选 VLAN ID 检查 | `netdev_rx_handler_register` 精确 VLAN ID 匹配 |
| **Packet 分发** | `ethernet_input` → `ip4_input` 遍历所有 netif | NIC driver 层遍历 rx_handlers 链 |
| **VLAN tag 处理** | lwIP 核心 hooks 解析/插入 | 专门的 VLAN net_device driver |

---

## lwIP VLAN 分发机制

### RX 方向 — 两层分发

**第一层: `LWIP_ARP_FILTER_NETIF`**

```c
// ethernet.c:121-127
netif = lwip_arp_filter_netif_fn(p, netif, type);
//  → switch(type):
//    ETHTYPE_VLAN: NETIF_FOREACH 找 netif->vlanid == packet VID
//    ETHTYPE_IP/ARP: NETIF_FOREACH 找 IP 匹配 && vlanid == 0
```

**第二层: `ip4_input_accept`**

```c
// ip4.c:607-618
NETIF_FOREACH(netif) {
    if (ip4_input_accept(netif)) {  // IP 地址匹配
        return netif;
    }
}
```

### TX 方向 — VLAN Tag 插入

```c
// ethernet_output() → LWIP_HOOK_VLAN_SET
vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
//  → netif->vlanid != 0 则返回 (pbuf->priority << 13) | netif->vlanid
```

### lwip_hook_vlan_check_fn — VLAN ID 安全检查

```c
// ethernet.c:398-421
int lwip_hook_vlan_check_fn(void *netif, void *eth_hdr, void *vlan_hdr) {
    u16_t vid = netif->vlanid & VLAN_ID_MASK;
    u16_t pkt_vid = vlan_hdr->prio_vid & VLAN_ID_MASK;
    return (vid == pkt_vid) ? 1 : 0;  // 1=接受, 0=丢弃
}
```

**陷阱**: 如果 `MAC_VLAN_FILTER=1` 且 `vnet_if.vlanid=0`，所有 VLAN-tagged packet 都会被丢弃。

---

## Linux VLAN 分发机制

### RX 方向 — rx_handler 链

```c
// drivers/net/vlan.c
static rx_handler_result_t vlan_rx_handler(struct sk_buff **pskb)
{
    vlan_dev_priv *vlan = vlan_dev_priv(skb->dev);
    __u16 vid = vhdr->h_vlan_TCI & VLAN_VID_MASK;

    if (vid != vlan->vlan_id) {
        return RX_HANDLER_PASS;  // 不匹配，交给其他 handler
    }
    // 剥离 VLAN tag，还原 EtherType
    skb->protocol = vhdr->h_vlan_proto;
    __skb_pull(skb, VLAN_HLEN);
    return RX_HANDLER_ANOTHER;  // 交给 VLAN device 协议栈
}
```

### TX 方向 — vlan_dev_hard_start_xmit

```c
// drivers/net/vlan.c
static netdev_tx_t vlan_dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    vhdr->h_vlan_proto = htons(ETH_P_8021Q);
    vhdr->h_vlan_TCI = htons(vlan->vlan_id | vlan->priority);
    skb->protocol = htons(ETH_P_8021Q);
    return dev_queue_xmit(skb->real_dev);  // 发送到物理设备
}
```

---

## 关键结论

1. **lwIP 的 VLAN 分发有两层机制**：
   - `LWIP_ARP_FILTER_NETIF=1`：通过 VLAN ID 匹配找到正确的 vlan_if[i]
   - `ip4_input_accept`：通过 IP 地址匹配作为最终验证

2. **Linux 的 VLAN 分发是基于 netdevice 和 rx_handler 的**

3. **AF_PACKET socket 绑定到 VLAN netif 无法工作**（`inp` 始终是 vnet_if）

4. **VIRT_BRG_SUPPORT 模式下 Bridge 不理解 VLAN tag**，可能导致 FDB 学习问题（MAC 地址偏移 4 字节）

5. **SafeOS 的实现是 IP 地址 + VLAN ID 的混合模型**，与 Linux 的纯 VLAN 模型不同

---

## 参考代码路径

| 文件 | 职责 |
|------|------|
| `vlanif.c` | VLAN netif 创建和管理 |
| `main.c` | 物理网口初始化 |
| `ethernet.c` | VLAN tag 解析/插入，`LWIP_ARP_FILTER_NETIF` |
| `ip4.c` | IP 层 netif 匹配 |
| `bridgeif.c` | lwIP bridgeif 实现 |
| `bridge.c` | VIRT_BRG_SUPPORT bridge 集成 |

---

## 相关页面

- [[entities/linux/lwip/lwip-vlan-implementation]] — lwIP IEEE 802.1Q VLAN 完整实现
- [[lwip-index]] — lwIP 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引
