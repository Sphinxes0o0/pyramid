---
type: entity
tags: [vdf, oam-service, remote-rescue, dbus, commonapi, mosquitto]
created: 2026-05-27
sources: []
---

# VDF oam-service — 远程救援和服务

## 定义

oam-service (Operations, Administration & Maintenance) 提供远程救援和服务功能，支持远程诊断、远程控制等车联网运维能力。

## 核心功能

| 功能 | 说明 |
|------|------|
| **远程救援** | 远程故障诊断和恢复 |
| **服务功能** | OAM 运维管理 |
| **D-Bus IPC** | NT2 平台进程间通信 |
| **CommonAPI** | CommonAPI 调用 |
| **MQTT** | 与云端通信 |

## 依赖

| 依赖 | 说明 |
|------|------|
| mosquitto | MQTT broker (NT1) |
| mosquittopp-dev | MQTT C++ 客户端 |
| sqlite3 | 本地数据存储 |
| Dbus | NT2 平台 IPC |
| CommonAPI | NT2 平台 RPC |
| nio-capi | 车辆 CAN 总线通信 |

## 构建

```bash
# Lion (NT1)
make clean && make all

# Tiger (NT2)
mkdir build && cd build
cmake -DHOST_DIR=$HOST_DIR -DCMAKE_TOOLCHAIN_FILE=$CMAKE_TOOLCHAIN_FILE ..
make
```

## 相关概念

- [[entities/linux/vdf/vdf-evm-report]] — 国标数据上传
- [[entities/linux/vdf/vdf-iot-gateway]] — 物物网关
- [[entities/linux/vdf/vdf-comm-control-service]] — 通信控制服务
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/oam-service/README.md`
