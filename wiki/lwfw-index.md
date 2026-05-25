---
type: index
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Firewall — Module Index

> SafeOS lwIP Lightweight Firewall 分析 (27 篇文档)

## Architecture (3)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-architecture]] | LWFW 整体架构：分层、系统组件、编译选项 |
| [[entities/linux/lwfw/lwfw-core-filtering]] | 核心过滤逻辑：Ingress/Egress 入口、包解析、速率限制 |
| [[entities/linux/lwfw/lwfw-data-structure]] | 数据结构设计：rule/policy、FIFO、双缓冲、CACHE_ALIGNMENT |

## Filtering & Classification (4)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-classification]] | 5-tuple 分类：L2/L3/L4 匹配、规则链表遍历 |
| [[entities/linux/lwfw/lwfw-filter-flow]] | 规则匹配流程详解：check_rule 各层匹配、动作编码 |
| [[entities/linux/lwfw/lwfw-list-search]] | list_search 线性扫描引擎：cdlist 链表、限速状态机 |
| [[entities/linux/lwfw/lwfw-tree-search]] | tree_search 决策树引擎：Hyperscan 风格、维度分割 |

## Hook & Initialization (2)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-hook-injection]] | Hook 注入点：ip4_input/egress_filter、ops 函数表、初始化顺序 |
| [[entities/linux/lwfw/lwfw-tcpip-thread]] | tcpip_thread 交互：RX 锁机制、LWCT 优先执行 |

## LWCT Connection Tracking (3)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-lwct]] | LWCT 模块总览：连接追踪、五元组哈希表、状态机 |
| [[entities/linux/lwfw/lwfw-lwct-gc-analysis]] | GC 线程深度分析：扫描算法、水位触发、引用计数 |
| [[entities/linux/lwfw/lwfw-lwct-interaction]] | LWFW 与 LWCT 交互：pbuf->_lwct 绑定、ct_state 匹配 |

## Configuration & Parsing (3)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-config-parsing]] | YAML 配置解析：libyaml、状态机、双策略热切换 |
| [[entities/linux/lwfw/lwfw-parser]] | 规则解析器：parser_state、规则字段解析 |
| [[entities/linux/lwfw/lwfw-parser-concurrency]] | 解析器并发安全：11 个 static 状态变量问题 |

## Events & IPC (3)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-notif]] | 事件通知机制：双线程架构、FIFO、限速 |
| [[entities/linux/lwfw/lwfw-ipc-mechanism]] | IPC 机制：共享内存 FIFO、seL4 IPC、内存屏障 |
| [[entities/linux/lwfw/lwfw-stats]] | 统计计数：g_lwfw_stats、stats_ct、stats_agent |

## Agent (2)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-agent]] | lwfw_agent 守护进程：事件消费、IOCTL、进程管理 |
| [[entities/linux/lwfw/lwfw-agent-log-system]] | Agent 日志系统：O(n²) 合并、JSON 格式化、轮转 |

## VLAN (2)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-vlan-interception-flow]] | VLAN 拦截流程：Ingress L2 解析、VLAN tag 提取 |
| [[entities/linux/lwfw/lwfw-vlan-isolation-guide]] | VLAN 隔离配置指南：YAML 示例、部署步骤 |

## Analysis & Optimization (3)

| Entity | Description |
|--------|-------------|
| [[entities/linux/lwfw/lwfw-hotswap-analysis]] | 热切换深度分析：双缓冲、RCU 可行性、深拷贝阻塞时间 |
| [[entities/linux/lwfw/lwfw-rule-matching]] | 规则匹配算法优化：位图索引、快速路径、SIMD |
| [[entities/linux/lwfw/lwfw-optimization]] | 优化建议汇总：P0-P3 问题清单、行动计划 |

## Source Page

- [[sources/safeos-lwfw]] — SafeOS LWFW Firewall Analysis 源摘要页

## Related Indexes

- [[lwip-index]] — lwIP 嵌入式协议栈 (LWFW 依赖 lwIP)
- [[kernel-net-index]] — Linux 内核网络子系统
