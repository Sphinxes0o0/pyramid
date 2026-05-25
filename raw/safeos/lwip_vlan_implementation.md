# SafeOS lwIP VLAN 实现深度分析

> 文档版本: 1.0
> 更新日期: 2026/04/22
> 代码路径: `/home/shiyang/nio/nt35/safeos/`

---

## 1. 概述

SafeOS 中 lwIP 的 VLAN 支持基于 **IEEE 802.1Q** 标准，实现了两层功能：

| 层次 | 功能 | 代码位置 |
|------|------|----------|
| **lwIP 核心** | 802.1Q VLAN Tag 解析/插入、VLAN ID 检查 | `external/lwip_ds_mcu/src/netif/ethernet.c`、`external/lwip_ds_mcu/src/core/ipv4/etharp.c` |
| **VLAN 接口** | 虚拟 VLAN netif 创建、配置、路由 | `os-framework/servers/net/src/vlanif.c` |
| **QoS VLAN** | 802.1Q PCP 优先级处理 | `os-framework/servers/net/src/qos.c` |
| **配置解析** | YAML 配置解析、静态 ARP 表 | `os-framework/servers/net/src/conf_parser.c` |

---

## 2. IEEE 802.1Q VLAN 数据结构

### 2.1 VLAN Tag 结构

```
Ethernet Header (14 bytes)
┌─────────────────────────────────────┬──────────────────────────────┐
│ Destination MAC (6B)               │ Source MAC (6B)              │
├─────────────────────────────────────┴──────────────────────────────┤
│ TPID (2B) = 0x8100                   │ TCI (2B)                    │
├─────────────────────────────────────┼──────────────────────────────┤
│ PCP (3 bits) │ DEI (1 bit) │ VID (12 bits)                       │
└─────────────────────────────────────┴──────────────────────────────┘
```

### 2.2 lwIP VLAN 数据结构

**struct eth_vlan_hdr** (`lwip/prot/ethernet.h:121`):

```c
struct eth_vlan_hdr {
  u16_t prio_vid;   // Bit[15:13] = PCP (Priority Code Point)
                    // Bit[12]    = DEI (Drop Eligibility Indicator)
                    // Bit[11:0]  = VID (VLAN Identifier)
  u16_t tpid;       // Tag Protocol Identifier (= 0x8100 for 802.1Q)
};

#define SIZEOF_VLAN_HDR  4
#define VLAN_ID(vlan_hdr) (lwip_htons((vlan_hdr)->prio_vid) & 0xFFF)  // 取低12位 VID
#define VLAN_ID_MASK     0x0FFF
```

**struct netif** (`lwip/netif.h:441`):

```c
struct netif {
    // ... 其他字段 ...
#if ETHARP_SUPPORT_VLAN
    u16_t vlanid;    // 本 netif 所属的 VLAN ID (0 = 不属于任何 VLAN)
#endif
    // ... 其他字段 ...
};
```

### 2.3 pbuf priority 字段 — VLAN PCP

```c
struct pbuf {
    // ... 其他字段 ...
    u8_t priority;    // buffer priority (0-7), 对应 802.1Q PCP
};
```

**PCP (Priority Code Point) 映射** (`etharp.c:438`):
```c
vlan_id = (pbuf->priority << 13) | netif->vlanid;
// PCP 占高 3 位，VID 占低 12 位
```

---

## 3. lwIP 核心 VLAN 支持

### 3.1 编译配置

**默认启用** (`external/lwip_ds_mcu/src/include/lwip/opt.h:723`):
```c
#define ETHARP_SUPPORT_VLAN             1   // 默认开启
```

### 3.2 RX 路径 — VLAN Tag 解析

**ethernet_input()** (`ethernet.c:133-169`):

```
ethernet_input(pbuf *p, netif *netif)
  │
  ├─► ethhdr = (struct eth_hdr *)p->payload
  │
  ├─► type = ethhdr->type
  │
  ├─► #if ETHARP_SUPPORT_VLAN
  │     if (type == ETHTYPE_VLAN (0x8100)) {
  │       vlan = (struct eth_vlan_hdr *)(p->payload + SIZEOF_ETH_HDR)
  │       next_hdr_offset = SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR  // = 18 bytes
  │       │
  │       ├─► #ifdef MAC_VLAN_FILTER
  │       │     ├─► LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)
  │       │     │     └─► lwip_hook_vlan_check_fn() — 见下方详解
  │       │     └─► 返回 0 → pbuf_free(p); return ERR_OK (丢弃)
  │       │     #endif
  │       │
  │       ├─► type = vlan->tpid  // VLAN 头后真正的 EtherType
  │       │
  │       └─► p->priority = (vlan->prio_vid >> 13)  // 提取 PCP 到 pbuf
  │     }
  │   #endif
  │
  └─► switch (type) {
          ETHTYPE_IP  → ip4_input()
          ETHTYPE_ARP → etharp_input()
      }
```

