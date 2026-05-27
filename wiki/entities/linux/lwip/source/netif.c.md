---
type: entity
tags: [lwip, netif, source, network-interface]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# netif.c — Network Interface Management

> netif：接口注册/删除、地址管理、loopback、up/down、callback

## 文件概览

| 属性 | 值 |
|------|-----|
| 路径 | `src/core/netif.c` |
| 行数 | 1913 |
| 功能 | netif 抽象层：添加删除、IP 地址管理、up/down 状态、loopback |
| 依赖 | ip, tcp, udp, raw, igmp, ethernet |

## 函数索引

### 初始化
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_init` | 182 | 模块初始化 (创建 loopback netif) |
| `netif_input` | 222 | 入口分发 (ethernet_input 或 ip_input) |

### netif 管理
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_add` | 281 | 添加 netif 到系统 (IP/mask/gw/init/input) |
| `netif_add_noaddr` | 244 | 添加 netif (无地址) |
| `netif_remove` | 768 | 从系统移除 netif |
| `netif_find` | 1769 | 按名称查找 netif (如 "en0") |
| `netif_find_without_num` | 1801 | 按名称查找 (不含数字) |
| `netif_find_fullname` | 1822 | 按完整名称查找 |
| `netif_get_by_index` | 1744 | 按索引查找 |
| `netif_name_to_index` | 1705 | 名称 → 索引 |
| `netif_index_to_name` | 1724 | 索引 → 名称 |

### 地址管理
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_set_addr` | 668 | 设置 IP/mask/gw (统一接口) |
| `netif_set_ipaddr` | 510 | 设置 IPv4 地址 |
| `netif_set_netmask` | 570 | 设置子网掩码 |
| `netif_set_gw` | 631 | 设置网关 |

### 状态管理
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_set_up` | 875 | bring up netif |
| `netif_set_down` | 956 | bring down netif |
| `netif_set_link_up` | 1025 | link up (触发 DHCP/AUTOIP) |
| `netif_set_link_down` | 1063 | link down |
| `netif_set_default` | 853 | 设为默认 netif |
| `netif_set_flags` | — | 设置 flags |
| `netif_clear_flags` | — | 清除 flags |

### Loopback
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_loop_output` | 1115 | 发送包到 loopback (本机) |
| `netif_loopif_init` | 154 | loopback 接口初始化 |
| `netif_poll` | 1263 | 处理 loopback 队列 |
| `netif_poll_all` | 1336 | 轮询所有 netif |
| `netif_loop_output_ipv4` | 1238 | IPv4 loopback output |
| `netif_loop_output_ipv6` | 1247 | IPv6 loopback output |

### Callbacks
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_set_status_callback` | 994 | 设置状态变化回调 |
| `netif_set_link_callback` | 1088 | 设置 link 变化回调 |
| `netif_set_remove_callback` | 1010 | 设置移除回调 |
| `netif_add_ext_callback` | 1850 | 添加扩展事件监听 |
| `netif_remove_ext_callback` | 1867 | 移除扩展事件监听 |
| `netif_invoke_ext_callback` | 1900 | 触发扩展事件 |

### IPv6
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_ip6_addr_set` | 1382 | 设置 IPv6 地址 |
| `netif_ip6_addr_set_parts` | 1404 | 设置 IPv6 地址 (分 4 个 u32) |
| `netif_ip6_addr_set_state` | 1461 | 设置地址状态 (VALID/TENTATIVE) |
| `netif_get_ip6_addr_match` | 1532 | 查找匹配的 IPv6 地址 |
| `netif_create_ip6_linklocal_address` | 1565 | 创建 IPv6 link-local 地址 |
| `netif_add_ip6_address` | 1633 | 添加 IPv6 地址 |

### 内部
| 函数 | 行号 | 功能 |
|------|------|------|
| `netif_do_ip_addr_changed` | 452 | 地址变更时通知各协议 |
| `netif_do_set_ipaddr` | 467 | 设置 IP 地址核心逻辑 |
| `netif_do_set_netmask` | 533 | 设置 netmask 核心逻辑 |
| `netif_do_set_gw` | 597 | 设置网关核心逻辑 |
| `netif_issue_reports` | 912 | 触发 IGMP/ARP/MLD reports |
| `netif_forward_enable` | 739 | 启用转发 |
| `netif_forward_disable` | 749 | 禁用转发 |

## 关键数据结构

### 全局变量
```c
struct netif *netif_list;     // netif 链表
struct netif *netif_default; // 默认 netif
#define NETIF_FOREACH(n)     // 遍历宏
```

### netif 核心字段 (struct netif)
```c
struct netif {
  struct netif *next;        // 链表
  char name[2];              // "en", "lo" 等
  u8_t num;                  // 编号 (en0, en1...)
  char *fullname;            // 完整名称

  // IP addresses
  ip_addr_t ip_addr;        // IPv4 地址
  ip_addr_t netmask;
  ip_addr_t gw;

  // Callbacks
  netif_input_fn input;      // 输入处理函数
  netif_output_fn output;   // IPv4 输出函数
  netif_output_fn output_ip6;  // IPv6 输出函数

  // Status
  u16_t mtu;                // MTU
  u8_t flags;               // NETIF_FLAG_*

  void *state;              // 驱动状态
  void *client_data[...];   // 客户端数据

  // ...
};
```

### 重要 Flags
```c
NETIF_FLAG_UP           // 接口已 up
NETIF_FLAG_LINK_UP      // 链路 up
NETIF_FLAG_ETHARP       // 支持 ARP
NETIF_FLAG_ETHERNET     // 以太网
NETIF_FLAG_IGMP         // IGMP
NETIF_FLAG_MLD6         // MLDv6
NETIF_FLAG_FORWARD      // 允许转发
```

## 调用链

### netif 添加
```
netif_add()
  → netif_set_addr()          // 设置 IP/mask/gw
  → init()                     // 调用驱动初始化
  → 分配 num (O(n²) 算法)
  → 插入 netif_list 链表
  → igmp_start()              // 如果 IGMP enabled
```

### 输入分发 (netif_input)
```
nic_rx_thread / tcpip_thread
  → netif_input(p, netif)
    → [NETIF_FLAG_ETHARP|ETHERNET] → ethernet_input()
    → else → ip_input()
```

### 地址变更
```
netif_set_ipaddr()
  → netif_do_set_ipaddr()
    → netif_do_ip_addr_changed()
      → tcp_netif_ip_addr_changed()
      → udp_netif_ip_addr_changed()
      → raw_netif_ip_addr_changed()
    → mib2_remove_ip4() / mib2_add_ip4()
    → netif_issue_reports()   // 触发 ARP gratuitous / IGMP reports
```

## 交叉引用

### Analysis 层
- [[entities/linux/lwip/lwip-netif]] — netif 结构详解
- [[entities/linux/lwip/lwip-netif-add]] — netif_add 流程
- [[entities/linux/lwip/lwip-ethernet-input]] — L2 入口
- [[entities/linux/lwip/lwip-ethernet-output]] — L2 出口
- [[entities/linux/lwip/lwip-network-init]] — 初始化流程

### IP 层
- [[entities/linux/lwip/source/ip4.c]] — ip4_route / ip4_output

### 协议层
- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread
