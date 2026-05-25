# IGMP 分析 — T-051

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: IGMPv1/v2/v3 处理、group management、多播组加入/离开

---

## 1. 概述

IGMP (Internet Group Management Protocol) 是 IPv4 多播组成员管理协议：

1. **加入多播组**: 主机通知路由器接收特定多播组流量
2. **离开多播组**: 主机通知路由器不再接收
3. **查询响应**: 路由器查询时，主机响应组成员状态

---

## 2. IGMP 消息类型

| 类型 | 名称 | 说明 |
|------|------|------|
| 0x11 | **IGMP_MEMB_QUERY** | 路由器查询 (General/Specific) |
| 0x12 | **IGMPv1_MEMB_REPORT** | 成员报告 (v1) |
| 0x16 | **IGMPv2_MEMB_REPORT** | 成员报告 (v2) |
| 0x17 | **IGMP_LEAVE_GROUP** | 离开组 (v2) |
| 0x22 | **IGMPv3_MEMB_REPORT** | 成员报告 (v3) |

---

## 3. IGMP Group 结构

**文件**: `include/lwip/igmp.h:74-87`

```c
struct igmp_group {
    struct igmp_group *next;     // 链表指针
    ip4_addr_t group_address;   // 多播组地址
    u8_t last_reporter_flag;   // 是否最后报告者
    u8_t group_state;          // 组状态
    u16_t timer;               // 报告定时器
    u8_t use;                  // 引用计数
};
```

### 3.1 Group 状态

```c
#define IGMP_GROUP_DELAYING_MEMBER  0x01  // 等待发送报告
#define IGMP_GROUP_PENDING_MEMBER   0x02  // 等待加入确认
#define IGMP_GROUP_NON_MEMBER       0x03  // 非成员
```

---

## 4. igmp_start — 初始化 netif 的 IGMP

**文件**: `core/ipv4/igmp.c:130-160`

```c
err_t igmp_start(struct netif *netif)
{
    struct igmp_group *group;

    // 分配 allsystems 组 (224.0.0.1)
    group = igmp_lookup_group(netif, IP4_ADDR_ALLSYS);

    // 设置 mac_filter
    netif_igmp_mac_filter(netif, IGMP_ADD_MAC_FILTER, ...);

    // 启动 IGMP
    netif->flags |= NETIF_FLAG_IGMP;
}
```

---

## 5. igmp_joingroup — 加入多播组

**文件**: `core/ipv4/igmp.c:523-575`

```c
err_t igmp_joingroup_netif(struct netif *netif, const ip4_addr_t *groupaddr)
{
    struct igmp_group *group;

    // ============================================
    // Step 1: 查找或创建组
    // ============================================
    group = igmp_lookup_group(netif, groupaddr);
    if (group == NULL) {
        return ERR_MEM;  // 内存不足
    }

    // ============================================
    // Step 2: 如果已经在组中，递增引用
    // ============================================
    if (group->group_state != IGMP_GROUP_NON_MEMBER) {
        group->use++;
        return ERR_OK;
    }

    // ============================================
    // Step 3: 加入新组
    // ============================================
    // 设置 MAC 过滤器
    netif_igmp_mac_filter(netif, IGMP_ADD_MAC_FILTER, groupaddr);

    // 设置状态为 PENDING
    group->group_state = IGMP_GROUP_PENDING_MEMBER;
    group->use = 1;

    // 启动报告定时器
    igmp_start_timer(group, IGMP_JOIN_DELAYING_MEMBER_TMR);

    // 发送 IGMP Report
    igmp_send_report(group);

    return ERR_OK;
}
```

---

## 6. igmp_leavegroup — 离开多播组

**文件**: `core/ipv4/igmp.c:621-680`

```c
err_t igmp_leavegroup_netif(struct netif *netif, const ip4_addr_t *groupaddr)
{
    struct igmp_group *group;

    // 查找组
    group = igmp_lookfor_group(netif, groupaddr);
    if (group == NULL) {
        return ERR_OK;  // 不在组中
    }

    // ============================================
    // Step 1: 递减引用计数
    // ============================================
    group->use--;
    if (group->use > 0) {
        return ERR_OK;  // 还有其他成员
    }

    // ============================================
    // Step 2: 如果是最后报告者，发送 Leave
    // ============================================
    if (group->last_reporter_flag) {
        igmp_send_leave(group);
    }

    // ============================================
    // Step 3: 删除 MAC 过滤器
    // ============================================
    netif_igmp_mac_filter(netif, IGMP_DEL_MAC_FILTER, groupaddr);

    // ============================================
    // Step 4: 设置状态为 NON_MEMBER
    // ============================================
    group->group_state = IGMP_GROUP_NON_MEMBER;

    return ERR_OK;
}
```