### 3.3 lwip_hook_vlan_check_fn — VLAN ID 检查

**文件**: `ethernet.c:398-421`

```c
int lwip_hook_vlan_check_fn(void *netif, void *eth_hdr, void *vlan_hdr)
{
    struct netif  *_netif  = (struct netif  *)netif;
    struct eth_hdr *_eth_hdr = (struct eth_hdr *)eth_hdr;
    struct eth_vlan_hdr *_vlan_hdr = (struct eth_vlan_hdr *)vlan_hdr;

    // 确保是 VLAN 包
    if (_eth_hdr->type != PP_HTONS(ETHTYPE_VLAN)) {
        return 0;  // 不是 VLAN 包，不匹配
    }

    // 确保 netif 已启用
    if (netif_is_up(_netif)) {
        // 提取本 netif 的 VLAN ID
        u16_t vid = PP_HTONS(_netif->vlanid) & PP_HTONS(VLAN_ID_MASK);
        // 提取包中的 VLAN ID
        u16_t pkt_vid = _vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK);

        if (vid == pkt_vid) {
            // VLAN ID 匹配，接受包
            return 1;
        }
    }

    return 0;  // VLAN ID 不匹配，丢弃
}
```

**调用链**:
```
ethernet_input()
  └─► #ifdef MAC_VLAN_FILTER
        if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
          pbuf_free(p); return ERR_OK;  // 丢弃
        }
```

### 3.4 TX 路径 — VLAN Tag 插入

**ethernet_output()** (`ethernet.c:339-355`):

```c
#if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
s32_t vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
if (vlan_prio_vid >= 0) {
    // 需要插入 VLAN Tag
    pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);  // 增加 4 字节
    vlanhdr = (struct eth_vlan_hdr *)(((u8_t *)p->payload) + SIZEOF_ETH_HDR);
    vlanhdr->tpid     = eth_type_be;           // 保存原始 EtherType
    vlanhdr->prio_vid = lwip_htons((u16_t)vlan_prio_vid);  // PCP + VID
    eth_type_be = PP_HTONS(ETHTYPE_VLAN);     // 改为 VLAN EtherType
}
#endif
```

### 3.5 lwip_hook_vlan_set_fn — VLAN Tag 设置

**文件**: `ethernet.c:423-441`

```c
int lwip_hook_vlan_set_fn(void *netif, void *pbuf,
                          const void *src, const void *dst, u16_t eth_type)
{
    struct netif  *_netif  = (struct netif  *)netif;
    struct pbuf *_pbuf = (struct pbuf *)pbuf;

    /* vlanid == 0 (NO_VLANID) 表示此 netif 不带 VLAN Tag */
    if (_netif->vlanid == NO_VLANID) {
        return -1;  // 不插入 VLAN Tag
    }

    // 组合 PCP (高 3 位) + VID (低 12 位)
    u16_t vlan_id = _netif->vlanid;
    vlan_id = ((_pbuf->priority << 13) | vlan_id);  // pbuf->priority 提供 PCP

    return vlan_id;  // 返回 >= 0，触发 VLAN Tag 插入
}
```

### 3.6 多 VLAN 播送 (Multicast VLAN)

**ethernet_output_multicast_vlan()** (`etharp.c:522-570`):

当 netif 配置了多个 VLAN 时 (通过 `lwip_hook_get_vlan_count` 和 `lwip_hook_get_vlan` 获取)，单播包的输出函数会为每个 VLAN 复制一份并插入对应的 VLAN Tag。

---

## 4. VLAN 接口层 (vlanif.c)

### 4.1 VLAN netif 的创建

```
vlanif_setup()                                [vlanif.c:223]
  │
  ├─► vlanif_find_next_conf()              // 从 YAML 配置中查找下一个 VLAN 条目
  │
  ├─► netif_add(&vlan_if[vlan_cnt],
  │       IP4_OR_NULL(addr),
  │       IP4_OR_NULL(addr),
  │       IP4_OR_NULL(addr),
  │       &conf_idx,                       // netif->state 指向配置索引
  │       vlanif_init,                     // 初始化函数
  │       tcpip_input)                     // input 函数
  │
  └─► netif_set_link_up() / netif_set_up()
```

### 4.2 vlanif_init — VLAN netif 初始化

**文件**: `vlanif.c:93-150`

