---
type: entity
tags: [vdf, nservice-config-agent, configuration, agent, vehicle]
created: 2026-05-27
sources: []
---

# VDF nservice-config-agent — 配置代理

## 定义

nservice-config-agent 是 NIO 车辆服务配置代理，负责从云端 TSP 获取并管理车辆配置参数，支持配置的动态下发和更新。

## 核心功能

| 功能 | 说明 |
|------|------|
| **配置获取** | 从 TSP 云端拉取配置 |
| **配置分发** | 分发给各服务组件 |
| **动态更新** | 支持运行时配置热更新 |
| **配置缓存** | 本地缓存配置副本 |

## 相关概念

- [[entities/linux/vdf/vdf-iot-gateway]] — 物物网关
- [[entities/linux/vdf/vdf-comm-control-service]] — 通信控制服务
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/nservice-config-agent/README.md`
