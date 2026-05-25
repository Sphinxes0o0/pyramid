# lwIP vs Linux VLAN Packet 分发机制对比分析

> 文档版本: 1.1
> 更新日期: 2026/04/22
> 代码路径: `/home/shiyang/nio/nt35/safeos/`

---

## 1. 概述

本文档对比分析 lwIP (SafeOS) 和 Linux 内核的 VLAN packet 分发机制，聚焦于：
- **RX 方向**：当一个 VLAN-tagged packet 到达物理网卡时，如何分发到正确的 VLAN interface
- **TX 方向**：从 VLAN interface 发送时，如何正确插入 VLAN tag

### 核心架构差异

| 维度 | lwIP (SafeOS) | Linux |
|------|---------------|-------|
| **VLAN netif 本质** | 独立的 `struct netif`，拥有独立 IP | 虚拟 net_device，依赖物理设备 |
| **VLAN 识别方式** | IP 地址匹配 + 可选 VLAN ID 检查 | VLAN net_device 堆叠在物理设备上 |
| **Packet 分发** | `ethernet_input` → `ip4_input` 遍历所有 netif | VLAN net_device 注册 `.real_dev` 接收回调 |
| **VLAN tag 处理** | lwIP 核心 hooks 解析/插入 | 内核 VLAN driver 处理 |

---

## 2. lwIP VLAN Packet 分发机制

### 2.1 RX 方向 — Packet 分发流程

```
物理网卡 DMA
    │
    ▼
rx_callback() [main.c:4781]
    │
    ▼
vnet_if.input(p, &vnet_if)  ──────► ethernet_input() [ethernet.c:89]
    │ (input = ethernet_input)          │
    │                                   ├─► 解析 Ethernet Header
    │                                   ├─► if (ETHTYPE_VLAN) {
    │                                   │     解析 VLAN Header (TPID=0x8100)
    │                                   │     p->priority = PCP (高3位)
    │                                   │     #ifdef MAC_VLAN_FILTER
    │                                   │       LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)
    │                                   │         └─► lwip_hook_vlan_check_fn()
    │                                   │             检查 netif->vlanid == packet VID
    │                                   │             不匹配则 DROP
    │                                   │     #endif
    │                                   │   }
    │                                   ├─► pbuf_remove_header(ETH_HDR + VLAN_HDR)
    │                                   └─► ip4_input(p, &vnet_if)
    │                                         │
    │                                         ├─► ip4_input_accept(vnet_if)
    │                                         │   └─► 目的IP == vnet_if.ip_addr ?
    │                                         │       是 → netif = vnet_if
    │                                         │       否 → 继续查找
    │                                         │
    │                                         ├─► #if !LWIP_SINGLE_NETIF
    │                                         │     NETIF_FOREACH(netif) {
    │                                         │       ip4_input_accept(netif)
    │                                         │         └─► 目的IP == vlan_if[i].ip_addr ?
    │                                         │             是 → netif = vlan_if[i] (匹配!)
    │                                         │     }
    │                                         │
    │                                         └─► UDP/TCP/RAW socket 接收
    │
    ▼
    (VLAN netif 从未被直接调用 input 函数)
```

**关键点**：
1. **所有 packet 都经过 physical netif 的 `ethernet_input`**
2. **VLAN netif 的 `input = tcpip_input` 从未被直接调用**
3. **分发机制**：在 `ip4_input` 中，通过 IP 地址匹配找到正确的 netif

### 2.2 lwip_hook_vlan_check_fn — VLAN ID 检查 (MAC_VLAN_FILTER)

**文件**: `ethernet.c:398-421`

```c
int lwip_hook_vlan_check_fn(void *netif, void *eth_hdr, void *vlan_hdr)
{
    struct netif  *_netif  = (struct netif  *)netif;
    struct eth_vlan_hdr *_vlan_hdr = (struct eth_vlan_hdr *)vlan_hdr;

    // 提取本 netif 的 VLAN ID
    u16_t vid = PP_HTONS(_netif->vlanid) & PP_HTONS(VLAN_ID_MASK);
    // 提取 packet 中的 VLAN ID
    u16_t pkt_vid = _vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK);

    if (vid == pkt_vid) {
        return 1;  // 匹配，接受
    }
    return 0;  // 不匹配，丢弃
}
```

