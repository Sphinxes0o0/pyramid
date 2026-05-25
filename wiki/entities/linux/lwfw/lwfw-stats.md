---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Statistics

## 定义

LWFW 维护三类统计计数器，通过 `g_lwfw_stats` 全局结构提供运行时可观测性，支持 `LWFW_STATS_SUPPORT` 条件编译开关。

## 统计数据结构

### stats_filter — 无状态防火墙统计

```c
struct stats_filter {
    uint32_t err_log_cnt;           // 错误日志计数
    uint32_t warn_log_cnt;          // 警告日志计数
    uint32_t throttled_logs;        // 被限流的日志数
    uint32_t drop_events;           // 丢弃的事件数
    uint32_t ct_notrack;           // 未追踪的包 (LWCT fallback)
    uint32_t total_rx_drop;        // RX 丢弃的包数
    uint32_t total_tx_drop;        // TX 丢弃的包数
    uint32_t total_event_cnt;      // 总事件数
    uint32_t cfg_loglevel;         // 日志级别配置次数
    uint32_t timer_notify_fail;    // 定时器通知失败
    uint32_t tree_build_fail;     // 树构建失败
    uint32_t tree_search_fail;     // 树搜索失败
};
```

### stats_ct — 连接追踪统计

```c
struct stats_ct {
    uint32_t gc_reclaim_fail;     // GC 回收失败
    uint32_t unexpected_pkt;      // 意外包
    uint32_t outoftrack_pkt;      // 追踪外的包
    uint32_t conn_full;           // 连接表满
    uint32_t tcp_rst;             // TCP RST
};
```

### stats_agent — 防火墙代理统计

```c
struct stats_agent {
    uint32_t err_log_cnt;
    uint32_t invalid_ioctl;       // 无效 IOCTL
    uint32_t file_write_fail;    // 文件写入失败
    uint32_t handle_events;      // 处理的事件数
    uint32_t reload_cfg_done;    // 配置重载完成
};
```

## 统计宏

```c
#ifdef LWFW_STATS_SUPPORT
#define LWFW_STATICS_INC(x)  (++x)
#define LWCT_STATICS_INC(x)  (++x)
#define LWFW_AGENT_STATICS_INC(x) (++x)
#else
#define LWFW_STATICS_INC(x)          // 空操作
#define LWCT_STATICS_INC(x)
#define LWFW_AGENT_STATISTICS_INC(x)
#endif
```

## 统计导出

`lwfw_dump_fw_info()` 导出所有统计：

```c
void lwfw_dump_fw_info(bool verbose) {
  LWFW_PRINTF("    total_rx_drop: %u", g_lwfw_stats.total_rx_drop);
  LWFW_PRINTF("    total_tx_drop: %u", g_lwfw_stats.total_tx_drop);
  LWFW_PRINTF("    throttled_logs: %u", g_lwfw_stats.throttled_logs);
  LWFW_PRINTF("    ct_notrack: %u", g_lwfw_stats.ct_notrack);
}
```

## 与 Linux 对比

| 特性 | LWFW | Linux iptables |
|------|------|----------------|
| 计数器位置 | 全局结构体 | /proc/net/stat |
| 包计数 | 通过 stats_filter | 通过 iptables -v |
| 速率限制 | 每规则 | 通过 limit 模块 |
| 事件统计 | FIFO + 通知线程 | 通过 ulogd |

## 相关概念

- [[entities/linux/lwfw/lwfw-agent]] — Agent 守护进程统计
- [[entities/linux/lwfw/lwfw-notif]] — 事件 FIFO 机制
- [[entities/linux/lwfw/lwfw-core-filtering]] — 丢弃统计更新点
- [[entities/linux/lwfw/lwfw-lwct-gc-analysis]] — LWCT GC 统计
