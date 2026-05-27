---
type: entity
tags: [vdf, evm-report, gb/t-32960, telematics, vehicle, protocol]
created: 2026-05-27
sources: []
---

# VDF evm-report — 国标新能源汽车数据上传

## 定义

evm-report 是基于 GB/T 32960 国标的新能源汽车实时数据上传服务，将车辆行驶数据（位置、电池、报警等）通过 MQTT 协议上传至 TSP 后台。

## 核心信息

| 属性 | 值 |
|------|-----|
| **文档** | NDTC: ndtc.nioint.com/#/siManagement/list/service_detail?serviceId=600 |
| **协议** | GB/T 32960 新能源汽车远程服务与管理系统 |
| **传输** | MQTT over TLS |
| **存储** | SQLite |
| **构建** | `niobuild -d build` |

## GB/T 32960 数据类型

| 数据类型 | 说明 |
|----------|------|
| 整车数据 | 车速、里程、SOC、运行模式 |
| 驱动电机数据 | 电机转速、温度、电压、电流 |
| 燃料电池数据 | 电压、电流、氢耗量 |
| 发动机数据 | 转速、油耗、扭矩 |
| 车辆位置 | 经纬度、海拔 |
| 极值数据 | 最高/最低温度、电压、电流 |
| 报警数据 | 故障码、级别、描述 |

## 架构

```
Vehicle CAN Bus
    │
    ▼
evm-report (niobuild)
    │
    ├─► 数据解析 (GB/T 32960 协议)
    ├─► MQTT Publish → TSP Backend
    └─► 本地 SQLite 缓存
```

## 相关概念

- [[entities/linux/vdf/vdf-iot-gateway]] — 物物模型网关
- [[entities/linux/vdf/vdf-data-collection]] — 数据采集
- [[entities/linux/vdf/vdf-diagnostic-lib]] — 诊断库
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/evm-report/README.md`
