---
type: entity
tags: [vdf, traffic-manager, qos, bandwidth, vehicle, network]
created: 2026-05-27
sources: []
---

# VDF traffic-manager — 流量管理器

## 定义

traffic-manager 负责车辆网络流量管理，QoS 控制，带宽分配和流量监控，确保关键服务（远程控制、安全）的网络优先级。

## 核心功能

| 功能 | 说明 |
|------|------|
| **流量监控** | 网络流量统计和监控 |
| **QoS 控制** | 服务质量保障 |
| **带宽分配** | 不同服务的带宽分配 |
| **流量整形** | 流量限制和整形 |

## 相关概念

- [[entities/linux/vdf/vdf-comm-control-service]] — 通信控制服务
- [[entities/linux/vdf/vdf-data-collection]] — 数据采集
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/traffic-manager/README.md`
