---
type: entity
tags: [vdf, iot-gateway, mqtt, telematics, can, obd, charging, keyfob]
created: 2026-05-27
sources: []
---

# VDF iot-gateway — 物联网关

## 定义

iot-gateway 是 NIO 车辆物联网关服务，负责车辆与云端 TSP 之间的通信，支持物物模型（车控、车状态）、地锁、充电桩、KeyFob 等多种 IoT 设备的协议转换和数据上报。

## 核心功能

| 功能 | 说明 |
|------|------|
| **物物模型** | 车辆控制、状态订阅 (via NIO nio-capi) |
| **地锁控制** | 停车场地锁的远程控制 |
| **充电桩通信** | 充电桩状态监控和控制 |
| **KeyFob** | 智能钥匙信号处理 |

## 协议支持

| 协议 | 说明 |
|------|------|
| MQTT | 与 TSP 云端通信 |
| NIO nio-capi | 车辆 CAN 总线通信 |
| HTTP/REST | 内部服务通信 |

## 架构

```
Vehicle CAN Bus
    │
    ▼
iot-gateway
    ├─► nio-capi (车控/车状态)
    ├─► 地锁控制
    ├─► 充电桩通信
    └─► KeyFob 处理
            │
            ▼
        MQTT → TSP Cloud
```

## 相关概念

- [[entities/linux/vdf/vdf-evm-report]] — 国标数据上传
- [[entities/linux/vdf/vdf-data-collection]] — 数据采集
- [[entities/linux/vdf/vdf-comm-control-service]] — 通信控制服务
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/iot-gateway/README.md`
