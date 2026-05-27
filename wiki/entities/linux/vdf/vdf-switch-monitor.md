---
type: entity
tags: [vdf, switch-monitor, ethernet, switch, vehicle-network]
created: 2026-05-27
sources: []
---

# VDF switch-monitor — 交换机监控

## 定义

switch-monitor 负责监控和管理车辆以太网交换机，支持端口状态监控、VLAN 配置、流量统计等交换器管理功能。

## 核心功能

| 功能 | 说明 |
|------|------|
| **端口监控** | 以太网端口状态和流量 |
| **VLAN 管理** | 802.1Q VLAN 配置 |
| **流量统计** | 端口流量计数 |
| **告警处理** | 链路故障告警 |

## 组件

| 组件 | 说明 |
|------|------|
| `switch-monitor/BBMGR` | 交换机管理工具 |
| `switch-monitor/smitool` | SMI-T 工具 |
| `switch-monitor-mpu` | MPU 版本 |

## 相关概念

- [[entities/linux/vdf/vdf-traffic-manager]] — 流量管理器
- [[entities/linux/vdf/vdf-comm-control-service]] — 通信控制服务
- [[entities/linux/vdf/vdf-iot-gateway]] — 物物网关
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/tools/switch-monitor/README.md`
