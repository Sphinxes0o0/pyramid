---
type: entity
tags: [linux, lwip, network, nsv, sys-net-ctl, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# sys_net_ctl — Network Control Interface

## 定义

`sys_net_ctl` 是 NSv 的网络控制接口，提供系统管理和监控功能，通过 seL4 IPC 接收应用的控制命令。

## 命令类型

| 命令 | 功能 | 对应 lwIP 函数 |
|------|------|---------------|
| `TEST` | 连接测试 | - |
| `IFCONFIG` | 网口配置/显示 | `ifconfig_update`, `netif_set_addr` |
| `NETSTAT` | socket 统计 | `netstat_fill_info` |
| `LWFWCFG` | 防火墙配置 | `lwfw_dump_fw_info`, `lwfw_config_reload` |
| `MC` | 多播组管理 | `igmp_joingroup`, `igmp_leavegroup` |

## IFCONFIG — 网口配置

```c
struct nsv_ifconfig_show {
    char ifname[NETIF_NAMESIZE];    // 网口名称
    ip4_addr_t ip_addr;             // IP 地址
    ip4_addr_t netmask;            // 子网掩码
    ip4_addr_t gw;                 // 网关
    uint8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
    uint32_t mtu;                   // MTU
    uint32_t flags;                 // 网口标志
    uint64_t rx_packets, rx_bytes, rx_errors;
    uint64_t tx_packets, tx_bytes, tx_errors;
};
```

## NETSTAT — 网络统计

```c
struct nsv_netstat_info {
    uint32_t total_sockets;    // 总 socket 数
    uint32_t active_sockets;   // 活跃 socket 数
    uint32_t tcp_sockets;      // TCP socket 数
    uint32_t udp_sockets;      // UDP socket 数
    uint32_t raw_sockets;      // RAW socket 数
    uint32_t packet_sockets;   // AF_PACKET socket 数

    struct socket_info {
        int fd; pid_t pid; int protocol;
        ip4_addr_t local_addr; uint16_t local_port;
        ip4_addr_t remote_addr; uint16_t remote_port;
        uint32_t state;
        uint64_t rx_bytes, tx_bytes;
    } sockets[NSV_MAX_SOCKETS];
};
```

## LWFWCFG — 防火墙配置

```c
case SYS_NET_CTL_CMD_LWFWCFG: {
    if (opt == SYS_NET_CTL_CMD_LWFWCFG_SHOW) {
        lwfw_dump_fw_info(true);  // 显示防火墙配置
    } else if (opt == SYS_NET_CTL_CMD_LWFWCFG_CONFIG) {
        err = lwfw_config_reload();  // 重新加载规则
    }
}
```

## MC — IGMP 多播组管理

```c
case SYS_NET_CTL_CMD_MC: {
    if (opt == SYS_NET_CTL_CMD_MC_ADD) {
        // 解析多播地址
        ip4_addr_t mc_addr;
        // 查找要操作的网口
        struct netif *netif = find_netif_by_name(ifname);
        // 调用 lwIP IGMP
        err = igmp_joingroup(&netif->ip_addr, &mc_addr);
    }
}
```

## 调用链

```
App                           NSv                         lwIP
 │                             │                           │
 │  seL4 IPC (SYS_NET_CTL)   │                           │
 │────────────────────────────►│                           │
 │                             ▼                           │
 │                       sys_net_ctl()                      │
 │                             │                           │
 │  ◄─────────────────────────│                           │
 │                             │                           │
 ├─► IFCONFIG_UPD              │                           │
 │                             ▼                           │
 │                       ifconfig_update()                 │
 │                             ▼                           │
 │                       netif_set_addr()                 │
 │                             │                           │
 ├─► LWFWCFG_SHOW              │                           │
 │                             ▼                           │
 │                       lwfw_dump_fw_info()               │
```

## 相关概念

- [[entities/linux/lwip/lwip-nsv-event-loop]] — NSv 事件循环
- [[entities/linux/lwip/lwip-firewall]] — LWFW 防火墙
- [[entities/linux/lwip/lwip-sel4-function]] — 整体 lwIP 调用链

## 来源详情

- [[sources/safeos-lwip-extensions]]
