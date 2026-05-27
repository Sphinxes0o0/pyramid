# DHCP Client 分析 — T-043

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: DHCP client 实现、地址获取、renewal

---

## 1. 概述

DHCP (Dynamic Host Configuration Protocol) 是 IPv4 的动态地址配置协议。lwIP 提供 DHCP Client 实现，符合 RFC 2131 和 RFC 2132。

**主要功能**：
- 自动获取 IP 地址
- 地址租约管理
- 自动续约

---

## 2. DHCP 状态机

### 2.1 状态定义

**文件**: `include/lwip/prot/dhcp.h:100-113`

```c
typedef enum {
    DHCP_STATE_OFF             = 0,  // DHCP 未启用
    DHCP_STATE_REQUESTING      = 1,    // 请求中
    DHCP_STATE_INIT            = 2,    // 初始化
    DHCP_STATE_REBOOTING       = 3,   // 重启中
    DHCP_STATE_REBINDING       = 4,   // 重新绑定中
    DHCP_STATE_RENEWING        = 5,   // 续约中
    DHCP_STATE_SELECTING        = 6,   // 选择中
    DHCP_STATE_INFORMING       = 7,   // 通知中
    DHCP_STATE_CHECKING       = 8,   // 检查中 (ARP 探测)
    DHCP_STATE_PERMANENT       = 9,   // 永久 (未实现)
    DHCP_STATE_BOUND           = 10,  // 已绑定
    DHCP_STATE_RELEASING       = 11,  // 释放中 (未实现)
    DHCP_STATE_BACKING_OFF     = 12   // 退避中
} dhcp_state_enum_t;
```

### 2.2 状态转换图

```
                           dhcp_start()
                                │
                                ▼
                          DHCP_STATE_INIT
                                │
                                ▼
                    ┌───────────────────────┐
                    │  DHCP_STATE_SELECTING  │
                    │    发送 DHCP_DISCOVER │
                    └───────────────────────┘
                                │
                    ┌───────────────────────┐
                    │    收到 DHCP_OFFER    │
                    └───────────────────────┘
                                │
                                ▼
                          DHCP_STATE_CHECKING
                                │
                                │ ARP 探测
                                ▼
                    ┌───────────────────────┐
                    │  DHCP_STATE_REQUESTING│
                    │   发送 DHCP_REQUEST  │
                    └───────────────────────┘
                                │
                    ┌───────────────────────┐
                    │    收到 DHCP_ACK     │
                    └───────────────────────┘
                                │
                                ▼
                          DHCP_STATE_BOUND
                                │
                    ┌───────────────────────┐
                    │     T1 过期          │
                    │  (50% lease time)    │
                    └───────────────────────┘
                                │
                                ▼
                         DHCP_STATE_RENEWING
                                │
                    ┌───────────────────────┐
                    │     续约成功          │
                    └───────────────────────┘
                                │
                                ▼
                          DHCP_STATE_BOUND
```

---

## 3. 数据结构

### 3.1 dhcp 结构体

**文件**: `include/lwip/dhcp.h:67`

```c
struct dhcp {
    u32_t xid;                    // 事务 ID
    u8_t pcb_allocated;           // PCB 是否已分配
    u8_t state;                   // 当前状态
    u8_t tries;                   // 重试次数

    u16_t request_timeout;         // 请求超时
    u16_t t1_timeout;             // T1 续约超时
    u16_t t2_timeout;             // T2 重新绑定超时
    u16_t t1_renew_time;         // 距续约的时间
    u16_t t2_rebind_time;        // 距重新绑定的时间
    u16_t lease_used;             // 租约已使用时间
    u16_t t0_timeout;             // 租约超时

    ip_addr_t server_ip_addr;     // DHCP 服务器地址
    ip4_addr_t offered_ip_addr;   // 获得的 IP 地址
    ip4_addr_t offered_sn_mask;   // 子网掩码
    ip4_addr_t offered_gw_addr;   // 网关地址

    u32_t offered_t0_lease;       // 租约时间
    u32_t offered_t1_renew;       // 建议续约时间
    u32_t offered_t2_rebind;      // 建议重新绑定时间
};
```

---

## 4. 核心函数

### 4.1 dhcp_start — 启动 DHCP

**文件**: `dhcp.c:737`

