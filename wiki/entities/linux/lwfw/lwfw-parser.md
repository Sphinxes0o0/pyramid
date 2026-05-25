---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Rule Parser

## 定义

LWFW YAML 规则解析器使用 **libyaml** 事件驱动解析，通过状态机处理 YAML 事件流，将配置转换为内存中的规则链表结构。

## 解析器架构

```c
typedef struct parser_state {
  lwfw_policy_config_t *fw_cfg;
  lwfw_rule_config_t *curr_rule;
  lwfw_rule_config_t *in_rules;
  lwfw_rule_config_t *out_rules;
  int in_rule_idx;
  int out_rule_idx;
  int state;  // 解析状态机当前状态
  bool process_default_ingress;
  bool process_default_egress;
  bool process_from;
  int src_port_index;
  int dst_port_index;
} parser_state_t;
```

## YAML 文件结构

```yaml
version: 1.0
revision: 1
name: "default_policy"
filter_mode: 1          # 0=DEFAULT(LIST), 1=TREE
default_ingress:
  action: deny
  event: false
ingress_rules:
  - index: 1
    priority: 1
    action: allow
    from:
      interface: "eth0"
      ip:
        source: "192.168.1.0/24"
      l4:
        protocol: tcp
        port_list: [80, 443]
```

## 解析流程

```
lwfw_config_init(filename)
  ├─ yaml_parser_init(&parser)
  ├─ parse_document()
  │     └─ parse_content()
  │           ├─ "version:" → 读版本号
  │           ├─ "filter_mode:" → 读过滤模式
  │           ├─ "default_ingress:" → PARSER_STATE_DEFAULT_INGRESS
  │           └─ "ingress_rules:" → PARSER_STATE_INGRESS_RULES
  └─ yaml_parser_delete(&parser)
```

## 支持的匹配字段

| 层级 | 字段 | 说明 |
|------|------|------|
| L2 | MAC Address | 源/目的 MAC + 掩码 |
| L2 | VLAN | VLAN ID 精确匹配 |
| L2 | EtherType | 以太网类型 |
| L3 | IP | 源/目的 IP + CIDR 前缀 |
| L3 | Protocol | TCP/UDP/ICMP/IP |
| L4 | Port | 源/目的端口 (范围或列表) |

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 静态解析器状态 | P0 | 11个 static 变量保存状态，多线程不安全 |
| sscanf 安全 | P1 | 不校验 octet 范围 (0-255) |
| IP 前缀解析 | P2 | 保存前缀数字而非掩码，运行时计算 |

## 相关概念

- [[entities/linux/lwfw/lwfw-config-parsing]] — 配置解析汇总
- [[entities/linux/lwfw/lwfw-parser-concurrency]] — 并发安全问题
- [[entities/linux/lwfw/lwfw-architecture]] — 策略数据结构