```c
err_t vlanif_init(struct netif *netif)
{
    // 从 netif->state 获取配置索引
    int conf_idx = *((int *)netif->state);

    // 设置 netif 名称 (如 "vl0", "vl1")
    netif->name[0] = vlan_conf[conf_idx].ifName[0];
    netif->name[1] = vlan_conf[conf_idx].ifName[1];

    // 设置 output 函数 (IP 层输出)
    netif->output = etharp_output;

    // 关键: 设置 linkoutput，指向物理网卡的 low_level_output
    netif->linkoutput = low_level_output;

    // 设置 VLAN ID
    netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10);

    // 设置 netif flags
    netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP |
                   NETIF_FLAG_LINK_UP | NETIF_FLAG_IGMP | NETIF_FLAG_ETHERNET;

    // 设置 IP 地址
    inet_aton(vlan_conf[conf_idx].ipAddr, &ip_inet);
    inet_aton(vlan_conf[conf_idx].netMask, &nm_inet);
    netif_set_addr(netif, &ip_addr, &netmask, NULL);

    return ERR_OK;
}
```

### 4.3 low_level_output — VLAN → 物理网卡

```c
static err_t low_level_output(struct netif *netif, struct pbuf *p)
{
    if (physical_netif != NULL) {
        // 所有 VLAN 包的输出都经过物理网卡
        // VLAN Tag 的插入由 lwip_hook_vlan_set_fn() 完成
        physical_netif->linkoutput(physical_netif, p);
    }
    return ERR_OK;
}
```

### 4.4 VLAN 与物理网卡的对应关系

```
┌─────────────────────────────────────────────────────────────┐
│  vnet_if (物理网卡, netif->vlanid = 0)                      │
│  - name = "ei"                                              │
│  - linkoutput = ethif_link_output                            │
│  - 处理不带 VLAN Tag 的以太网帧                               │
└─────────────────────────────────────────────────────────────┘
                              ↑
                              │ VLAN 包 (已插入 VLAN Tag)
                              │
┌─────────────────────────────────────────────────────────────┐
│  vlan_if[0] (VLAN 网卡, netif->vlanid = 100)               │
│  - name = "vl0"                                              │
│  - linkoutput = low_level_output                             │
│  - 所有包输出到 physical_netif (vnet_if)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. 配置方式

### 5.1 VLAN 配置结构 (YAML)

**vlan_conf_t** (`conf_parser.h:28-41`):

```c
typedef struct {
    char* ifName;       // VLAN 接口名 (如 "vl0")
    char* ifType;        // 接口类型 ("Ether" 或 "VLAN")
    char* description;  // 描述
    char* hwAddr;        // MAC 地址
    char* vid;          // VLAN ID (字符串，如 "100")
    char* linkspeed;    // 链路速度
    char* mtu;          // MTU
    char* lowerIfs;     // 下层物理接口
    char* ipAddr;       // IP 地址
    char* netMask;      // 子网掩码
    char* forwarding;   // 是否启用转发 ("true"/"false")
} vlan_conf_t;
```

**vlan_arp_conf_t** (`conf_parser.h:43-48`) — 静态 ARP 表:

```c
typedef struct {
    char* ifName;       // VLAN 接口名
    char* addr;         // IP 地址
    char* mac;         // MAC 地址
} vlan_arp_conf_t;
```

### 5.2 YAML 配置示例

```yaml
# net_conf.yaml (通过 CPIO 打包到镜像)
network:
  ethernet:
    - ifName: "PFE"
      ifType: "Ether"
      hwAddr: "70:b3:d5:20:03:01"
      ipAddr: "172.20.0.2"
      netMask: "255.255.255.0"
      forwarding: "false"
      linkspeed: "0"
      mtu: "1500"

  vlan:
    - ifName: "vl0"
      ifType: "VLAN"
      vid: "100"
      lowerIfs: "PFE"
      ipAddr: "192.168.100.1"
      netMask: "255.255.255.0"
      forwarding: "false"
    - ifName: "vl1"
      ifType: "VLAN"
      vid: "200"
      lowerIfs: "PFE"
      ipAddr: "192.168.200.1"
      netMask: "255.255.255.0"
      forwarding: "false"

  arp_table:
    - ifName: "vl0"
      addr: "192.168.100.10"
      mac: "00:11:22:33:44:55"
```

### 5.3 配置解析流程

```
net_conf.yaml (CPIO 打包)
    │
    ▼
