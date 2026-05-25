# netif 结构分析 — T-010

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: struct netif 所有字段详解、client_data 机制、netif_list/netif_default 链表管理

---

## 1. 概述

### 1.1 netif 在架构中的位置

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Application                                 │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Socket API                                   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        lwIP Protocol Stack                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    struct netif                               │   │
│  │   vnet_if (物理网口)                                        │   │
│  │   vlan_if[0] (VLAN 100)                                    │   │
│  │   vlan_if[1] (VLAN 200)                                    │   │
│  │   ...                                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DMA / elem_ring                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. struct netif 详解

**文件**: `external/lwip_ds_mcu/src/include/lwip/netif.h:297`

### 2.1 链表结构

```c
struct netif {
#if !LWIP_SINGLE_NETIF
  struct netif *next;  // 指向下一个 netif
#endif
```

### 2.2 IPv4 配置

```c
#if LWIP_IPV4
  ip_addr_t ip_addr;   // IP 地址 (网络字节序)
  ip_addr_t netmask;   // 子网掩码
  ip_addr_t gw;        // 默认网关
#endif
```

### 2.3 回调函数 (核心!)

```c
  /** 接收数据包: 由 NIC driver 调用，传入 TCP/IP 栈 */
  netif_input_fn input;

#if LWIP_IPV4
  /** 输出函数: IP 模块调用，发送数据包 (通常 etharp_output) */
  netif_output_fn output;
#endif

  /** 链路层输出: ethernet_output 调用，发送原始帧 (通常是 ethif_link_output) */
  netif_linkoutput_fn linkoutput;

#if LWIP_RAW_ETH
  /** 底层驱动操作 */
  netif_drv_op_fn drv_op;
#endif
```

### 2.4 回调函数类型定义

```c
// netif.h 中定义的函数指针类型
typedef err_t (*netif_input_fn)(struct pbuf *p, struct netif *inp);
typedef err_t (*netif_output_fn)(struct netif *netif, struct pbuf *p, const ip4_addr_t *ipaddr);
typedef err_t (*netif_linkoutput_fn)(struct netif *netif, struct pbuf *p);
```

### 2.5 状态和回调

```c
  void *state;  // 设备特定状态 (由驱动使用)

#ifdef netif_get_client_data
  void* client_data[LWIP_NETIF_CLIENT_DATA_INDEX_MAX + LWIP_NUM_NETIF_CLIENT_DATA];
#endif

#if LWIP_NETIF_STATUS_CALLBACK
  netif_status_callback_fn status_callback;  // 状态变化回调
#endif
#if LWIP_NETIF_LINK_CALLBACK
  netif_status_callback_fn link_callback;    // 链路变化回调
#endif
```

### 2.6 网络属性

```c
  u16_t mtu;           // 最大传输单元 (通常 1500)
  u8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
  u8_t hwaddr_len;     // MAC 地址长度 (通常 6)
  u8_t flags;          // 标志位 (@ref netif_flags)
  char name[2];        // 接口名缩写 (如 "et", "vl")
  char *fullname;      // 接口全名 (如 "eth0", "vlan100")
  u8_t num;            // 接口编号 (用于 ifindex)
```

### 2.7 VLAN 支持

```c
#if ETHARP_SUPPORT_VLAN
  u16_t vlanid;        // VLAN ID (0 = 无 VLAN)
#endif
```

### 2.8 统计

```c
  struct netif_pkt_stats stats;  // 包统计
```

---

## 3. netif_list 链表管理

### 3.1 全局变量

```c
// netif.h:462
extern struct netif *netif_list;      // 所有 netif 的链表
extern struct netif *netif_default;   // 默认 netif
```

### 3.2 NETIF_FOREACH 宏

```c
// netif.h:463
#if LWIP_SINGLE_NETIF
#define NETIF_FOREACH(netif) if (((netif) = netif_default) != NULL)
#else
#define NETIF_FOREACH(netif) for ((netif) = netif_list; (netif) != NULL; (netif) = (netif)->next)
#endif
```

### 3.3 netif_add 添加流程

**文件**: `external/lwip_ds_mcu/src/core/netif.c:281`

```c
struct netif *netif_add(
    struct netif *netif,
    const ip4_addr_t *ipaddr,
    const ip4_addr_t *netmask,
    const ip4_addr_t *gw,
    void *state,
    netif_init_fn init,
    netif_input_fn input)
{
    // 1. 初始化字段
    netif->state = state;
    netif->num = netif_num++;
    netif->input = input;

    // 2. 调用驱动初始化函数
    init(netif);  // 如 init_ethif()

    // 3. 添加到链表头部
#if !LWIP_SINGLE_NETIF
    netif->next = netif_list;
    netif_list = netif;
#endif

    return netif;
}
```

### 3.4 链表结构

```
netif_list (链表头)
      │
      ▼
┌─────────────┐    next     ┌─────────────┐    next     ┌─────────────┐
│   vnet_if   │ ─────────► │  vlan_if[0] │ ─────────► │  vlan_if[1] │
│ (物理网口)   │            │  (VLAN 100) │            │  (VLAN 200) │
└─────────────┘            └─────────────┘            └─────────────┘
```

---

## 4. netif_num 编号机制

