---
type: source
source-type: github
created: 2026-05-27
title: "VDF Apps — NIO 车辆分布式框架 Recipes"
date: 2026-05-27
size: medium
path: raw/vdf/apps/recipes
summary: "NIO VDF应用层recipes：evm-report(GB/T 32960国标)、iot-gateway(物物模型/地锁/充电桩/KeyFob)、oam-service(远程救援)、traffic-manager、data-collection、diagnostic-lib、persistent-data-lib等"
tags: [vdf, nio, vehicle, telematics, gb/t-32960, mqtt, can, uds, obd]
sources: []
---

# VDF Apps — NIO 车辆分布式框架 Recipes

> NIO Vehicle Distributed Framework 应用层服务集

## 概述

VDF (Vehicle Distributed Framework) 是 NIO 车辆应用层框架，包含多个 recipe（服务组件），通过 CAN/MQTT/D-Bus 等协议与车辆硬件和 TSP 云端通信。

## 核心服务

### 数据上报类

| 服务 | 说明 | 协议 |
|------|------|------|
| **evm-report** | GB/T 32960 新能源汽车数据上传 | MQTT |
| **event_report** | 事件数据上报 | MQTT |
| **data-collection** | CAN 信号采集 (MT/PMT 模式) | CAN |

### 通信控制类

| 服务 | 说明 | 协议 |
|------|------|------|
| **iot-gateway** | 物物模型网关（车控/地锁/充电桩/KeyFob） | MQTT + nio-capi |
| **comm-control-service** | 通信链路管理（断线重连/心跳保活） | MQTT/HTTP |
| **traffic-manager** | 流量管理、QoS 控制 | - |

### 诊断类

| 服务 | 说明 | 协议 |
|------|------|------|
| **diagnostic-lib** | UDS/OBD 诊断库 | CAN |
| **vca-uds-library** | VCA UDS 诊断协议库 | CAN |
| **oam-service** | 远程救援和服务功能 | MQTT + D-Bus |

### 存储与配置类

| 服务 | 说明 |
|------|------|
| **persistent-data-lib** | SQLite 持久化存储 |
| **nservice-config-agent** | 云端配置下发代理 |
| **vdf-common** | 公共库 |

### 工具与监控类

| 服务 | 说明 |
|------|------|
| **switch-monitor** | 以太网交换机监控 (BBMGR/smitool) |
| **diagnostic-tool** | 诊断工具 |
| **event_report** | 事件上报 |

## 关键依赖

| 依赖 | 用途 |
|------|------|
| **nio-capi** | 车辆 CAN 总线通信 (NT2) |
| **mosquitto** | MQTT broker (NT1) |
| **sqlite3** | 本地数据存储 |
| **D-Bus** | 进程间通信 (NT2) |
| **CommonAPI** | RPC 调用 (NT2) |

## 构建方式

```bash
# 普通构建
niobuild -d build

# 带测试构建
niobuild -d build -e -ebuildtest="on"
```

## 服务架构

```
Vehicle CAN Bus
    │
    ├── iot-gateway (nio-capi) ──► 物物模型
    ├── evm-report ──► GB/T 32960 ──► MQTT ──► TSP
    ├── data-collection ──► CAN 信号采集
    ├── diagnostic-lib ──► UDS/OBD 诊断
    └── oam-service ──► 远程救援
            │
            ├── MQTT ──► TSP Cloud
            ├── D-Bus ──► 进程间通信
            └── nio-capi ──► CAN 总线
```

## 相关页面

- [[vdf-index]] — VDF 模块索引
- [[entities/linux/vdf/vdf-evm-report]] — evm-report 实体
- [[entities/linux/vdf/vdf-iot-gateway]] — iot-gateway 实体
- [[entities/linux/vdf/vdf-oam-service]] — oam-service 实体
- [[entities/linux/vdf/vdf-diagnostic-lib]] — diagnostic-lib 实体
