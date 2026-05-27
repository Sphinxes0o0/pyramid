---
type: entity
tags: [vdf, data-collection, can-signal, vehicle-data, logging]
created: 2026-05-27
sources: []
---

# VDF data-collection — 数据采集

## 定义

data-collection 负责从车辆 CAN 总线采集信号数据，支持 MT (Multi-thread) 和 PT (Process Test) 两种测试模式，用于车辆数据记录和分析。

## 核心功能

| 功能 | 说明 |
|------|------|
| **CAN 信号采集** | 从车辆 CAN 总线采集信号 |
| **MT 模式** | Multi-thread 多线程采集 |
| **PMT 模式** | Process-based Multi-thread |
| **日志解析** | `python tool/log_parser/read_log.py` |

## 构建

```bash
niobuild -d build           # 普通构建
niobuild -d build -e -ebuildtest="on"  # UT/MT 构建
```

## 日志解析

```bash
python tool/log_parser/read_log.py ./2023-10-26.log
```

输出格式：
```
Event:
event_type: b'event_type_0'
event_data: b'event_value_0'

WTI: version: 19
sample_ts: 1698282660695
alarm_signal {
  signal_int {
    sn: "1698282660590"
    name: "100"
    value: 0
  }
}
```

## 相关概念

- [[entities/linux/vdf/vdf-evm-report]] — 国标数据上传
- [[entities/linux/vdf/vdf-diagnostic-lib]] — 诊断库
- [[entities/linux/vdf/vdf-iot-gateway]] — 物物网关
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/data-collection/README.md`