**SafeOS 中的行为**：
- `vnet_if.vlanid = 0` (物理网口无 VLAN ID)
- `vlan_if[i].vlanid = 配置值` (如 100, 200)
- **如果 MAC_VLAN_FILTER 启用且 vnet_if.vlanid=0，所有 VLAN-tagged packet 都会被丢弃！**

**结论**：SafeOS 中 MAC_VLAN_FILTER 很可能被禁用，或者 VLAN 分发完全依赖 IP 地址匹配。

### 2.3 TX 方向 — VLAN Tag 插入

```
App Socket 发送
    │
    ▼
协议栈构建 IP packet
    │
    ▼
etharp_output() [etharp.c]
    │
    ▼
ethernet_output(netif, p, src, dst, ETHTYPE_IP) [ethernet.c:333]
    │
    ├─► #if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
    │     vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type)
    │       └─► lwip_hook_vlan_set_fn() [ethernet.c:423]
    │             if (netif->vlanid == NO_VLANID) return -1;  // 不插 VLAN
    │             return (pbuf->priority << 13) | netif->vlanid;  // PCP+VID
    │
    │     if (vlan_prio_vid >= 0) {
    │       pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);  // +4 bytes
    │       vlanhdr->tpid = eth_type_be;       // 保存原始 EtherType
    │       vlanhdr->prio_vid = lwip_htons(vlan_prio_vid);  // PCP+VID
    │       eth_type_be = PP_HTONS(ETHTYPE_VLAN);  // 0x8100
    │     }
    │
    └─► netif->linkoutput(netif, p)
           │
           ├─► vlan_if[i].linkoutput = low_level_output()
           │     └─► physical_netif->linkoutput(physical_netif, p)
           │           └─► ethif_link_output() → NIC DMA
           │
           └─► vnet_if.linkoutput = ethif_link_output()
                 └─► ethif_link_output() → NIC DMA
```

**关键点**：
1. **VLAN tag 插入由 `LWIP_HOOK_VLAN_SET` hook 控制**
2. **每个 netif 的 `vlanid` 决定是否插入 VLAN tag**
3. **VLAN netif 的 `linkoutput = low_level_output`**，直接调用物理网口的 `linkoutput`

### 2.4 vlanif_init — VLAN netif 初始化

**文件**: `vlanif.c:93-150`

```c
err_t vlanif_init(struct netif *netif)
{
    // netif->state 指向配置索引
    int conf_idx = *((int *)netif->state);

    // 名称如 "vl0", "vl1"
    netif->name[0] = vlan_conf[conf_idx].ifName[0];
    netif->name[1] = vlan_conf[conf_idx].ifName[1];

    // IP 层输出函数
    #if LWIP_IPV4
    netif->output = etharp_output;
    #endif

    // 关键：linkoutput 调用物理网口的 linkoutput
    netif->linkoutput = low_level_output;

    // 从配置读取 VLAN ID (如 100)
    netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10);

    netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP |
                   NETIF_FLAG_LINK_UP | NETIF_FLAG_IGMP | NETIF_FLAG_ETHERNET;
    return ERR_OK;
}
```

---

## 3. Linux VLAN Packet 分发机制

### 3.1 核心架构 — VLAN net_device 堆叠

Linux 的 VLAN 实现是**真正的网络设备堆叠**：