### 4.1 自动分配

```c
// netif.c:396-418
{
    struct netif *netif2;
    int num_netifs;
    do {
        if (netif->num == 255) {
            netif->num = 0;
        }
        num_netifs = 0;
        for (netif2 = netif_list; netif2 != NULL; netif2 = netif2->next) {
            if (netif2->num == netif->num) {
                netif->num++;
                break;
            }
            num_netifs++;
        }
    } while (netif2 != NULL);
}
```

### 4.2 编号用途

- `netif_get_index(netif)` 返回 `num + 1` (避免 0)
- AF_PACKET socket 绑定使用 `netif_idx`

---

## 5. SafeOS 中的 netif 实例

### 5.1 vnet_if (物理网口)

**main.c:6441**:

```c
struct netif *netif = netif_add(&vnet_if,
    &addr, &netmask, &gw,  // IP: 172.20.0.1
    NULL,                    // state = NULL
    init_ethif,             // 初始化函数
    ethernet_input);         // input = ethernet_input

// vnet_if 字段:
// .name = "et"
// .num = 0
// .vlanid = 0  (无 VLAN)
// .input = ethernet_input
// .linkoutput = ethif_link_output
// .output = etharp_output
```

### 5.2 vlan_if[i] (VLAN 网口)

**vlanif.c:231-237**:

```c
netif = netif_add(&vlan_if[vlan_cnt],
    IP4_OR_NULL(addr),      // IP: 172.20.100.1
    IP4_OR_NULL(addr),
    IP4_OR_NULL(addr),
    &conf_idx,              // state = 配置索引
    vlanif_init,            // 初始化函数
    tcpip_input);           // input = tcpip_input

// vlan_if 字段:
// .name = "vl"
// .num = 1, 2, ...
// .vlanid = 100, 200, ...  (VLAN ID)
// .input = tcpip_input
// .linkoutput = low_level_output
// .output = etharp_output
```

---

## 6. input 函数分发机制

### 6.1 RX 流程

```
NIC DMA → rx_callback → vnet_if.input(p, &vnet_if)
                           │
                           ▼
                      ethernet_input(p, &vnet_if)
                           │
                           ├─► LWIP_ARP_FILTER_NETIF
                           │     └─► 选择正确的 netif (vnet_if 或 vlan_if[i])
                           │
                           ▼
                      ip4_input(p, netif)
```

### 6.2 netif_input 调用链

```
ethernet_input(p, netif)
    │
    ▼
ip4_input(p, netif)  // netif 已被 LWIP_ARP_FILTER_NETIF 选择
    │
    ▼
udp_input(p, netif) / tcp_input(p, netif)
    │
    ▼
socket 接收队列
```

### 6.3 input 函数对比

| netif | input 函数 | 说明 |
|-------|-----------|------|
| **vnet_if** | `ethernet_input` | 物理网口，直接接收所有 packet |
| **vlan_if[i]** | `tcpip_input` | VLAN 网口，从未被直接调用 |

---

## 7. client_data 机制

### 7.1 定义

```c
#ifdef netif_get_client_data
  void* client_data[LWIP_NETIF_CLIENT_DATA_INDEX_MAX + LWIP_NUM_NETIF_CLIENT_DATA];
#endif
```

### 7.2 用途

允许在 netif 上存储任意的客户端数据，常用于：
- Bridge: 存储 port 信息 (`bridgeif_port_t`)
- 其他驱动特定数据

### 7.2 使用示例

```c
// bridgeif.c
bridgeif_netif_client_id = netif_alloc_client_data_id();
port = netif_get_client_data(netif, bridgeif_netif_client_id);
```

---

## 8. 关键标志位

```c
// netif_flags 定义
NETIF_FLAG_BROADCAST    // 支持广播
NETIF_FLAG_ETHARP       // 支持 ARP
NETIF_FLAG_LINK_UP      // 链路up
NETIF_FLAG_IGMP         // 支持 IGMP
NETIF_FLAG_ETHERNET     // 以太网
```

---

## 9. 总结

### 9.1 netif 是 lwIP 的核心抽象

```
┌─────────────────────────────────────────────────────────────────────┐
│                            netif                                     │
├─────────────────────────────────────────────────────────────────────┤
│  身份:   name[2] + num → 接口标识                                  │
│  配置:   ip_addr + netmask + gw → IP 层配置                       │
│  收发:   input + output + linkoutput →  packet 收发                 │
│  状态:   state + flags + vlanid →  接口状态                        │
│  统计:   stats → 计数器                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 SafeOS 中的 netif 链表

```
vnet_if (物理网口, vlanid=0, IP=172.20.0.1)
    ↓ next
vlan_if[0] (VLAN 100, vlanid=100, IP=172.20.100.1)
    ↓ next
vlan_if[1] (VLAN 200, vlanid=200, IP=172.20.200.1)
    ↓ next
NULL
```

### 9.3 关键设计

1. **input 回调**: 是 packet 进入协议栈的入口
2. **linkoutput 回调**: 是 packet 发送到链路的出口
3. **vlanid 字段**: lwIP 的 VLAN 支持，通过此字段区分 VLAN
4. **链表组织**: 所有 netif 通过 `next` 指针串联
