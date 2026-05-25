---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Architecture

## 定义

LWFW (Lightweight Firewall) 是 SafeOS lwIP 网络栈中的**状态ful 防火墙模块**，通过 hook 注入到 IPv4 输入/输出路径，支持 5-tuple 规则匹配和可选的连接追踪 (LWCT)。

## 系统分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    lwfw_agent (用户态)                       │
│  - 事件处理与日志写入                                         │
│  - JSON 格式化                                               │
│  - 配置文件热重载                                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ seL4 IPC + 共享内存
┌──────────────────────────┴──────────────────────────────────┐
│                    lwfw (内核态 lwIP)                        │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              ip4_filter 过滤入口                          │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  lwfw_pkt_info_constructor()  包信息提取                  │ │
│  │    ├─ L2 解析 (VLAN, MAC, EtherType)                    │ │
│  │    ├─ L3 解析 (IP, Protocol)                            │ │
│  │    └─ L4 解析 (Port)                                    │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  过滤器引擎抽象层                                         │ │
│  │    ├─ list_search_engine (规则<20)                       │ │
│  │    └─ tree_search_engine (规则≥20)                       │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  lwct 连接追踪 (可选)                                     │ │
│  │    ├─ lwct_main_hook()                                  │ │
│  │    ├─ 连接哈希表                                          │ │
│  │    └─ GC 线程                                            │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 核心数据结构

```
lwfw_firewall_t (全局控制结构)
  ├─ ctrl           : 版本、状态、配置路径
  ├─ policy         : lwfw_policy_t* (当前策略)
  ├─ inactive_policy: lwfw_policy_t* (热切换缓冲)
  └─ policy_lock    : sync_mutex_t

lwfw_policy_t (策略结构)
  ├─ rule_tables[2] : 规则表 (Ingress/Egress)
  ├─ filter_engine   : const lwfw_backend_engine_t*
  └─ params          : 策略参数

lwfw_rule_t (单条规则)
  ├─ flags          : 匹配标志位图
  ├─ action         : 动作 (ALLOW/DENY/EVENT)
  ├─ l2 / l3 / l4  : L2/L3/L4 匹配字段
  ├─ ct_state       : 连接状态
  └─ rlimit         : 限速参数
```

## 过滤器引擎抽象

```c
typedef struct lwfw_backend_engine {
  char name[16];
  int (*init)(void *handle, void *data);
  int (*deinit)(void *handle, void *data);
  int (*do_filter)(void *handle, void *data, void* result);
  int (*dump)(void *handle, void *data);
} lwfw_backend_engine_t;
```

| 引擎 | 触发条件 | 数据结构 | 搜索复杂度 |
|------|---------|---------|-----------|
| `list_search_eng` | 规则数 < 20 | cdlist 链表 | O(n) |
| `tree_search_eng` | 规则数 ≥ 20 | hyperscan 树 | O(log n) |

## 编译选项

| 选项 | 默认 | 说明 |
|------|------|------|
| `NIO_LWIP_LWFW` | 需启用 | 全局开关 |
| `NIO_LWIP_LWCT` | 需启用 | 连接追踪支持 |
| `LWFW_ADVANCED_FUNC_L2` | 关闭 | L2 过滤 (VLAN/MAC) |
| `LWFW_TREE_SEARCH_EN` | 开启 | 树搜索引擎 |

## 设计亮点

1. **引擎抽象**: 通过 `lwfw_backend_engine_t` 统一接口，支持多引擎切换
2. **双缓冲热切换**: 规则更新时不影响当前策略
3. **连接追踪集成**: 通过 pbuf->_lwct 扩展字段传递连接状态
4. **共享内存 FIFO**: 内核与用户态高效事件传递

## 设计局限

1. **全局锁竞争**: `policy_lock` 在热切换时持有，可能阻塞包处理
2. **lwct 单实例**: GC 线程退出无重启机制
3. **事件合并 O(n²)**: `lwfw_agent` 中事件合并算法可优化

## 相关概念

- [[entities/linux/lwfw/lwfw-core-filtering]] — 核心过滤逻辑
- [[entities/linux/lwfw/lwfw-hook-injection]] — Hook 注入点
- [[entities/linux/lwfw/lwfw-data-structure]] — 数据结构设计
- [[entities/linux/lwfw/lwfw-list-search]] — 线性扫描引擎
- [[entities/linux/lwfw/lwfw-tree-search]] — 树搜索引擎