---

## 7. igmp_input — 处理接收的 IGMP 消息

**文件**: `core/ipv4/igmp.c:361-500`

```c
void igmp_input(struct pbuf *p, struct netif *inp, const ip4_addr_t *dest)
{
    struct igmp_msg *igmp;
    struct igmp_group *group;

    // ============================================
    // Step 1: 校验长度和 Checksum
    // ============================================
    if (p->len < IGMP_MINLEN) goto lenerr;
    igmp = (struct igmp_msg *)p->payload;
    if (inet_chksum(igmp, p->len)) goto chkerr;

    // 查找组
    group = igmp_lookfor_group(inp, dest);

    switch (igmp->igmp_msgtype) {
        // ============================================
        // 成员查询
        // ============================================
        case IGMP_MEMB_QUERY:
            if (ip4_addr_cmp(dest, &allsystems) &&
                ip4_addr_isany(&igmp->igmp_group_address)) {
                // General Query: 延迟响应所有组
                for (group = netif_igmp_data(inp)->next; group; group = group->next) {
                    igmp_delaying_member(group, igmp->igmp_maxresp);
                }
            } else {
                // Specific Query: 延迟响应特定组
                if (group != NULL) {
                    igmp_delaying_member(group, igmp->igmp_maxresp);
                }
            }
            break;

        // ============================================
        // 成员报告 (v2)
        // ============================================
        case IGMP_V2_MEMB_REPORT:
            if (group->group_state == IGMP_GROUP_DELAYING_MEMBER) {
                // 停止本地报告定时器
                igmp_stop_timer(group);
            }
            group->last_reporter_flag = 0;
            break;

        // ============================================
        // 离开组 (v2)
        // ============================================
        case IGMP_LEAVE_GROUP:
            if (group->group_state == IGMP_GROUP_DELAYING_MEMBER) {
                // 发送特定组查询
                igmp_send_group_leave_query(group);
            }
            break;
    }

    pbuf_free(p);
    return;
}
```

---

## 8. 多播数据接收

### 8.1 MAC 过滤器

当 netif 启用 IGMP 时，需要设置 MAC 过滤器：

```c
// 加入多播组时
netif_igmp_mac_filter(netif, IGMP_ADD_MAC_FILTER, groupaddr);
// 设置 NIC 接受该多播 MAC 地址的帧

// 离开多播组时
netif_igmp_mac_filter(netif, IGMP_DEL_MAC_FILTER, groupaddr);
// 移除该 MAC 地址的过滤
```

### 8.2 IP 层多播路由

```c
// ip4_input 中处理多播
if (ip4_addr_ismulticast(dest)) {
    // 查找 IGMP 组
    if (igmp_lookfor_group(inp, dest) != NULL) {
        // 在组中，接收数据包
        return raw_udp_input(p, inp);
    } else {
        // 不在组中，丢弃
        pbuf_free(p);
        return ERR_OK;
    }
}
```

---

## 9. IGMP Timer

### 9.1 定时器超时

```c
void igmp_tmr(void)
{
    struct igmp_group *group;

    // 遍历所有 netif 的组
    NETIF_FOREACH(netif) {
        for (group = netif_igmp_data(netif); group; group = group->next) {
            if (group->timer > 0) {
                group->timer--;
                if (group->timer == 0) {
                    // 定时器到期，发送报告
                    igmp_send_report(group);
                }
            }
        }
    }
}
```

### 9.2 定时器参数

```c
#define IGMP_JOIN_DELAYING_MEMBER_TMR  1  // 加入后延迟 1 秒报告
#define IGMP_JOIN_DELAYING_MEMBER_TMR  1  // 响应查询的延迟
```

---

## 10. 总结

### 10.1 IGMP 流程

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
    ├─► netif_igmp_mac_filter(DEL)
    └─► 设置 group_state = NON_MEMBER
```

### 10.2 与 lwIP 集成

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

### 10.3 关键设计

1. **组状态机**: NON_MEMBER → PENDING → DELAYING → NON_MEMBER
2. **MAC 过滤器**: 通知 NIC 接受特定多播 MAC
3. **定时器**: 延迟响应避免 IGMP 风暴
4. **last_reporter**: 只有最后报告者发送 Leave
