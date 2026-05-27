---
type: index
tags: [vdf, nio, vehicle, telematics, embedded]
created: 2026-05-27
sources: [vdf-apps, vdf-tools, vdf-sel4]
---

# VDF — NIO Vehicle Distributed Framework Index

> NIO 车辆分布式框架应用层服务索引 (Batch H: 2026-05-27)

## Entity Pages

### 数据上报类
| Entity | Description |
|--------|-------------|
| [[entities/linux/vdf/vdf-evm-report]] | GB/T 32960 新能源汽车数据上传：整车数据、驱动电机、燃料电池、车辆位置、极值数据、报警数据 |
| [[entities/linux/vdf/vdf-data-collection]] | CAN 信号采集：MT/PMT 测试模式、日志解析 |
| [[entities/linux/vdf/vdf-event-report]] | 事件数据上报 (event_report recipe) |

### 通信控制类
| Entity | Description |
|--------|-------------|
| [[entities/linux/vdf/vdf-iot-gateway]] | 物物模型网关：车控/车状态 (nio-capi)、地锁、充电桩、KeyFob |
| [[entities/linux/vdf/vdf-comm-control-service]] | 通信链路管理：MQTT/HTTP 断线重连、心跳保活 |
| [[entities/linux/vdf/vdf-traffic-manager]] | 流量管理：QoS 控制、带宽分配、流量监控 |

### 诊断类
| Entity | Description |
|--------|-------------|
| [[entities/linux/vdf/vdf-diagnostic-lib]] | UDS/OBD 诊断库：故障码读取、实时数据、PID |
| [[entities/linux/vdf/vdf-vca-uds-library]] | VCA UDS 诊断协议库：ISO 14229 会话控制、故障码管理 |
| [[entities/linux/vdf/vdf-oam-service]] | 远程救援服务：远程诊断、D-Bus/CommonAPI、MQTT |

### 存储与配置类
| Entity | Description |
|--------|-------------|
| [[entities/linux/vdf/vdf-persistent-data-lib]] | SQLite 持久化存储：配置、故障记录、行驶数据 |
| [[entities/linux/vdf/vdf-nservice-config-agent]] | 云端配置代理：TSP 配置下发、动态更新 |

### 工具与监控类
| Entity | Description |
|--------|-------------|
| [[entities/linux/vdf/vdf-switch-monitor]] | 以太网交换机监控：BBMGR、smitool、VLAN 管理 |

## Source Pages

- [[sources/vdf-apps]] — VDF Apps 概览 (evm-report/iot-gateway/oam-service 等)
- [[sources/vdf-tools]] — VDF Tools 工具集 (switch-monitor)
- [[sources/vdf-sel4]] — VDF seL4 构建与刷版指南

## VDF 架构

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
            ├── D-Bus ──► 进程间通信 (NT2)
            └── nio-capi ──► CAN 总线
```

## 构建方式

```bash
niobuild -d build           # 普通构建
niobuild -d build -e -ebuildtest="on"  # UT/MT 构建
```

## 相关索引

- [[safeos-index]] — SafeOS NSv 架构索引
- [[lwip-index]] — lwIP 嵌入式协议栈索引
- [[lwfw-index]] — LWFW 防火墙索引
