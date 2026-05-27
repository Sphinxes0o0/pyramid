---
type: entity
tags: [vdf, comm-control-service, communication, control, vehicle]
created: 2026-05-27
sources: []
---

# VDF comm-control-service — 通信控制服务

## 定义

comm-control-service 负责车辆通信控制，管理车辆与云端的通信链路，支持多协议（MQTT、HTTP）、断线重连、心跳保活等通信管理功能。

## 核心功能

| 功能 | 说明 |
|------|------|
| **链路管理** | MQTT/HTTP 连接管理 |
| **断线重连** | 自动重连机制 |
| **心跳保活** | 链路存活检测 |
| **多协议支持** | MQTT、HTTP 等 |
| **MT/UT 测试** | 支持多线程和单元测试 |

## 相关概念

- [[entities/linux/vdf/vdf-iot-gateway]] — 物物网关
- [[entities/linux/vdf/vdf-traffic-manager]] — 流量管理器
- [[entities/linux/vdf/vdf-evm-report]] — 国标数据上传
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/comm-control-service/README.md`
