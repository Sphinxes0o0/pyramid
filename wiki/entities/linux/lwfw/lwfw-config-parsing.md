---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Configuration Parsing

## 定义

LWFW 使用 **YAML 格式**配置文件定义防火墙策略和规则，通过 libyaml 事件驱动解析器加载到内存链表结构。

## 配置结构

```yaml
apiVersion: nt3.networking.firewall/v1
kind: NetworkPolicy
spec:
  default:
    ingress:
       action: deny
       event: false
       log: false
  ingress:
    state: enable
    rules:
    - index: 1
      name: sample-rule
      action: deny
      protocol: ip
      from:
        L3:
          ipBlock:
            ip: 172.20.0.0
            prefix: 16
        L4:
          list: [80, 443]
```

## 解析流程

```
YAML 文件
    │
    ▼
yaml_parser_initialize()
    │
    ▼
yaml_parser_set_input_file/string()
    │
    ▼
事件循环:
  yaml_parser_parse()
    │
    ▼
consume_event()  // 状态机处理
    │
    ├─► STATE_SECTION: 识别顶层字段
    ├─► STATE_SPEC: 解析 spec
    ├─► STATE_INGRESS/EGRESS: 解析规则
    └─► STATE_RULE: 解析单条规则
    │
    ▼
verify_lwfw_policy_config()  // 验证
    │
    ▼
lwfw_init_rule_tables()  // 构建内存链表
```

## 关键设计

1. **YAML 解析**: 使用 libyaml 事件驱动解析
2. **状态机**: 管理解析上下文，支持嵌套结构
3. **双策略**: active/inactive 策略支持原子切换
4. **内存池**: 规则使用 MEMP_LWFW_RULE 内存池分配
5. **优先级排序**: 规则按 priority 字段排序插入链表

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 静态 parser state | P0 | 多线程并发调用时状态机会冲突 |
| sscanf 安全 | P1 | 不校验 octet 范围 (0-255) |
| IP 前缀解析 | P2 | 保存的是前缀数字不是掩码 |

## 相关概念

- [[entities/linux/lwfw/lwfw-architecture]] — 策略数据结构
- [[entities/linux/lwfw/lwfw-core-filtering]] — 规则表初始化
- [[entities/linux/lwfw/lwfw-parser-concurrency]] — 并发安全问题详解
- [[entities/linux/lwfw/lwfw-hotswap-analysis]] — 运行时热重载
