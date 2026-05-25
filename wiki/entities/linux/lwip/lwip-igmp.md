---
type: entity
tags: [linux, lwip, network, igmp, multicast]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP IGMP Analysis

## 定义

IGMP (Internet Group Management Protocol) 是 IPv4 多播组成员管理协议，负责：加入多播组、离开多播组、查询响应。

## IGMP 消息类型

| 类型 | 名称 | 说明 |
|------|------|------|
| 0x11 | **IGMP_MEMB_QUERY** | 路由器查询 (General/Specific) |
| 0x12 | **IGMPv1_MEMB_REPORT** | 成员报告 (v1) |
| 0x16 | **IGMPv2_MEMB_REPORT** | 成员报告 (v2) |
| 0x17 | **IGMP_LEAVE_GROUP** | 离开组 (v2) |
| 0x22 | **IGMPv3_MEMB_REPORT** | 成员报告 (v3) |

## IGMP Group 结构

```c
struct igmp_group {
    struct igmp_group *next;     // 链表指针
    ip4_addr_t group_address;    // 多播组地址
    u8_t last_reporter_flag;    // 是否最后报告者
    u8_t group_state;           // 组状态
    u16_t timer;                // 报告定时器
    u8_t use;                   // 引用计数
};
```

### Group 状态

```c
#define IGMP_GROUP_DELAYING_MEMBER  0x01  // 等待发送报告
#define IGMP_GROUP_PENDING_MEMBER   0x02  // 等待加入确认
#define IGMP_GROUP_NON_MEMBER       0x03  // 非成员
```

## 核心函数

### igmp_joingroup — 加入多播组

```c
err_t igmp_joingroup_netif(struct netif *netif, const ip4_addr_t *groupaddr) {
    group = igmp_lookup_group(netif, groupaddr);
    if (group->group_state != IGMP_GROUP_NON_MEMBER) {
        group->use++;
        return ERR_OK;
    }
    netif_igmp_mac_filter(netif, IGMP_ADD_MAC_FILTER, groupaddr);
    group->group_state = IGMP_GROUP_PENDING_MEMBER;
    group->use = 1;
    igmp_start_timer(group, IGMP_JOIN_DELAYING_MEMBER_TMR);
    igmp_send_report(group);
    return ERR_OK;
}
```

### igmp_leavegroup — 离开多播组

```c
err_t igmp_leavegroup_netif(struct netif *netif, const ip4_addr_t *groupaddr) {
    group->use--;
    if (group->use > 0) return ERR_OK;
    if (group->last_reporter_flag) {
        igmp_send_leave(group);  // 发送 Leave
    }
    netif_igmp_mac_filter(netif, IGMP_DEL_MAC_FILTER, groupaddr);
    group->group_state = IGMP_GROUP_NON_MEMBER;
    return ERR_OK;
}
```

## IGMP 流程

```
加入多播组:
  igmp_joingroup()
    ├─► igmp_lookup_group()
    ├─► netif_igmp_mac_filter(ADD)
    ├─► 设置 group_state = PENDING
    ├─► 启动报告定时器
    └─► 发送 IGMP Report

接收 IGMP 查询:
  igmp_input()
    ├─► General Query → 延迟响应所有组
    └─► Specific Query → 延迟响应特定组

定时器到期:
  igmp_tmr()
    └─► 发送 IGMP Report

离开多播组:
  igmp_leavegroup()
    ├─► 递减 use 计数
    ├─► 发送 Leave (如果是最后报告者)
    └─► 设置 group_state = NON_MEMBER
```

## 与 lwIP 集成

```
ip4_input()
    │
    ├─► 检查是否是 IGMP (proto = 2)
    │     └─► igmp_input()
    │
    └─► 检查是否是多播
          ├─► IGMP 组查找
          └─► 如果在组中，接收数据
```

## 相关概念

- [[entities/linux/lwip/lwip-ip4-input]] — ip4_input 中的多播路由
- [[entities/linux/lwip/lwip-udp-socket]] — UDP mcast_group 字段
- [[entities/linux/lwip/lwip-netif]] — NETIF_FLAG_IGMP 标志
