---
type: entity
tags: [vdf, diagnostic-lib, uds, obd, can, vehicle, fault-code]
created: 2026-05-27
sources: []
---

# VDF diagnostic-lib — 诊断库

## 定义

diagnostic-lib 提供车辆诊断功能，支持 UDS (Unified Diagnostic Services) 和 OBD (On-board Diagnostics) 协议，用于读取故障码、清除故障码、读取实时数据等。

## 核心功能

| 功能 | 说明 |
|------|------|
| **UDS 诊断** | ISO 14229 统一诊断服务 |
| **OBD 诊断** | OBD-II 车载诊断 |
| **故障码读取** | DTC (Diagnostic Trouble Code) |
| **实时数据** | PID (Parameter ID) 实时数据流 |
| **CAN 通信** | 车辆 CAN 总线交互 |

## 相关概念

- [[entities/linux/vdf/vdf-data-collection]] — 数据采集
- [[entities/linux/vdf/vdf-vca-uds-library]] — VCA UDS 库
- [[entities/linux/vdf/vdf-evm-report]] — 国标数据上传
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/diagnostic-lib/README.md`
