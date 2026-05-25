---
type: entity
tags: [linux, lwip, network, dhcp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP DHCP Client Analysis

## 定义

DHCP (Dynamic Host Configuration Protocol) 是 IPv4 的动态地址配置协议。lwIP 提供 DHCP Client 实现，符合 RFC 2131 和 RFC 2132，主要功能：自动获取 IP 地址、地址租约管理、自动续约。

## DHCP 状态机

```c
typedef enum {
    DHCP_STATE_OFF             = 0,  // DHCP 未启用
    DHCP_STATE_REQUESTING      = 1,    // 请求中
    DHCP_STATE_INIT            = 2,    // 初始化
    DHCP_STATE_REBOOTING       = 3,   // 重启中
    DHCP_STATE_REBINDING       = 4,   // 重新绑定中
    DHCP_STATE_RENEWING        = 5,   // 续约中
    DHCP_STATE_SELECTING       = 6,    // 选择中
    DHCP_STATE_INFORMING       = 7,   // 通知中
    DHCP_STATE_CHECKING        = 8,   // 检查中 (ARP 探测)
    DHCP_STATE_BOUND           = 10,  // 已绑定
} dhcp_state_enum_t;
```

## 状态转换图

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
                    │     T1 过期 (50%)    │
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

## dhcp 结构体

```c
struct dhcp {
    u32_t xid;                    // 事务 ID
    u8_t pcb_allocated;           // PCB 是否已分配
    u8_t state;                   // 当前状态
    u8_t tries;                   // 重试次数

    u16_t t1_renew_time;         // 距续约的时间
    u16_t t2_rebind_time;        // 距重新绑定的时间
    u16_t lease_used;             // 租约已使用时间
    u16_t t0_timeout;             // 租约超时

    ip_addr_t server_ip_addr;     // DHCP 服务器地址
    ip4_addr_t offered_ip_addr;   // 获得的 IP 地址
    ip4_addr_t offered_sn_mask;   // 子网掩码
    ip4_addr_t offered_gw_addr;   // 网关地址
};
```

## 定时器管理

| 定时器 | 触发时间 | 处理 |
|--------|----------|------|
| T1 | 50% lease | 续约 (RENEWING) |
| T2 | 87.5% lease | 重新绑定 (REBINDING) |
| Lease | 100% lease | 释放并重新发现 |

## 核心流程

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

## 相关概念

- [[entities/linux/lwip/lwip-network-init]] — dhcp_start 在初始化中调用
- [[entities/linux/lwip/lwip-netif]] — netif_set_addr 设置 IP 地址
- [[entities/linux/lwip/lwip-udp-socket]] — DHCP 使用 UDP socket 通信
