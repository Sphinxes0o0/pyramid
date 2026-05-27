# sys_net_ctl 分析 — T-105

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: netstat/ifconfig/lwfwcfg 等控制命令

---

## 1. 概述

`sys_net_ctl` 是 NSv 的网络控制接口，提供系统管理和监控功能。

**子命令类型**：
- `TEST` — 连接测试
- `IFCONFIG` — 网口配置和状态
- `NETSTAT` — 网络统计信息
- `LWFWCFG` — 防火墙配置
- `MC` — IGMP 多播组管理

---

## 2. 命令分发

**文件**: `main.c:2140-2340`

```c
switch (sel4_msg_info_get_label(info)) {
    // ...
    case SYS_NET_CTL: {
        int cmd = (int)sel4_get_mr(0);
        int opt = (int)sel4_get_mr(1);

        switch (cmd) {
            case SYS_NET_CTL_CMD_TEST:       // 测试命令
            case SYS_NET_CTL_CMD_IFCONFIG:   // 网口配置
            case SYS_NET_CTL_CMD_NETSTAT:   // 网络统计
            case SYS_NET_CTL_CMD_LWFWCFG:   // 防火墙配置
            case SYS_NET_CTL_CMD_MC:        // 多播组管理
        }
    }
}
```

---

## 3. IFCONFIG — 网口配置

### 3.1 子命令

| 命令 | 说明 |
|------|------|
| `IFCONFIG_UPD` | 更新网口配置 |
| `IFCONFIG_SHOW` | 显示网口信息 |

### 3.2 IFCONFIG_SHOW

```c
case SYS_NET_CTL_CMD_IFCONFIG: {
    if (opt == SYS_NET_CTL_CMD_IFCONFIG_UPD) {
        // 更新网口配置
        struct nsv_ifconfig_options config_ifconfig;
        memcpy(&config_ifconfig, p, sizeof(struct nsv_ifconfig_options));
        err = ifconfig_update(&config_ifconfig);
    } else if (opt == SYS_NET_CTL_CMD_IFCONFIG_SHOW) {
        // 显示网口状态
        ifconfig_ifs_show_shm(&config_ifconfig, ifconfig_show_info,
                             proto_stat_show_info, arp_show_info);
    }
}
```

### 3.3 网口信息结构

```c
struct nsv_ifconfig_show {
    char ifname[NETIF_NAMESIZE];    // 网口名称
    ip4_addr_t ip_addr;             // IP 地址
    ip4_addr_t netmask;             // 子网掩码
    ip4_addr_t gw;                  // 网关
    uint8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
    uint32_t mtu;                   // MTU
    uint32_t flags;                 // 网口标志
    // 统计信息
    uint64_t rx_packets;
    uint64_t rx_bytes;
    uint64_t rx_errors;
    uint64_t tx_packets;
    uint64_t tx_bytes;
    uint64_t tx_errors;
};
```

---

## 4. NETSTAT — 网络统计

### 4.1 子命令

| 命令 | 说明 |
|------|------|
| `NETSTAT_SHOW` | 显示 socket 统计 |

### 4.2 NETSTAT_SHOW

```c
case SYS_NET_CTL_CMD_NETSTAT: {
    if (opt == SYS_NET_CTL_CMD_NETSTAT_SHOW) {
        struct nsv_netstat_info *info = netstat_info;
        // 填充 socket 统计信息
        netstat_fill_info(info);
        // 返回给调用者
        sys_reply_with_two_direct(0, info, sizeof(*info));
    }
}
```

### 4.3 Socket 统计信息

```c
struct nsv_netstat_info {
    uint32_t total_sockets;         // 总 socket 数
    uint32_t active_sockets;        // 活跃 socket 数
    uint32_t tcp_sockets;           // TCP socket 数
    uint32_t udp_sockets;           // UDP socket 数
    uint32_t raw_sockets;           // RAW socket 数
    uint32_t packet_sockets;        // AF_PACKET socket 数

    // per-socket 信息
    struct socket_info {
        int fd;                     // 文件描述符
        pid_t pid;                  // 进程 PID
        int protocol;               // 协议
        ip4_addr_t local_addr;     // 本地地址
        uint16_t local_port;       // 本地端口
        ip4_addr_t remote_addr;    // 远端地址
        uint16_t remote_port;      // 远端端口
        uint32_t state;             // 连接状态
        // 统计
        uint64_t rx_bytes;
        uint64_t tx_bytes;
    } sockets[NSV_MAX_SOCKETS];
};
```

---

## 5. LWFWCFG — 防火墙配置

### 5.1 子命令

| 命令 | 说明 |
|------|------|
| `LWFWCFG_SHOW` | 显示防火墙规则 |
| `LWFWCFG_CONFIG` | 配置防火墙规则 |