conf_parser.c: readconf_binary()
    │
    ├─► parse_ether_conf()       // 解析物理网卡配置
    ├─► parse_vlan_conf()        // 解析 VLAN 配置 → vlan_conf[]
    └─► parse_vlan_arp_conf()    // 解析静态 ARP → vlan_arp_conf[]

vlanif_setup()                   // 根据 vlan_conf[] 创建 VLAN netif
vlanif_update_arp_entry()       // 根据 vlan_arp_conf[] 添加静态 ARP 条目
```

### 5.4 网络配置初始化调用链

```
main()                                          [main.c]
  │
  ├─► init_ds_ring()
  ├─► create_nic_thread()
  ├─► tcpip_init()
  ├─► netif_add(&vnet_if, ...)                // 创建物理网卡 vnet_if
  │
  ├─► ethif_update(netif)                     // 从 YAML 配置物理网卡 IP
  │     └─ netif_set_addr(net_if, &ip_addr, &netmask, NULL)
  │
  ├─► vlanif_conf_init(vlan_conf_cnt)         // 分配 vlan_if[] 内存
  │
  ├─► vlanif_setup()                          // 创建 VLAN netif
  │     ├─ vlanif_find_next_conf()            // 查找 VLAN 配置
  │     ├─ netif_add(&vlan_if[i], vlanif_init, tcpip_input)
  │     └─ netif->vlanid = vid                // 设置 VLAN ID
  │
  ├─► vlanif_update_arp_entry()              // 添加 VLAN 静态 ARP 条目
  │     └─ etharp_add_static_entry(&ip_addr, &ethaddr)
```

---

## 6. VLAN PCP QoS 支持

### 6.1 pbuf priority 字段

`pbuf->priority` (0-7) 对应 802.1Q PCP 优先级：

| PCP 值 | 优先级 | 典型用途 |
|--------|--------|----------|
| 0 | Best Effort | 普通数据 |
| 1 | Background | 低优先级 |
| 2 | Excellent Effort | |
| 3 | Critical | |
| 4 | Video | < 100ms 延迟 |
| 5 | Voice | < 10ms 延迟 |
| 6 | Internetwork Control | |
| 7 | Network Control | 最高优先级 |

### 6.2 TX 路径: pbuf priority → VLAN PCP

```c
// lwip_hook_vlan_set_fn() 中 (ethernet.c:438)
vlan_id = (pbuf->priority << 13) | netif->vlanid;
// pbuf->priority 左移 13 位，插入 VLAN TCI 的 PCP 字段
```

### 6.3 RX 路径: VLAN PCP → pbuf priority

```c
// ethernet_input() 中 (ethernet.c:167)
p->priority = PP_HTONS(vlan->prio_vid) >> 13;
// 从 VLAN TCI 提取 PCP，存入 pbuf->priority 供上层使用
```

### 6.4 PFE VLAN Bridge (QoS)

**qos.c:setup_pfe_vlan_bridge()** (`qos.c:49-120`):

```c
#ifdef ENABLE_PFE_VLAN_BRIDGE
// PFE 硬件 VLAN Bridge 配置
for each VLAN (vid):
    demo_l2_bd_add(p_cl, &bd, vlanid);  // 添加 VLAN 到 L2 Bridge
    demo_l2_bd_get_by_vlan(p_cl, &bd, vlanid);  // 按 VLAN 查找 BD
