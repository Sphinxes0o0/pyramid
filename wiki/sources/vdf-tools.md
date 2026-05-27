---
type: source
source-type: github
created: 2026-05-27
title: "VDF Tools — 车辆工具集"
date: 2026-05-27
size: small
path: raw/vdf/tools
summary: "VDF工具集：switch-monitor(以太网交换机监控/BBMGR/smitool)、switch-monitor-mpu(MPU版本)"
tags: [vdf, switch-monitor, ethernet, switch, vehicle-network, smi-t]
sources: []
---

# VDF Tools — 车辆工具集

## switch-monitor — 以太网交换机监控

### 概述

switch-monitor 负责监控和管理车辆以太网交换机，支持端口状态监控、VLAN 配置、流量统计等。

### 组件

| 组件 | 说明 |
|------|------|
| `BBMGR` | 交换机管理工具 |
| `smitool` | SMI-T (Server Management Interface Technology) 工具 |
| `switch-monitor-mpu` | MPU 平台版本 |
| `switch-monitor-mpu/BBMGR` | MPU 交换机管理 |
| `switch-monitor-mpu/smitool` | MPU SMI-T 工具 |

### SMI-T 协议

SMI-T (Server Management Interface Technology) 是用于管理网络交换机的标准协议，通常基于 UDP/TCP 端口 161。

## 相关页面

- [[vdf-index]] — VDF 模块索引
- [[entities/linux/vdf/vdf-switch-monitor]] — switch-monitor 实体
