---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Filter Flow

## 定义

LWFW 规则匹配流程详解，覆盖从数据包解析到 L2/L3/L4 各层匹配、事件生成、最终决策输出的完整数据流。

## 完整过滤流程

```
数据包进入
    │
    ▼
ip4_filter_dispatch_incoming/outgoing()
    │ 检查规则表 state
    ▼
ip4_filter()
    │
    ├─ lwct 未跟踪兜底检查
    │
    ├─ lwfw_pkt_info_constructor()
    │     ├─ 提取接口名
    │     ├─ L2 解析 (VLAN/MAC/EtherType)
    │     ├─ L3 解析 (IP/Proto)
    │     └─ L4 解析 (Port)
    │
    ├─ filter_engine->do_filter()
    │     │
    │     └─ list_search_do_filter() 或 tree_search_do_filter()
    │           │
    │           └─ 遍历规则，调用 check_rule()
    │                 ├─ check_lwfw_l2_info()
    │                 ├─ check_lwfw_l3_info()
    │                 └─ check_lwfw_l4_info()
    │
    ├─ 如匹配且需事件 → lwfw_generate_secure_event()
    │
    └─ return action (ALLOW/DENY/EVENT)
```

## L2 匹配详情

```c
// EtherType 匹配 (精确)
if ((rule->flags & LWFW_RULE_FLAGS_ETHER_TYPE) &&
    rule_l2_info->ether_type != packet_info->ether_type)
  goto out;

// VLAN 匹配 (精确)
if ((rule->flags & LWFW_RULE_FLAGS_VLAN) &&
    rule_l2_info->vlan != packet_info->vlan)
  goto out;

// Src MAC + Mask 匹配
if (rule->flags & LWFW_RULE_FLAGS_SRC_MAC) {
  for (int i = 0; i < ETH_HWADDR_LEN; i++) {
    if ((rule_l2_info->src_mac.mask[i] & packet_info->src_mac[i]) !=
        (rule_l2_info->src_mac.addr[i] & rule_l2_info->src_mac.mask[i]))
      goto out;
  }
}
```

## L3 匹配详情

```c
// Protocol 匹配 (精确)
if ((rule->flags & LWFW_RULE_FLAGS_PROTOCOL) &&
    rule_l3_info->proto != packet_info->proto)
  goto out;

// Src IP + CIDR 掩码
if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK_LEN) {
  uint32_t mask = ~((1UL << (32 - masklen)) - 1);
  if ((packet_info->src_ip & mask) != (rule_l3_info->src_ip.addr & mask))
    goto out;
}
```

## L4 匹配详情

```c
// 范围匹配
src_matched = (src_port >= range[0] && src_port <= range[1]);

// 列表匹配
for (i=0; i<LWFW_MAX_PORT_COUNT; i++) {
  if (port_list[i] == 0) break;
  if (port_list[i] == src_port) { src_matched = 1; break; }
}
```

## 动作编码

```c
typedef enum lwfw_action {
  LWFW_ACTION_CODE_DENY   = BIT(0),  // 丢弃包
  LWFW_ACTION_CODE_EVENT  = BIT(1),  // 上报事件
  LWFW_ACTION_LOGGING_MASK = BIT(2),  // 记录日志
} lwfw_action_t;
```

## 相关概念

- [[entities/linux/lwfw/lwfw-classification]] — 分类入口
- [[entities/linux/lwfw/lwfw-core-filtering]] — 核心过滤
- [[entities/linux/lwfw/lwfw-vlan-interception-flow]] — VLAN 拦截流程
- [[entities/linux/lwfw/lwfw-notif]] — 事件生成与通知
