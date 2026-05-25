---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Parser Concurrency

## 定义

LWFW YAML 解析器存在**严重的并发安全问题**：11 个 `static` 局部变量用于保存解析状态机上下文，多线程并发调用时会相互覆盖，导致解析结果不确定。

## 静态状态变量列表

| 函数 | 静态变量 | 影响 |
|------|----------|------|
| `parse_spec_default_xgress` | `spec_default_xgress_state` | 影响 default ingress/egress 解析 |
| `parse_spec_parameters` | `spec_default_parameters_state` | 影响参数解析 |
| `parse_spec_xgress_rule_l3_ipblock` | `spec_xgress_rule_fromto_l3_ipblock_state` | 影响 L3 IP 解析 |
| `parse_spec_xgress_rule_l4_range` | `spec_xgress_rule_fromto_l4_range_state` | 影响 L4 端口范围解析 |
| `parse_spec_xgress_rule_rate_limit` | `spec_xgress_rule_rate_limit_state` | 影响限速解析 |
| `consume_event` (4个) | `spec_xgress_rule_state` 等 | 影响所有规则解析 |

## 问题代码示例

```c
// lwfw_parser.c:775
static lwfw_parser_err_t parse_spec_xgress_rule_l3_ipblock(...) {
    static enum spec_xgress_rule_fromto_l3_ipblock_state
        spec_xgress_rule_fromto_l3_ipblock_state = SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_INIT;
    // ...
}

// lwfw_parser.c:1153-1156
static lwfw_parser_err_t consume_event(...) {
    static enum spec_xgress_rule_state spec_xgress_rule_state = SPEC_XGRESS_RULE_STATE_INIT;
    static enum spec_xgress_rule_fromto_state spec_xgress_rule_fromto_state = ...;
    // ...
}
```

## 当前调用场景

```c
// lwfw_init() - 单线程调用，暂无问题
lwfw_init()
  └─ lwfw_manifest_parse(cfg_path)  // 单线程

// lwfw_notif.c - 通知线程可能触发重载
lwfw_notification_timer_thread()
  └─ if (cfg_in_reloading)
        └─ lwfw_config_reload_manifest(cfg_path)
```

## sscanf 安全问题

```c
// lwfw_parser.c:827
int result = sscanf(data, "%d.%d.%d.%d", &ip[0], &ip[1], &ip[2], &ip[3]);
if (result != 4) return LWFW_PARSER_ERR_INVALID_IP;
// 没有验证每个 octet 的范围 (0-255)！
```

建议修复:
```c
// 使用 inet_pton
struct in_addr addr;
if (inet_pton(AF_INET, data, &addr) != 1)
    return LWFW_PARSER_ERR_INVALID_IP;
```

## 修复方案

将所有静态状态移入 `parser_state` 结构体:

```c
struct parser_state {
    // 现有字段
    enum state state;
    lwfw_policy_config_t *fw_cfg;
    // ...

    // 新增子状态字段
    enum spec_xgress_state spec_xgress_state;
    enum spec_xgress_rule_fromto_l3_ipblock_state l3_ipblock_state;
    enum spec_xgress_rule_fromto_l4_range_state l4_range_state;
    // ...
};
```

## 相关概念

- [[entities/linux/lwfw/lwfw-parser]] — 解析器整体
- [[entities/linux/lwfw/lwfw-optimization]] — P0 问题汇总
- [[entities/linux/lwfw/lwfw-hotswap-analysis]] — 热切换触发重载
