# lwip_arp_filter_netif_fn 分析 — T-052

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: lwip_arp_filter_netif_fn 函数：VLAN-aware netif 选择、ARP filter、packet 分发核心

---

## 1. 概述

`lwip_arp_filter_netif_fn()` 是 lwIP 的 **VLAN-aware netif 选择函数**，负责：
1. 根据 Ethernet Type 选择正确的 netif
2. VLAN-tagged packet 通过 VID 匹配
3. 非 VLAN packet 通过 IP 地址匹配
4. 支持 ARP/IP/IPVLAN 分发

### 1.1 调用链

```
rx_callback
    │
    ▼
ethernet_input(p, &vnet_if)
    │
    ▼
LWIP_ARP_FILTER_NETIF(p, netif, type)
    │
    ▼
lwip_arp_filter_netif_fn(p, netif, type)
    │
    ├─► ETHTYPE_ARP → 通过 IP 地址 + vlanid==0 匹配
    ├─► ETHTYPE_IP → 通过 IP 地址 + vlanid==0 匹配
    └─► ETHTYPE_VLAN → 通过 VLAN ID 匹配
```

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/netif/ethernet.c:459-517`

### 2.1 函数定义

```c
struct netif *lwip_arp_filter_netif_fn(void *_p, void *_netifIn, u16_t type)
{
    struct pbuf *p = (struct pbuf *)_p;
    struct netif *netifIn = (struct netif *)_netifIn;
    struct netif *netif = NULL;
    struct etharp_hdr *etharphdr = NULL;
    struct ip_hdr *iphdr = NULL;
    ip_addr_t src, dst;

    switch (type)
    {
        // ============================================
        // Case 1: ARP Packet
        // ============================================
        case ETHTYPE_ARP: {
            etharphdr = (struct etharp_hdr *)((unsigned char *)p->payload + 14);
            memcpy(&dst, &etharphdr->dipaddr, sizeof(ip4_addr_t));

            NETIF_FOREACH(netif) {
                // 匹配条件: IP 地址匹配 + vlanid == 0
                if (netif_is_up(netif) &&
                    ip4_addr_cmp(&dst, &(netif->ip_addr)) &&
                    netif->vlanid == 0u) {
                    return netif;  // ← 找到匹配的 netif
                }
            }
            break;
        }

        // ============================================
        // Case 2: IP Packet (非 VLAN)
        // ============================================
        case ETHTYPE_IP: {
            iphdr = (struct ip_hdr *)((unsigned char *)p->payload + 14);
            ip_addr_copy_from_ip4(dst, iphdr->dest);

            // 广播特殊处理
            if (dst.addr == IPADDR_BROADCAST) {
                // 广播使用 default netif
            } else {
                NETIF_FOREACH(netif) {
                    if (netif_is_up(netif) && (
                           // 单播: 目的 IP 匹配
                           ip4_addr_cmp(&dst, &(netif->ip_addr)) ||
                           // 多播: 检查多播组成员
                           ip_in_multicast_group(&dst, netif)) &&
                           netif->vlanid == 0u) {  // ← 关键! 非 VLAN netif
                        return netif;
                    }
                }
            }
            break;
        }

        // ============================================
        // Case 3: VLAN-tagged Packet
        // ============================================
        case ETHTYPE_VLAN: {
            NETIF_FOREACH(netif) {
                if (netif_is_up(netif)) {
                    struct eth_hdr *ethhdr = (struct eth_hdr *)p->payload;
                    struct eth_vlan_hdr *vlan_hdr =
                        (struct eth_vlan_hdr *)(((char *)ethhdr) + SIZEOF_ETH_HDR);

                    // 从 netif 的 vlanid 和 packet 的 VID 比较
                    u16_t vid = PP_HTONS(netif->vlanid) & PP_HTONS(VLAN_ID_MASK);
                    if (vid == (vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK))) {
                        return netif;  // ← VID 匹配!
                    }
                }
            }
            return NULL;  // VLAN packet 但无匹配 → 丢弃
        }

        default:
            break;
    }

    return netifIn;  // fallback 到输入 netif
}
```

---

## 3. VLAN 分发决策矩阵

### 3.1 分发逻辑

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ethernet_input 入口                                  │
│                    (p->payload = Ethernet Header)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    LWIP_ARP_FILTER_NETIF (Type 检查)
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
    ETHTYPE_ARP              ETHTYPE_IP               ETHTYPE_VLAN
          │                         │                         │
          ▼                         ▼                         ▼
    从 ARP header            从 IP header              从 VLAN Tag
    提取目的 IP              提取目的 IP               提取 VID
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    │
                                    ▼
                    NETIF_FOREACH 遍历所有 netif
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
    ip4_addr_cmp()           ip4_addr_cmp()            VID 比较
    + vlanid == 0            + vlanid == 0             (netif->vlanid)
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    │
                                    ▼
                        返回匹配的 netif
                        (或 netifIn 作为 fallback)
```

### 3.2 VID 匹配详解

```c
// Packet 中的 VID (来自 VLAN Tag)
u16_t packet_vid = vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK);

// Netif 中的 VID (配置值)
u16_t netif_vid = PP_HTONS(netif->vlanid) & PP_HTONS(VLAN_ID_MASK);

// 比较
if (netif_vid == packet_vid) {
    return netif;  // VID 匹配!
}
```

---

## 4. 与 ethernet_input 的关系

### 4.1 调用位置

**文件**: `ethernet.c:89-117` (ethernet_input 函数)