```

### 6.5 QoS 队列优先级配置

```c
// qos.c: setup_qos_conf()
snprintf(sch0_que, SIZE_S, "que%s:0,", eqos_conf[count].priority);
// 将 VLAN priority 映射到 QoS 队列
```

---

## 7. 关键配置宏

### 7.1 lwIP VLAN 相关配置

| 宏 | 默认值 | 说明 |
|----|--------|------|
| `ETHARP_SUPPORT_VLAN` | 1 | 启用 VLAN 支持 |
| `MAC_VLAN_FILTER` | (未定义) | 启用 `LWIP_HOOK_VLAN_CHECK` |
| `ETHARP_VLAN_CHECK` | (未定义) | 固定检查特定 VLAN ID |
| `MULTICAST_SUPPORT_VLAN` | (未定义) | 启用多 VLAN 播送 |

### 7.2 NSv 平台配置

在 `os-framework/settings/easy-settings-*.cmake` 或 `nsv/gen_config.h` 中:

```c
#define ETHARP_SUPPORT_VLAN           1   // 必须在编译时启用
#define MAC_VLAN_FILTER              1   // 启用 VLAN ID 检查 Hook
```

---

## 8. 支持的功能和限制

### 8.1 支持的功能

| 功能 | 状态 | 说明 |
|------|------|------|
| VLAN Tag 解析 (RX) | ✅ | 支持 ETHTYPE_VLAN (0x8100) |
| VLAN Tag 插入 (TX) | ✅ | 通过 `LWIP_HOOK_VLAN_SET` |
| per-netif VLAN ID | ✅ | `netif->vlanid` 字段 |
| VLAN ID 过滤 (RX) | ✅ | `lwip_hook_vlan_check_fn()` |
| 802.1Q PCP 优先级 | ✅ | `pbuf->priority` ↔ VLAN TCI |
| 多 VLAN netif | ✅ | 物理网卡 + 多个 VLAN 网卡 |
| VLAN 静态 ARP 表 | ✅ | `etharp_add_static_entry()` |
| VLAN 转发 (IP forwarding) | ✅ | `netif_forward_enable()` |
| VLAN IGMP 支持 | ✅ | `NETIF_FLAG_IGMP` |

### 8.2 不支持的功能

| 功能 | 状态 | 说明 |
|------|------|------|
| QinQ (双重 VLAN Tag) | ❌ | 不支持嵌套 VLAN |
| VLAN 学习和 STP | ❌ | 由硬件/PFE 处理，lwIP 不涉及 |
| 动态 VLAN 注册 (GVRP) | ❌ | |
| IPv6 VLAN | ❌ | `LWIP_IPV6 = 0` |

---

## 9. 调用链总结

### 9.1 VLAN RX (接收) 路径

```
NIC 驱动
  │
  │ DMA → CMA buffer
  │
  ▼
nic_rx_thread() → rx_callback() → vnet_if.input()
  │
  ▼
ethernet_input(p, netif)                       [ethernet.c:89]
  │
  ├─► ETH_P_VLAN (0x8100) 判断
  │
  ├─► lwip_hook_vlan_check_fn(netif, ethhdr, vlan_hdr)
  │     └─► netif->vlanid == VLAN_ID(vlan_hdr) ?  // VLAN ID 匹配检查
  │
  ├─► p->priority = VLAN_PCP(vlan_hdr)          // 提取 PCP
  │
  ├─► type = vlan->tpid                          // 真正 EtherType
  │
  └─► ip4_input(p, netif) / etharp_input(p, netif)
```

### 9.2 VLAN TX (发送) 路径

```
App sendto(socket, buf, len, ...)
  │
  ▼
lwip_sendto() → ip4_output_if()
  │
  ▼
netif->output = etharp_output() → ethernet_output()
  │
  ├─► LWIP_HOOK_VLAN_SET(netif, pbuf, src, dst, eth_type)
  │     └─► lwip_hook_vlan_set_fn()
  │           ├─► netif->vlanid == NO_VLANID ?  // 不带 VLAN
  │           │     return -1;                   // 无 VLAN Tag
  │           └─► return (pbuf->priority << 13) | netif->vlanid
  │                 // 组合 PCP + VID，触发 VLAN Tag 插入
  │
  ▼
low_level_output(netif, pbuf)               [vlanif.c:52]
  │
  └─► physical_netif->linkoutput(physical_netif, pbuf)
        // 转发到物理网卡
```

---

## 10. 关键文件清单

| 文件 | 职责 |
|------|------|
| `external/lwip_ds_mcu/src/netif/ethernet.c` | VLAN Tag 解析/插入、`lwip_hook_vlan_check_fn`、`lwip_hook_vlan_set_fn` |
| `external/lwip_ds_mcu/src/core/ipv4/etharp.c` | VLAN multicast 输出、ARP + VLAN 集成 |
| `external/lwip_ds_mcu/src/include/lwip/prot/ethernet.h` | `struct eth_vlan_hdr`、`VLAN_ID` 宏 |
| `external/lwip_ds_mcu/src/include/lwip/netif.h` | `netif->vlanid`、`NO_VLANID` |
| `external/lwip_ds_mcu/src/include/lwip/pbuf.h` | `pbuf->priority` (PCP) |
| `os-framework/servers/net/src/vlanif.c` | VLAN netif 创建/初始化、low_level_output |
| `os-framework/servers/net/src/vlanif.h` | VLAN 接口声明 |
| `os-framework/servers/net/src/conf_parser.c` | YAML 配置解析 (vlan_conf、vlan_arp_conf) |
| `os-framework/servers/net/src/conf_parser.h` | VLAN 配置结构体定义 |
| `os-framework/servers/net/src/qos.c` | PFE VLAN Bridge、QoS 优先级 |
| `os-framework/servers/net/src/ifconfig.c` | `ifconfig` 显示 VLAN 信息 (vlanid) |
| `libs/util_libs/liblwip/default_opts/lwipopts.h` | lwIP 编译选项 |