```
┌─────────────────────────────────────────────────────────────┐
│                    User Space Application                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Socket Layer                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   VLAN net_device (e.g., eth0.100)           │
│   - ifindex                         - real_dev 指向 eth0     │
│   - vlan_id = 100                   - vlan_filter 数组      │
│   - dev->netdev_ops = &vlan_netdev_ops                       │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │      .real_dev_ops = eth_open  │
              │      .hard_start_xmit = vlan_dev_xmit
              │      接收回调 = netif_rx_internal
              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Physical net_device (eth0)                │
│   - 处理实际的 DMA                   - 发送 vlan_dev_xmit   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      NIC Driver (e.g., igb)                  │
│   - DMA 缓冲区管理                    - 中断处理             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 RX 方向 — VLAN Packet 接收

**关键数据结构** (`include/linux/if_vlan.h`):

```c
struct vlan_dev_priv {
    unsigned int        vlan_id;           // VLAN ID (0-4095)
    unsigned short      vlan_proto;        // ETH_P_8021Q (0x8100)
    struct net_device  *real_dev;         // 指向物理设备
    struct VLAN_PCS     *vlan_pcs;
    struct vlan_prio_tc2queue_entry prio_tc_map[8021Q_PRIO_MAP_SIZE];
};
```

**VLAN net_device 的接收回调** (`drivers/net/vlan.c`):

```c
static rx_handler_result_t vlan_rx_handler(struct sk_buff **pskb)
{
    struct vlan_dev_priv *vlan = vlan_dev_priv(skb->dev);
    struct vlan_hdr *vhdr;

    // 从 skb->mac_header 提取 VLAN tag
    vhdr = (struct vlan_hdr *)(skb->data - VLAN_HLEN);

    // 检查 VLAN ID 是否匹配
    if (vhdr->h_vlan_TCI & VLAN_VID_MASK) {
        __u16 vid = vhdr->h_vlan_TCI & VLAN_VID_MASK;

        if (vid != vlan->vlan_id) {
            // VLAN ID 不匹配，检查是否有其他 VLAN device 订阅
            return RX_HANDLER_PASS;  // 交给其他 handler
        }
    }

    // 匹配：移除 VLAN tag，还原原始 EtherType
    skb->protocol = vhdr->h_vlan_proto;
    __skb_pull(skb, VLAN_HLEN);
    skb->vlan_tci = 0;

    // 设置 real_dev 为接收设备
    skb->dev = vlan->real_dev;

    // 交给上层协议栈
    return RX_HANDLER_ANOTHER;
}
```

**VLAN filter 机制** (`net/core/dev.c`):

```c
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc)
{
    struct packet_type *ptype, *pt_prev;
    rx_handler_result_t rx_res;

    // 调用 RX handlers 链
    rx_res = __netif_receive_skb_core_sublist(skb, &pt_prev);

    // 如果返回 PASS，skb 会被传递给normal 协议栈
}
```

**关键点**：
1. **VLAN net_device 通过 `netdev_rx_handler_register` 注册接收回调**
2. **当物理设备收到 VLAN-tagged packet 时，遍历所有注册的 handler**
3. **每个 VLAN device 检查 VLAN ID，匹配则处理，不匹配则 PASS**
4. **真正的分发发生在 NIC 驱动层，而不是协议栈**

### 3.3 TX 方向 — VLAN Tag 插入

**VLAN device 的发送函数** (`drivers/net/vlan.c`):

```c
static netdev_tx_t vlan_dev_hard_start_xmit(struct sk_buff *skb,
                                            struct net_device *dev)
{
    struct vlan_dev_priv *vlan = vlan_dev_priv(dev);
    struct vlan_hdr *vhdr;
    unsigned int len;

    // 添加 VLAN tag (4 bytes)
    if (skb_cow_head(skb, VLAN_HLEN) < 0) {
        kfree_skb(skb);
        return NETDEV_TX_OK;
    }

    // 压入 VLAN header
    vhdr = (struct vlan_hdr *)__skb_push(skb, VLAN_HLEN);

    // 填充 VLAN header
    vhdr->h_vlan_proto = htons(ETH_P_8021Q);
    vhdr->h_vlan_TCI = htons(vlan->vlan_id | vlan->priority);

    // 更新 protocol
    skb->protocol = htons(ETH_P_8021Q);

    // 发送到物理设备
    skb->dev = vlan->real_dev;
    return dev_queue_xmit(skb);  // 调用物理设备的 qdisc
}
```

---

## 4. 核心差异对比

### 4.1 架构差异

| 特性 | lwIP (SafeOS) | Linux |
|------|---------------|-------|
| **VLAN netif 类型** | 独立的 `struct netif`，与物理 netif 平级 | 虚拟 `struct net_device`，堆叠在物理设备上 |
| **VLAN 识别** | 依赖 IP 地址匹配 + 可选 VLAN ID hook | 通过 `netdev_rx_handler` 精确匹配 VLAN ID |
| **Tag 处理位置** | lwIP `ethernet_input`/`ethernet_output` | 专门的 VLAN net_device driver |
| **多个 VLAN** | 每个 VLAN 一个独立 netif | 每个 VLAN 一个独立 net_device |
| **MAC_VLAN_FILTER** | 可选 hook，默认禁用 | always enabled via rx_handler |

### 4.2 RX 分发流程对比

**lwIP 方式**：
```
Packet 到达物理 netif
    │
    ▼
ethernet_input(p, &vnet_if)
    │
    ├─► 解析 VLAN tag (如果存在)
    ├─► VLAN ID 检查 (如果 MAC_VLAN_FILTER 启用)
    │
    ▼