```c
void dhcp_start(struct netif *netif)
{
    struct dhcp *dhcp;

    // 1. 分配 dhcp 结构
    dhcp = (struct dhcp *)memp_malloc(MEMP_DHCP);
    if (dhcp == NULL) {
        return;
    }

    // 2. 初始化
    memset(dhcp, 0, sizeof(struct dhcp));
    netif_set_client_data(netif, LWIP_NETIF_CLIENT_DATA_INDEX_DHCP, dhcp);

    // 3. 设置状态为 INIT
    dhcp_set_state(dhcp, DHCP_STATE_INIT);

    // 4. 绑定 UDP PCB
    dhcp->pcb = udp_new();
    udp_bind(dhcp->pcb, IP_ADDR_ANY, DHCP_CLIENT_PORT);
    udp_recv(dhcp->pcb, dhcp_recv, netif);

    // 5. 开始发现
    dhcp_discover(netif);
}
```

### 4.2 dhcp_discover — 发现 DHCP 服务器

**文件**: `dhcp.c`

```c
static err_t dhcp_discover(struct netif *netif)
{
    struct dhcp *dhcp = netif_dhcp_data(netif);

    // 生成事务 ID
    dhcp->xid = LWIP_RAND();

    // 创建 DHCP DISCOVER 消息
    p = dhcp_create_msg(netif, dhcp, DHCP_DISCOVER, &options_out_len);

    // 发送广播
    udp_sendto(dhcp->pcb, p, IP_BROADCAST, DHCP_SERVER_PORT);

    // 切换到 SELECTING 状态
    dhcp_set_state(dhcp, DHCP_STATE_SELECTING);

    pbuf_free(p);
    return ERR_OK;
}
```

### 4.3 dhcp_recv — 接收 DHCP 回复

**文件**: `dhcp.c`

```c
static void dhcp_recv(void *arg, struct udp_pcb *pcb, struct pbuf *p,
                     const ip_addr_t *addr, u16_t port)
{
    // 解析 DHCP 选项
    dhcp_parse_reply(p, dhcp);

    // 根据消息类型处理
    switch (dhcp->msg_type) {
        case DHCP_OFFER:
            // 保存 offer 的 IP 地址
            dhcp->offered_ip_addr = ...;
            // 发送 REQUEST
            dhcp_select(netif);
            break;

        case DHCP_ACK:
            // 调用 dhcp_bind 应用地址
            dhcp_bind(netif);
            break;

        case DHCP_NAK:
            // 重试发现
            dhcp_handle_nak(netif);
            break;
    }
}
```

### 4.4 dhcp_bind — 应用获得的地址

**文件**: `dhcp.c:1137`

```c
static void dhcp_bind(struct netif *netif)
{
    struct dhcp *dhcp = netif_dhcp_data(netif);

    // 设置 IP 地址
    netif_set_addr(netif, &dhcp->offered_ip_addr,
                   &dhcp->offered_sn_mask,
                   &dhcp->offered_gw_addr);

    // 设置定时器
    dhcp->t0_timeout = dhcp->offered_t0_lease;
    dhcp->t1_renew_time = dhcp->offered_t1_renew;
    dhcp->t2_rebind_time = dhcp->offered_t2_rebind;

    // 切换到 BOUND 状态
    dhcp_set_state(dhcp, DHCP_STATE_BOUND);
}
```

---

## 5. 定时器管理

### 5.1 定时器配置

```c
#define DHCP_COARSE_TIMER_SECS  60   // 粗粒度定时器 (1 分钟)
#define DHCP_FINE_TIMER_MSECS   500  // 细粒度定时器 (500 ms)
```

### 5.2 粗粒度定时器 (dhcp_coarse_tmr)

**文件**: `dhcp.c:430`

```c
void dhcp_coarse_tmr(void)
{
    NETIF_FOREACH(netif) {
        struct dhcp *dhcp = netif_dhcp_data(netif);
        if (dhcp->state == DHCP_STATE_OFF) continue;

        // 检查租约超时 (t0)
        if (dhcp->lease_used == dhcp->t0_timeout) {
            dhcp_release_and_stop(netif);
            dhcp_start(netif);  // 重新发现
        }

        // 检查 T2 重新绑定超时 (t2)
        if (dhcp->t2_rebind_time == 1) {
            dhcp_t2_timeout(netif);
        }

        // 检查 T1 续约超时 (t1)
        if (dhcp->t1_renew_time == 1) {
            dhcp_t1_timeout(netif);
        }
    }
}
```

### 5.3 T1 续约定时器

```c
static void dhcp_t1_timeout(struct netif *netif)
{
    // 进入 RENEWING 状态
    dhcp_set_state(dhcp, DHCP_STATE_RENEWING);

    // 发送 DHCP_REQUEST 进行续约
    dhcp_request(netif);
}
```