### 5.2 LWFWCFG_CONFIG

```c
case SYS_NET_CTL_CMD_LWFWCFG: {
    if (opt == SYS_NET_CTL_CMD_LWFWCFG_SHOW) {
        // 显示当前防火墙配置
        lwfw_dump_fw_info(true);
    } else if (opt == SYS_NET_CTL_CMD_LWFWCFG_CONFIG) {
        // 重新加载防火墙规则
        err = lwfw_config_reload();
    }
}
```

### 5.3 lwfw_dump_fw_info 输出示例

```
========= Dump firewall statistics: =========
    err_log_cnt: 0
    warn_log_cnt: 0
    throttled_logs: 0
    total_rx_drop: 100
    total_tx_drop: 50
    total_event_cnt: 1000
    cfg_loglevel: 3
    cfg_reload: 5
```

---

## 6. MC — IGMP 多播组管理

### 6.1 子命令

| 命令 | 说明 |
|------|------|
| `MC_ADD` | 加入多播组 |
| `MC_DEL` | 离开多播组 |
| `MC_SHOW` | 显示多播组状态 |

### 6.2 MC_ADD

```c
case SYS_NET_CTL_CMD_MC: {
    if (opt == SYS_NET_CTL_CMD_MC_ADD) {
        // 解析多播地址
        ip4_addr_t mc_addr;
        memcpy(&mc_addr, p, sizeof(mc_addr));

        // 查找要操作的网口
        struct netif *netif = find_netif_by_name(ifname);

        // 调用 lwIP IGMP
        err = igmp_joingroup(&netif->ip_addr, &mc_addr);

        // 记录到 net_process_info
        net_process_mc_add(pid, mc_addr, ifname);
    }
}
```

### 6.3 IGMP 流程

```
App                         NSv                         lwIP
 │                           │                           │
 │  MC_ADD (igmp_joingroup) │                           │
 │─────────────────────────►│                           │
 │                           ▼                           │
 │                     igmp_joingroup()                  │
 │                           │                           │
 │                           ▼                           │
 │                     netif->output()                  │
 │                           │                           │
 │                           ▼                           │
 │                     ethif_link_output()              │
 │                           │                           │
 │                           ▼                           │
 │                       NIC DMA                        │
 │                                                   │
 │  ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
 │                           │
 │                     IGMP Report 发送
```

---

## 7. TEST — 测试命令

### 7.1 子命令

```c
case SYS_NET_CTL_CMD_TEST: {
    // 返回测试结果
    int test_type = sel4_get_mr(2);
    switch (test_type) {
        case TEST_PING:
            // PING 测试
            break;
        case TEST_CONNECT:
            // 连接测试
            break;
    }
}
```

---

## 8. 数据结构

### 8.1 nsv_ifconfig_options

```c
struct nsv_ifconfig_options {
    char ifname[NETIF_NAMESIZE];    // 网口名称
    ip4_addr_t ip_addr;             // IP 地址
    ip4_addr_t netmask;             // 子网掩码
    ip4_addr_t gw;                  // 网关
    uint8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
    uint32_t mtu;                   // MTU
    uint32_t flags;                 // 标志
};
```

---

## 9. 总结

### 9.1 sys_net_ctl 命令汇总

| 命令 | 功能 | 对应 lwIP 函数 |
|------|------|---------------|
| TEST | 连接测试 | - |
| IFCONFIG | 网口配置/显示 | `ifconfig_update`, `netif_set_addr` |
| NETSTAT | socket 统计 | `netstat_fill_info` |
| LWFWCFG | 防火墙配置 | `lwfw_dump_fw_info`, `lwfw_config_reload` |
| MC | 多播组管理 | `igmp_joingroup`, `igmp_leavegroup` |

### 9.2 调用链

```
App                           NSv                         lwIP
 │                             │                           │
 │  seL4 IPC (SYS_NET_CTL)    │                           │
 │────────────────────────────►│                           │
 │                             ▼                           │
 │                     sys_net_ctl()                      │
 │                             │                           │
 │  ◄─────────────────────────│                           │
 │                             │                           │
 ├─► IFCONFIG_UPD              │                           │
 │                             ▼                           │
 │                       ifconfig_update()                 │
 │                             │                           │
 │                             ▼                           │
 │                       netif_set_addr()                 │
 │                             │                           │
 ├─► NETSTAT_SHOW              │                           │
 │                             ▼                           │
 │                       netstat_fill_info()               │
 │                             │                           │
 ├─► LWFWCFG_SHOW              │                           │
 │                             ▼                           │
 │                       lwfw_dump_fw_info()               │
 │                             │                           │
 ├─► MC_ADD                    │                           │
 │                             ▼                           │
 │                       igmp_joingroup()                  │
 │                             │                           │
 ▼                             ▼                           ▼
```