ip4_input(p, &vnet_if)
    │
    ├─► 检查 vnet_if.ip_addr
    │
    ├─► NETIF_FOREACH 遍历所有 netif
    │     ├─► vlan_if[0].ip_addr == dest_ip ?
    │     ├─► vlan_if[1].ip_addr == dest_ip ?
    │     └─► ...
    │
    ▼
Socket 接收
```

**Linux 方式**：
```
Packet 到达物理 NIC
    │
    ▼
NIC driver DMA 完成，中断
    │
    ▼
netif_rx(skb)
    │
    ▼
__netif_receive_skb_core()
    │
    ├─► packet_type 匹配 (IP, ARP 等)
    │
    ├─► RX handlers 链
    │     ├─► vlan_rx_handler (VLAN device 1)
    │     │     └─► VLAN ID 匹配？ → 剥离 VLAN tag，交给 VLAN device
    │     ├─► vlan_rx_handler (VLAN device 2)
    │     │     └─► VLAN ID 匹配？ → 剥离 VLAN tag，交给 VLAN device
    │     └─► ...
    │
    ▼
Socket 接收
```

### 4.3 TX VLAN tag 插入对比

**lwIP 方式**：
```
App send() on VLAN socket
    │
    ▼
ethernet_output(netif, p, ...)
    │
    ├─► LWIP_HOOK_VLAN_SET(netif, p, ...)
    │     └─► if (netif->vlanid != 0)
    │           return (pbuf->priority << 13) | netif->vlanid
    │
    ├─► pbuf_add_header(p, ETH_HDR + VLAN_HDR)
    ├─► 填充 VLAN header (TPID=0x8100, TCI=PCP|VID)
    │
    ▼
netif->linkoutput(netif, p)  // = low_level_output
    │
    ▼
physical_netif->linkoutput() → NIC DMA
```

**Linux 方式**：
```
App send() on VLAN socket
    │
    ▼
sock_sendmsg()
    │
    ▼
vlan_dev_hard_start_xmit(skb, dev)
    │
    ├─► skb_cow_head(skb, VLAN_HLEN)
    ├─► __skb_push(skb, VLAN_HLEN)
    ├─► 填充 VLAN header (h_vlan_proto=0x8100, h_vlan_TCI=VID|PCP)
    │
    ▼
dev_queue_xmit(skb->real_dev)  // 发送到物理设备
    │
    ▼
物理设备的 qdisc → NIC DMA
```

---

## 5. SafeOS lwIP VLAN 实现的关键限制

### 5.1 MAC_VLAN_FILTER 的陷阱

如果 `MAC_VLAN_FILTER=1` 且 `vnet_if.vlanid=0`：

```c
// ethernet.c:149
if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
    pbuf_free(p);  // 所有 VLAN packet 被丢弃！
    return ERR_OK;
}
```

**问题**：物理网口的 `vlanid=0`，任何 VID!=0 的 packet 都会匹配失败。

**解决方案**：
1. **禁用 MAC_VLAN_FILTER**（最可能的选择）
2. **设置 vnet_if.vlanid 为特定值**（但只能接收一个 VLAN）
3. **修改 lwip_hook_vlan_check_fn 遍历所有 vlan_if**

### 5.2 IP 地址依赖性

lwIP 的 VLAN 分发完全依赖 IP 地址匹配：

```c
// ip4.c:607-618
#if !LWIP_SINGLE_NETIF
NETIF_FOREACH(netif) {
    if (netif == inp) continue;
    if (ip4_input_accept(netif)) {
        break;  // 找到 IP 匹配的 netif
    }
}
#endif
```

**限制**：
- 如果两个 VLAN interface 使用相同的 IP 子网，将无法正确分发
- 对于广播/多播 packet，分发行为可能不符合预期

### 5.3 AF_PACKET Socket 绑定问题

当 AF_PACKET socket 绑定到 VLAN netif 时：

```c
// raw.c:324-328
if ((pcb->netif_idx != AF_PACKET_NOBIND) && (NULL != inp)) {
    if (pcb->netif_idx != netif_get_index(inp)) {
        skip_this_frame = 1;  // inp 是 vnet_if，永不匹配！
    }
}
```

**问题**：传入 `ethernet_input` 的 `inp` 始终是 `vnet_if`，所以绑定到 VLAN netif 的 AF_PACKET socket 永远收不到 packet。

---

## 5.4 LWIP_ARP_FILTER_NETIF — 真正的 VLAN 感知分发机制

**关键发现**：`ethernet_input` 中有一层 **VLAN 感知的 netif 选择机制** (`LWIP_ARP_FILTER_NETIF=1`):

```c
// ethernet.c:121-127
#if LWIP_ARP_FILTER_NETIF
  netif = LWIP_ARP_FILTER_NETIF_FN(p, netif, lwip_htons(type));
  if(NULL == netif)
  {
    goto free_and_return;
  }