### 5.4 T2 重新绑定定时器

```c
static void dhcp_t2_timeout(struct netif *netif)
{
    // 进入 REBINDING 状态
    dhcp_set_state(dhcp, DHCP_STATE_REBINDING);

    // 发送 DHCP_REQUEST 进行重新绑定
    dhcp_rebind(netif);
}
```

---

## 6. DHCP 消息格式

### 6.1 DHCP Header

```c
struct dhcp_msg {
    u8_t op;           // 操作码: 1=BOOTREQUEST, 2=BOOTREPLY
    u8_t htype;       // 硬件类型: 1=Ethernet
    u8_t hlen;        // 硬件地址长度: 6=Ethernet
    u8_t hops;        // 跳数
    u32_t xid;        // 事务 ID
    u16_t secs;       // 已用秒数
    u16_t flags;      // 标志
    u32_t ciaddr;     // 客户端 IP 地址
    u32_t yiaddr;     // 你的 IP 地址 (offered)
    u32_t siaddr;     // 服务器 IP 地址
    u32_t giaddr;     // 网关 IP 地址
    u8_t chaddr[16];  // 客户端硬件地址
    u8_t sname[64];   // 服务器名称
    u8_t file[128];   // 启动文件名
    u32_t cookie;     // DHCP 魔法数
    u8_t options[];   // DHCP 选项
};
```

### 6.2 DHCP 选项

| 选项 | 代码 | 说明 |
|------|------|------|
| DHCP Message Type | 53 | 消息类型 (DISCOVER/OFFER/REQUEST/ACK/NAK) |
| Server Identifier | 54 | DHCP 服务器地址 |
| IP Address Lease Time | 51 | 租约时间 (秒) |
| Renewal Time Value (T1) | 58 | 续约时间 |
| Rebinding Time Value (T2) | 59 | 重新绑定时间 |
| Subnet Mask | 1 | 子网掩码 |
| Router | 3 | 默认网关 |
| Domain Name Server | 6 | DNS 服务器 |

---

## 7. SafeOS 中的 DHCP 使用

### 7.1 启动 DHCP

**文件**: `main.c:6462`

```c
#ifdef CONFIG_PLAT_QEMU_ARM_VIRT
    dhcp_start(&vnet_if);
#else
    // 从配置读取静态 IP
    ethif_update(netif);
#endif
```

### 7.2 定时器调用

DHCP 定时器通过 `dhcp_coarse_tmr()` 和 `dhcp_fine_tmr()` 调用，通常在系统定时器上下文中。

---

## 8. 租约时间线

```
0%                        50%                       87.5%                      100%
│                          │                          │                          │
│                          │                          │                          │
T1 (续约)                  │                          │                          │
│◄─────────────────────────│                          │                          │
│                          │                          │                          │
│                          T2 (重新绑定)              │                          │
│                          │◄─────────────────────────│                          │
│                          │                          │                          │
│                          │                          │                          │
│                          │                          │                          │
│                          │                          │                          │
│                          │                          │                          │
租约开始 ──────────────────────────────────────────────────────────────────── 租约结束
     │                          │                          │
     │                          │                          │
     ▼                          ▼                          ▼
  DHCP_STATE_BOUND ──── DHCP_STATE_RENEWING ─── DHCP_STATE_REBINDING
     │                          │                          │
     │                          │                          │
     └──────────────────────────┴──────────────────────────┘
                  如果续约/重绑定成功，回到 BOUND
```

---

## 9. 总结

### 9.1 DHCP 状态机

```
INIT → SELECTING → CHECKING → REQUESTING → BOUND
         │                                       │
         │◄──────────────────────────────────────┘
         │              (NAK 或超时重试)
```

### 9.2 关键定时器

| 定时器 | 触发时间 | 处理 |
|--------|----------|------|
| T1 | 50% lease | 续约 (RENEWING) |
| T2 | 87.5% lease | 重新绑定 (REBINDING) |
| Lease | 100% lease | 释放并重新发现 |

### 9.3 核心流程

```
dhcp_start()
    │
    ├─► dhcp_discover() → SELECTING
    │       │
    │       ▼
    │     等待 OFFER
    │       │
    │       ▼
    │     dhcp_select() → REQUESTING
    │       │
    │       ▼
    │     等待 ACK
    │       │
    │       ▼
    └─► dhcp_bind() → BOUND
            │
            ├─► netif_set_addr()
            │
            └─► 启动 T1/T2 定时器
```
