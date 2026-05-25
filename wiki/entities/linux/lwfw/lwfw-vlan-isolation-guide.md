---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW VLAN Isolation Guide

## 定义

LWFW 支持通过 **VLAN ID 精确匹配**实现 VLAN 间通信隔离，适用于 SafeOS NSv 网络栈的微分段安全策略配置。

## 支持的功能

| 功能 | 支持 | 说明 |
|------|------|------|
| VLAN 精确匹配 | YES | `vlan: 100` |
| VLAN + 源 IP 组合 | YES | `vlan: 100` + `from.L3.ipBlock` |
| VLAN + 目标 IP 组合 | YES | `vlan: 100` + `to.L3.ipBlock` |
| VLAN + 协议组合 | YES | `vlan: 100` + `protocol: tcp` |
| VLAN + 端口组合 | YES | `vlan: 100` + `to.L4.list: [80,443]` |
| CIDR 前缀匹配 | YES | `prefix: 24` |

## 不支持的功能

| 功能 | 状态 | 替代方案 |
|------|------|----------|
| VLAN 范围 | NO | 为每个 VLAN 写单独规则 |
| VLAN 掩码 | NO | 使用规则组 |
| 否定匹配 | NO | 使用默认拒绝策略 |
| 规则继承 | NO | 每条规则独立配置 |
| 时间条件 | NO | 依赖外部定时任务 |

## 配置示例

```yaml
apiVersion: nt3.networking.firewall/v1
kind: NetworkPolicy
metadata:
  name: enterprise-vlan-isolation
spec:
  default:
    ingress:
       action: deny    # 默认拒绝
       event: true
       log: true

  ingress:
    state: enable
    rules:
    # 规则 1: 允许 VLAN 100 内部通信
    - index: 1
      name: allow-vlan100-internal
      state: enable
      action: allow
      vlan: 100
      from:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 16

    # 规则 2: 允许 VLAN 200 内部通信
    - index: 2
      name: allow-vlan200-internal
      action: allow
      vlan: 200
      from:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 16
```

## 部署与验证

```bash
# 触发热重载
lwfwcfg reload

# 查看防火墙状态
lwfwcfg show

# 测试跨 VLAN 通信（应被拦截）
ping 192.168.200.1  # 应被拒绝
```

## 编译要求

```c
#define LWFW_ADVANCED_FUNC_L2 1  // 启用 L2 (VLAN/MAC) 过滤
#define NIO_LWIP_LWFW 1           // 启用 lwfw 功能
```

## 注意事项

1. **编译开关**: 必须启用 `LWFW_ADVANCED_FUNC_L2` 才能使用 VLAN 过滤
2. **热切换**: 配置更新后自动热切换，无需重启服务
3. **规则顺序**: 规则按 `index` 小到大逐条匹配，确保允许规则在阻断规则之前
4. **双向配置**: 需要同时配置 Ingress 和 Egress 规则才能完全隔离

## 相关概念

- [[entities/linux/lwfw/lwfw-vlan-interception-flow]] — VLAN 拦截流程
- [[entities/linux/lwfw/lwfw-config-parsing]] — YAML 配置解析
- [[entities/linux/lwfw/lwfw-hotswap-analysis]] — 热切换机制