#endif
```

**`lwip_arp_filter_netif_fn`** (ethernet.c:459-517) 的 VLAN 处理逻辑：

```c
switch (type) {
    case ETHTYPE_VLAN: {
        // 遍历所有 netif，通过 VLAN ID 匹配
        NETIF_FOREACH(netif) {
            if (netif_is_up(netif)) {
                struct eth_vlan_hdr *vlan_hdr = ...;
                u16_t vid = netif->vlanid & VLAN_ID_MASK;
                if (vid == (vlan_hdr->prio_vid & VLAN_ID_MASK)) {
                    return netif;  // 返回匹配 VID 的 netif
                }
            }
        }
        return NULL;  // 无匹配
    }
    case ETHTYPE_ARP:
    case ETHTYPE_IP:
        // 仅匹配 vlanid == 0 的 netif
        NETIF_FOREACH(netif) {
            if (netif_is_up(netif) && ip4_addr_cmp(&dst, &netif->ip_addr) && netif->vlanid == 0u) {
                return netif;
            }
        }
        break;
}
```

**完整 RX 流程** (修正后的理解):

```
物理网卡 DMA
    │
    ▼
rx_callback() [main.c:4781]
    │
    ▼
vnet_if.input(p, &vnet_if)  ──────► ethernet_input() [ethernet.c:89]
    │                                    │
    │                                    ├─► ethhdr = p->payload
    │                                    ├─► type = ethhdr->type
    │                                    │
    │                                    ├─► #if LWIP_ARP_FILTER_NETIF (line 121)
    │                                    │     netif = lwip_arp_filter_netif_fn(p, netif, type)
    │                                    │     │
    │                                    │     └─► switch (type) {
    │                                    │           case ETHTYPE_VLAN:
    │                                    │             NETIF_FOREACH 遍历
    │                                    │             找 netif->vlanid == packet VID
    │                                    │             return 匹配的 vlan_if[i]
    │                                    │           case ETHTYPE_IP/ARP:
    │                                    │             NETIF_FOREACH 遍历
    │                                    │             找 IP 匹配 && vlanid == 0
    │                                    │         }
    │                                    │     if (netif == NULL) drop
    │                                    │
    │                                    ├─► #if ETHARP_SUPPORT_VLAN (line 133)
    │                                    │     if (type == ETHTYPE_VLAN) {
    │                                    │       解析 VLAN Header
    │                                    │       #ifdef MAC_VLAN_FILTER
    │                                    │         LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)
    │                                    │       #endif
    │                                    │       type = vlan->tpid
    │                                    │       p->priority = PCP
    │                                    │     }
    │                                    │
    │                                    └─► ip4_input(p, netif)  // netif 已是正确的 vlan_if[i]!
```

**关键点**：
1. **`LWIP_ARP_FILTER_NETIF=1`** 是 VLAN 分发的关键机制
2. 对于 **VLAN-tagged packet**，通过 **VLAN ID 匹配**找到正确的 vlan_if[i]
3. 对于 **非 VLAN packet**，通过 **IP 地址匹配 + vlanid==0** 找到 vnet_if
4. **`MAC_VLAN_FILTER`** 是额外的安全检查（两层机制）

---

## 5.5 VIRT_BRG_SUPPORT 模式下 VLAN 与桥接的交互

**架构**：

```
TX 路径 (VLAN netif → 物理网口):
vlan_if[i].linkoutput = low_level_output
    │
    └─► physical_netif->linkoutput = ethif_link_output_overload
            │
            ├─► vbridge_port.port_input() ──► Bridge 处理 (MAC 学习)
            └─► ethif_link_output() ──► NIC DMA

RX 路径 (物理网口 → Bridge + lwIP):
rx_callback()
    │
    ├─► vbridge_port.port_input() ──► Bridge 处理 (转发决策)
    └─► vnet_if.input() = ethernet_input ──► VLAN 解析 → lwIP
