---
type: entity
tags: [vdf, persistent-data-lib, sqlite, storage, vehicle-data]
created: 2026-05-27
sources: []
---

# VDF persistent-data-lib — 持久化数据存储

## 定义

persistent-data-lib 提供车辆数据的持久化存储能力，基于 SQLite 实现，用于存储车辆配置、故障记录、行驶数据等需要持久化的信息。

## 核心功能

| 功能 | 说明 |
|------|------|
| **数据持久化** | SQLite 本地存储 |
| **配置存储** | 车辆配置参数 |
| **故障记录** | 故障历史数据 |
| **数据查询** | SQL 查询接口 |

## 相关概念

- [[entities/linux/vdf/vdf-data-collection]] — 数据采集
- [[entities/linux/vdf/vdf-evm-report]] — 国标数据上传
- [[vdf-index]] — VDF 模块索引

## 来源详情

- `vdf/apps/recipes/persistent-data-lib/README.md`