```c
ethernet_input(struct pbuf *p, struct netif *netif)
{
    struct eth_hdr *ethhdr;
    u16_t type;

    ethhdr = (struct eth_hdr *)p->payload;
    type = ethhdr->type;

    // ============================================
    // Step 2: VLAN 分发 — 调用 lwip_arp_filter_netif_fn
    // ============================================
    #if LWIP_ARP_FILTER_NETIF
    netif = LWIP_ARP_FILTER_NETIF_FN(p, netif, lwip_htons(type));
    if(NULL == netif) {
        goto free_and_return;
    }
    #endif

    // 后续 VLAN 解析...
}
```

### 4.2 调用时机

| EtherType | 调用 lwip_arp_filter_netif_fn? | 分发依据 |
|-----------|-------------------------------|----------|
| **ETHTYPE_ARP** | ✅ | 目的 IP + vlanid==0 |
| **ETHTYPE_IP** | ✅ | 目的 IP + vlanid==0 |
| **ETHTYPE_VLAN** | ✅ | VLAN ID |
| **其他** | ❌ | 使用 netifIn |

---

## 5. SafeOS 中的 netif 链表

### 5.1 netif 结构

```
netif_list (链表头)
      │
      ▼
┌─────────────┐    next     ┌─────────────┐    next     ┌─────────────┐
│   vnet_if   │ ─────────► │  vlan_if[0] │ ─────────► │  vlan_if[1] │
│ (物理网口)   │            │  (VLAN 100) │            │  (VLAN 200) │
└─────────────┘            └─────────────┘            └─────────────┘
     .vlanid = 0                .vlanid = 100              .vlanid = 200
     .ip_addr                   .ip_addr                   .ip_addr
     =172.20.0.1               =172.20.100.1              =172.20.200.1
```

### 5.2 分发示例

| 收到的 Packet | EtherType | VID/IP | 分发到 |
|--------------|-----------|--------|--------|
| 非 VLAN, IP=172.20.0.1 | ETHTYPE_IP | 172.20.0.1 | vnet_if |
| VLAN VID=100 | ETHTYPE_VLAN | VID=100 | vlan_if[0] |
| VLAN VID=200 | ETHTYPE_VLAN | VID=200 | vlan_if[1] |
| VLAN VID=300 | ETHTYPE_VLAN | VID=300 | NULL (丢弃) |

---

## 6. 关键设计点

### 6.1 两阶段分发

```
第一阶段: lwip_arp_filter_netif_fn (在 ethernet_input 开始时调用)
   └─► 选择正确的 netif (vnet_if 或 vlan_if[i])

第二阶段: ethernet_input 内部 VLAN 解析
   └─► 剥离 VLAN Tag，提取真正的 EtherType
```

### 6.2 vlanid=0 的含义

```c
netif->vlanid == 0u  // 表示这是物理网口，不带 VLAN Tag
```

- **vnet_if**: `vlanid = 0` (物理网口)
- **vlan_if[i]**: `vlanid = 100/200/...` (VLAN 网口)

### 6.3 多播处理

```c
// IP 分发时检查多播组成员
ip_in_multicast_group(&dst, netif)
```

这允许多播 packet 也能正确分发到加入该多播组的 netif。

---

## 7. 性能特征

### 7.1 复杂度

```
O(n) — 遍历 netif_list 直到找到匹配
n = netif 数量 (通常 1-5 个)
```

### 7.2 优化建议

1. **按 vlanid 排序**: 如果 netif 数量增加，可以按 vlanid 排序二分查找
2. **VLAN ID 哈希表**: 为 VLAN netif 建立 VID → netif 哈希表
3. **缓存最近匹配**: 利用 locality，最近匹配的 netif 缓存

---

## 8. 与 Linux 的对比

### 8.1 Linux 的 VLAN 分发

Linux 使用 **VLAN device (veth)** 方式：
```
eth0 (物理网口)
    │
    ├── vlan100 (虚拟设备)
    │       └── 接收 VLAN 100 tagged packet
    │
    └── vlan200 (虚拟设备)
            └── 接收 VLAN 200 tagged packet
```

每个 VLAN device 有独立的 netdev_ops，通过 **netdev_rx_handler** 注册到物理网口。

### 8.2 SafeOS (lwIP) 的 VLAN 分发

```
vnet_if (物理网口，vlanid=0)
    │
    ├── vlan_if[0] (VLAN 100，vlanid=100)
    └── vlan_if[1] (VLAN 200，vlanid=200)

所有 packet 都经过 vnet_if.input = ethernet_input
通过 lwip_arp_filter_netif_fn 软分发
```

---

## 9. 总结

### 9.1 lwip_arp_filter_netif_fn 的核心作用

```
接收 packet
    │
    ├─► ETHTYPE_ARP → 提取目的 IP，匹配 netif_ip_addr + vlanid==0
    ├─► ETHTYPE_IP → 提取目的 IP，匹配 netif_ip_addr + vlanid==0
    └─► ETHTYPE_VLAN → 提取 VID，匹配 netif->vlanid
```

### 9.2 关键设计

1. **软分发**: 所有 packet 经过 vnet_if，通过函数指针选择真正处理者
2. **vlanid=0 标记**: 物理网口使用 vlanid=0，与 VLAN netif 区分
3. **VID 精确匹配**: VLAN packet 必须 VID 完全匹配才能接收
4. **Fallback**: 无匹配时返回 netifIn，让 ethernet_input 自己处理

### 9.3 SafeOS 特供

这是 **SafeOS lwIP 的核心 VLAN 分发机制**，区别于标准 lwIP：
- 标准 lwIP: 依赖 `netif->input` 回调，VLAN device 的 input 从不被调用
- SafeOS lwIP: 统一通过 `lwip_arp_filter_netif_fn` 软分发
