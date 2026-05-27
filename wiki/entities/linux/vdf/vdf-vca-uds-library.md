---
type: entity
tags: [vdf, vca-uds-library, uds, diagnostic, vehicle]
created: 2026-05-27
sources: []
---

# VDF vca-uds-library — VCA UDS 诊断库

## 定义

vca-uds-library 是 NIO VCA (Vehicle Control Assistant) 的 UDS (Unified Diagnostic Services) 诊断协议库，提供标准 UDS 诊断服务实现。

## 核心功能

| 功能 | 说明 |
|------|------|
| **UDS 协议** | ISO 14229 统一诊断服务 |
| **诊断会话** | Session Control |
| **故障码** | DTC (Diagnostic Trouble Code) 管理 |
| **例程控制** | Routine Control |
| **数据传输** | Read/Write Memory By Address |

## 相关概念

- [[entities/linux/vdf/vdf-diagnostic-lib]] — 诊断库
- [[entities/linux/vdf/vdf-data-collection]] — 数据采集
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/vca-uds-library/README.md`
