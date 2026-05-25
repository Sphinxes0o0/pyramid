---
type: source
source-type: github
created: 2026-05-25
tags: [lwfw, firewall, network, safeos]
title: "SafeOS LWFW Firewall Analysis"
date: 2026-05-25
size: large
path: raw/safeos/lwfw_*.md + raw/safeos/lwfw_analysis/
summary: "SafeOS lwIP LWFW (Lightweight Firewall) 深度分析文档集：架构、hook 注入、规则解析、连接追踪、过滤引擎、事件通知、热切换、VLAN 隔离 (27 篇)"
---

# SafeOS LWFW Firewall Analysis

## 核心内容

LWFW (Lightweight Firewall) 是 SafeOS lwIP 网络栈中的**状态ful 防火墙模块**，支持：

1. **5-tuple 规则匹配**: L2 (VLAN/MAC)、L3 (IP/CIDR)、L4 (Port) 多层匹配
2. **可选连接追踪 (LWCT)**: TCP/UDP/ICMP 状态机，pbuf->_lwct 扩展字段
3. **双过滤引擎**: list_search (O(n)) vs tree_search (O(log n))
4. **共享内存 FIFO + seL4 IPC**: 高效内核→用户态事件传递
5. **VLAN 间隔离**: 基于 VLAN ID 精确匹配的微分段安全策略

## 关键发现

### P0 严重问题
- **位标志冲突**: `SRC_IP_MASK_LEN` (BIT 7) 与 `SRC_L4_PORT_RANGE` 冲突
- **热重载窗口期无防护**: 重载期间 Ingress 防火墙完全失效
- **初始化失败无 return**: lwfw_init 失败后继续执行

### P1 高优先级
- **静态解析器状态**: 11 个 static 变量，多线程不安全
- **深拷贝无错误恢复**: 拷贝失败时策略损坏无法回滚
- **GC 线程退出无重启**: 连接表可能逐渐满

### 架构亮点
- **引擎抽象**: `lwfw_backend_engine_t` 统一接口支持多引擎
- **双缓冲热切换**: active/inactive 策略原子交换
- **pbuf->_lwct 扩展**: 连接状态直接绑定到数据包

## 相关页面

- [[lwfw-index]] — LWFW 模块索引
- [[entities/linux/lwip/lwip-tcpip-thread]] — 过滤执行上下文
- [[kernel-net-index]] — Linux 内核网络子系统
