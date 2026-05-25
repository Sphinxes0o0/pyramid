# SafeOS lwIP 防火墙实现分析文档

> 分析日期: 2026/04/22
> 代码版本: release/vsel4.01.04.04

---

## 文档索引

| 文档 | 描述 |
|------|------|
| [lwfw_architecture.md](lwfw_architecture.md) | **lwfw 整体架构分析**：模块分层、编译选项、数据流概览 |
| [lwfw_core_filtering.md](lwfw_core_filtering.md) | lwfw 核心过滤逻辑：ip4_filter、check_rule、list_search、速率限制、热切换 |
| [lwfw_filter_flow.md](lwfw_filter_flow.md) | 规则匹配流程详解：包解析、L2/L3/L4 匹配、事件生成 |
| [lwfw_parser.md](lwfw_parser.md) | YAML 规则解析器：配置格式、状态机、参数验证 |
| [lwfw_notif.md](lwfw_notif.md) | 事件通知机制：共享内存 FIFO、双线程架构、IPC 通信 |
| [lwfw_lwct.md](lwfw_lwct.md) | 连接跟踪模块：TCP/UDP/ICMP 状态机、哈希表、GC 线程 |
| [lwfw_lwct_gc_analysis.md](lwfw_lwct_gc_analysis.md) | lwct GC 线程深度分析：扫描算法、水位触发、引用计数 |
| [lwfw_lwct_interaction.md](lwfw_lwct_interaction.md) | lwfw 与 lwct 交互：pbuf->_lwct 扩展、连接状态绑定 |
| [lwfw_tree_search.md](lwfw_tree_search.md) | 树搜索过滤引擎：Hyperscan 风格决策树、规则编译、搜索算法 |
| [lwfw_list_search.md](lwfw_list_search.md) | list_search 线性扫描引擎：cdlist 遍历、限速状态机 |
| [lwfw_agent.md](lwfw_agent.md) | lwfw_agent 守护进程：事件消费、JSON 日志、文件轮转 |
| [lwfw_agent_log_system.md](lwfw_agent_log_system.md) | lwfw_agent 日志系统：O(n²) 事件合并、JSON 格式化、轮转 |
| [lwfw_hook_injection.md](lwfw_hook_injection.md) | 网络栈 Hook 注入点：ip4_input/egress_filter、初始化顺序 |
| [lwfw_data_structure.md](lwfw_data_structure.md) | 数据结构设计：策略、规则、包信息结构体、内存布局 |
| [lwfw_optimization.md](lwfw_optimization.md) | **优化建议汇总**：性能、安全性、稳定性、代码质量 |
| [lwfw_hotswap_analysis.md](lwfw_hotswap_analysis.md) | 热切换与策略原子性：双缓冲机制、copy_offset 技巧 |
| [lwfw_parser_concurrency.md](lwfw_parser_concurrency.md) | 解析器并发问题：静态状态变量、线程安全隐患 |
| [lwfw_ipc_mechanism.md](lwfw_ipc_mechanism.md) | IPC 机制分析：共享内存 FIFO、seL4 IPC |
| [lwfw_rule_matching.md](lwfw_rule_matching.md) | 规则匹配算法优化：位图索引、快速路径、SIMD 加速 |
| [lwfw_vlan_isolation_guide.md](lwfw_vlan_isolation_guide.md) | **VLAN 隔离配置指南**：方案设计、YAML 示例、部署步骤 |

---

## 快速参考

### 代码位置

| 组件 | 仓库 | 路径 |
|------|------|------|
| lwfw 核心 | util_libs | `libs/util_libs/liblwfw/src/lwfw.c` |
| 规则解析 | util_libs | `libs/util_libs/liblwfw/src/lwfw_parser.c` |
| 事件通知 | util_libs | `libs/util_libs/liblwfw/src/lwfw_notif.c` |
| 连接跟踪 | util_libs | `libs/util_libs/liblwfw/src/lwct/` |
| 树搜索 | util_libs | `libs/util_libs/liblwfw/src/tree_hs.c` |
| Hook 注入 | lwip_ds_mcu | `external/lwip_ds_mcu/src/core/ipv4/ip4.c` |
| CLI 配置 | os-framework | `os-framework/servers/net/src/lwfwcfg.c` |
| 守护进程 | os-framework | `os-framework/servers/daemons/lwfw_agent/` |

### 全局变量

```c
lwfw_firewall_t *lwfw_p;          // 防火墙全局句柄
lwfw_policy_t lwfw_policy;       // 当前活跃策略
lwfw_policy_t lwfw_policy_swap;   // 热切换备份
int lwct_enable;                   // 连接跟踪开关
struct stats_filter g_lwfw_stats;  // 防火墙统计
```

### Hook 调用路径

```
Ingress:  ip4_input() → lwfw_p->ops->ingress_filter()
Egress:   ip4_output_if() → lwfw_p->ops->egress_filter()
```

### 关键配置

| 参数 | 默认值 |
|------|--------|
| `LWFW_EVENT_NUM` | 512 |
| `LWFW_TREE_BUCKET_SIZE` | 8 |
| `LWFW_TREE_NODE_NUM` | 64 |
| `LWCT_BUCKET_COUNT` | 8192 |
| `LWCT_GC_SCAN_INTERVAL` | 3000ms |