```

**关键发现**：

1. **Bridge 不理解 VLAN**：
   - `bridge_port_input()` 只是将 raw bytes 复制到共享内存
   - 发送给 hypervisor 进行 MAC 地址学习
   - 不解析 VLAN tag，MAC 地址偏移计算会出错！

2. **VLAN tag 保留在 packet 中**：
   - `vbridge_port_output()` 发送时保留原始 packet 格式
   - 包括 VLAN tag (如果存在)

3. **潜在问题**：
   - Bridge 的 FDB 学习会受到 VLAN tag 影响
   - 如果 packet 有 VLAN tag，MAC dst/src 位置会偏移 4 字节
   - Bridge 可能学习到错误的 MAC 地址

**示例问题**：

```
VLAN-tagged packet:
┌─────────────────────────────────────────────────────┬──────────────────┐
│ Ethernet Header (14 bytes)                          │ VLAN Tag (4B)   │
├──────┬───────────────────┬──────────────────────────┼──────────────────┤
│ DST  │ SRC               │ TPID=0x8100│TCI        │ Payload...       │
│ (6B) │ (6B)              ├─────────────┼───────────┼──────────────────┤
└──────┴───────────────────┴─────────────┴───────────┴──────────────────┘
                               ▲
                               └── Bridge 认为这里开始是 MAC src！
```

---

## 6. 总结

### lwIP (SafeOS) 架构
```
┌──────────────────────────────────────────────────────┐
│           VLAN netif (vlan_if[i])                   │
│   - 独立 IP 地址                                      │
│   - input = tcpip_input (从未被调用)                  │
│   - linkoutput = low_level_output → 物理网口          │
└──────────────────────────────────────────────────────┘
                         ▲
                         │ (通过 IP 地址匹配被发现)
                         │
┌──────────────────────────────────────────────────────┐
│           Physical netif (vnet_if)                    │
│   - 接收所有 packet                                   │
│   - input = ethernet_input                            │
│   - VLAN ID 检查 (如果 MAC_VLAN_FILTER 启用)          │
└──────────────────────────────────────────────────────┘
```

### Linux 架构
```
┌──────────────────────────────────────────────────────┐
│           VLAN net_device (eth0.100)                 │
│   - 注册 rx_handler，精确匹配 VLAN ID                  │
│   - 剥离 VLAN tag 后传递给协议栈                       │
└──────────────────────────────────────────────────────┘
                         ▲
                         │ (通过 rx_handler 链)
                         │
┌──────────────────────────────────────────────────────┐
│           Physical net_device (eth0)                  │
│   - 接收所有 packet，包含 VLAN tag                    │
│   - 遍历所有注册的 rx_handlers                        │
└──────────────────────────────────────────────────────┘
```

### 关键结论

1. **lwIP 的 VLAN 分发有两层机制**：
   - `LWIP_ARP_FILTER_NETIF=1`：通过 VLAN ID 匹配找到正确的 vlan_if[i]
   - `ip4_input_accept`：通过 IP 地址匹配作为最终验证
2. **Linux 的 VLAN 分发是基于 netdevice 和 rx_handler 的**
3. **AF_PACKET socket 绑定到 VLAN netif 无法工作**（inp 始终是 vnet_if）
4. **VIRT_BRG_SUPPORT 模式下 Bridge 不理解 VLAN tag**，可能导致 FDB 学习问题
5. **SafeOS 的实现是 IP 地址 + VLAN ID 的混合模型**，与 Linux 的纯 VLAN 模型不同

---

## 7. 参考代码路径

### SafeOS lwIP
- `os-framework/servers/net/src/vlanif.c` — VLAN netif 创建和管理
- `os-framework/servers/net/src/main.c` — 物理网口初始化
- `external/lwip_ds_mcu/src/netif/ethernet.c` — VLAN tag 解析/插入，`LWIP_ARP_FILTER_NETIF` 实现
- `external/lwip_ds_mcu/src/core/ipv4/ip4.c` — IP 层 netif 匹配
- `external/lwip_ds_mcu/src/netif/bridgeif.c` — lwIP bridgeif 实现
- `os-framework/servers/net/src/bridge.c` — VIRT_BRG_SUPPORT bridge 集成

### Linux 内核
- `drivers/net/vlan.c` — VLAN net_device 实现
- `include/linux/if_vlan.h` — VLAN 数据结构
- `net/core/dev.c` — RX handler 机制
- `net/8021q/vlan_core.c` — VLAN 核心功能
