# LWFW 统计计数分析 — T-073

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: 统计计数、byte/packet counter、事件统计

---

## 1. 概述

LWFW 维护三类统计计数器：

1. **stats_filter**: 无状态防火墙统计
2. **stats_ct**: 连接追踪统计
3. **stats_agent**: 防火墙代理统计

---

## 2. 统计数据结构

### 2.1 stats_filter — 无状态防火墙统计

**文件**: `include/lwfw_stats.h:8-28`

```c
struct stats_filter {
    // 日志统计
    uint32_t err_log_cnt;           // 错误日志计数
    uint32_t warn_log_cnt;          // 警告日志计数
    uint32_t throttled_logs;        // 被限流的日志数
    uint32_t throttled_events;      // 被限流的事件数
    uint32_t drop_events;           // 丢弃的事件数

    // 连接追踪
    uint32_t ct_notrack;           // 未追踪的包 (LWCT fallback)

    // 包丢弃统计
    uint32_t total_rx_drop;        // RX 丢弃的包数
    uint32_t total_tx_drop;        // TX 丢弃的包数

    // 事件统计
    uint32_t total_event_cnt;      // 总事件数

    // 配置统计
    uint32_t cfg_loglevel;         // 日志级别配置次数
    uint32_t cfg_reload;           // 配置重载次数
    uint32_t cfg_rule;             // 规则配置次数
    uint32_t cfg_parameter;        // 参数配置次数
    uint32_t invalid_rule_idx;     // 无效规则索引数

    // 错误统计
    uint32_t timer_notify_fail;     // 定时器通知失败
    uint32_t event_notify_fail;    // 事件通知失败
    uint32_t fifo_not_ready;      // FIFO 未就绪

    // 树搜索统计
    uint32_t tree_build_fail;     // 树构建失败
    uint32_t tree_search_fail;     // 树搜索失败
};
```

### 2.2 stats_ct — 连接追踪统计

**文件**: `include/lwfw_stats.h:31-49`

```c
struct stats_ct {
    // 配置统计
    uint32_t cfg_state;            // 状态配置次数
    uint32_t cfg_parameter;        // 参数配置次数

    // GC 统计
    uint32_t gc_reclaim_fail;     // GC 回收失败
    uint32_t gc_scan_yield;      // GC 扫描让出

    // 包处理统计
    uint32_t unexpected_pkt;      // 意外包
    uint32_t notrack_pkt;        // 未追踪的包
    uint32_t outoftrack_pkt;      // 追踪外的包
    uint32_t pbuf_conn_err;      // pbuf 连接错误
    uint32_t invert_fail;         // 反向查找失败

    // 连接状态统计
    uint32_t conn_full;           // 连接表满
    uint32_t conn_is_null;        // 连接为空
    uint32_t conn_is_dying;       // 连接正在关闭
    uint32_t conn_will_drop;      // 将被丢弃的连接
    uint32_t icmp_err;            // ICMP 错误
    uint32_t icmp_not_support;    // 不支持的 ICMP
    uint32_t tcp_rst;             // TCP RST
    uint32_t pkt_handle_err;      // 包处理错误
};
```

### 2.3 stats_agent — 防火墙代理统计

**文件**: `include/lwfw_stats.h:52-87`

```c
struct stats_agent {
    uint8_t major_ver;
    uint8_t minor_ver;
    uint8_t revise_ver;
    uint8_t reserved;

    // 日志统计
    uint32_t err_log_cnt;
    uint32_t warn_log_cnt;
    uint32_t throttled_logs;

    // IOCTL 统计
    uint32_t invalid_ioctl;       // 无效 IOCTL

    // 文件统计
    uint32_t no_event_file;      // 无事件文件
    uint32_t dir_create_fail;    // 目录创建失败
    uint32_t file_rename_fail;   // 文件重命名失败
    uint32_t file_open_fail;     // 文件打开失败
    uint32_t file_write_fail;    // 文件写入失败

    // 轮换统计
    uint32_t time_rotate_fail;   // 时间轮换失败
    uint32_t dir_rotate_fail;    // 目录轮换失败
    uint32_t filesize_rotate_fail; // 文件大小轮换失败
    uint32_t total_rotate_fail;   // 总轮换失败
    uint32_t total_rotate_success; // 总轮换成功

    // 事件处理统计
    uint32_t handle_events;      // 处理的事件数
    uint32_t handle_events_err;  // 事件处理错误

    // IPC 统计
    uint32_t agent_not_ready;    // 代理未就绪
    uint32_t ipc_notify;         // IPC 通知
    uint32_t ipc_ioctl;          // IPC IOCTL
    uint32_t ipc_err1;           // IPC 错误 1
    uint32_t ipc_err2;           // IPC 错误 2

    // 消息统计
    uint32_t idx_err;            // 索引错误
    uint32_t invalid_msg;        // 无效消息
    uint32_t invalid_msglen;     // 无效消息长度
    uint32_t json_fail;          // JSON 解析失败

    // 配置重载统计
    uint32_t retry_cfg_link;     // 重试配置链接
    uint32_t reload_cfg_done;    // 配置重载完成
    uint32_t reload_cfg_too_long; // 配置重载超时
    uint32_t reload_call_svc_fail; // 调用服务失败
    uint32_t reload_cfg_parse_fail; // 配置解析失败
};
```

---

## 3. 统计宏

### 3.1 LWFW_STATICS_INC

**文件**: `include/lwfw_stats.h:89-98`

```c
#ifdef LWFW_STATS_SUPPORT
#define LWFW_STATICS_INC(x)  (++x)    // 递增计数器
#define LWCT_STATICS_INC(x) (++x)
#define LWFW_AGENT_STATICS_INC(x) (++x)
#else
#define LWFW_STATICS_INC(x)          // 空操作 (统计关闭)
#define LWCT_STATICS_INC(x)
#define LWFW_AGENT_STATISTICS_INC(x)
#endif
```

---

## 4. 统计更新点

### 4.1 包丢弃统计

**文件**: `lwfw.c:828-834`

```c
if ((ret & LWFW_ACTION_CODE_DENY) == LWFW_ACTION_CODE_DENY) {
    LWFW_STATICS_INC(g_lwfw_stats.total_rx_drop);  // RX 丢弃
    return ERR_VAL;
}
```

**文件**: `lwfw.c:874-880`

```c
if ((ret & LWFW_ACTION_CODE_DENY) == LWFW_ACTION_CODE_DENY) {
    LWFW_STATICS_INC(g_lwfw_stats.total_tx_drop);  // TX 丢弃
    return ERR_VAL;
}
```

### 4.2 事件统计

**文件**: `lwfw.c:665`

```c
LWFW_STATICS_INC(g_lwfw_stats.total_event_cnt);
```

### 4.3 日志限流统计

**文件**: `lwfw.c:84-96`

```c
bool log_need_throttle(uint32_t log_level)
{
  if (log_level == LWFW_LOG_ERR) {
    LWFW_STATICS_INC(g_lwfw_stats.err_log_cnt);
  }
  if (log_level == LWFW_LOG_WARN) {
    LWFW_STATICS_INC(g_lwfw_stats.warn_log_cnt);
  }
  if (lwfw_p->policy->params.logs_per_second != 0 &&
      __atomic_fetch_add(&g_lwfw_curr_log_cnt, 1, __ATOMIC_RELAXED)
      >= lwfw_p->policy->params.logs_per_second) {
    LWFW_STATICS_INC(g_lwfw_stats.throttled_logs);
    return true;
  }
}
```

### 4.4 LWCT fallback 统计

**文件**: `lwfw.c:741`

```c
if (lwct_enable == 1 && !p->_lwct) {
    LWFW_STATICS_INC(g_lwfw_stats.ct_notrack);
    // ...
}
```

---

## 5. 统计导出

### 5.1 lwfw_dump_fw_info — 导出所有统计

**文件**: `lwfw.c:1321-1393`

```c
void lwfw_dump_fw_info(bool verbose)
{
  LWFW_PRINTF("========= Dump firewall statistics: =========");
  LWFW_PRINTF("    err_log_cnt: %u", g_lwfw_stats.err_log_cnt);
  LWFW_PRINTF("    warn_log_cnt: %u", g_lwfw_stats.warn_log_cnt);
  LWFW_PRINTF("    log_pps: %u", g_lwfw_curr_log_cnt);
  LWFW_PRINTF("    throttled_logs: %u", g_lwfw_stats.throttled_logs);
  LWFW_PRINTF("    event_pps: %u", lwfw_p->ctrl.event_pps);
  LWFW_PRINTF("    throttled_events: %u", g_lwfw_stats.throttled_events);
  LWFW_PRINTF("    drop_events: %u", g_lwfw_stats.drop_events);
  LWFW_PRINTF("    ct_notrack: %u", g_lwfw_stats.ct_notrack);
  LWFW_PRINTF("    total_rx_drop: %u", g_lwfw_stats.total_rx_drop);
  LWFW_PRINTF("    total_tx_drop: %u", g_lwfw_stats.total_tx_drop);
  LWFW_PRINTF("    total_event_cnt: %u", g_lwfw_stats.total_event_cnt);
  LWFW_PRINTF("    cfg_loglevel: %u", g_lwfw_stats.cfg_loglevel);
  LWFW_PRINTF("    cfg_reload: %u", g_lwfw_stats.cfg_reload);
  LWFW_PRINTF("    cfg_rule: %u", g_lwfw_stats.cfg_rule);
  LWFW_PRINTF("    cfg_parameter: %u", g_lwfw_stats.cfg_parameter);
  LWFW_PRINTF("    invalid_rule_idx: %u", g_lwfw_stats.invalid_rule_idx);
  LWFW_PRINTF("    timer_notify_fail: %u", g_lwfw_stats.timer_notify_fail);
  LWFW_PRINTF("    event_notify_fail: %u", g_lwfw_stats.event_notify_fail);
  LWFW_PRINTF("    fifo_not_ready: %u", g_lwfw_stats.fifo_not_ready);
  LWFW_PRINTF("    tree_build_fail: %u", g_lwfw_stats.tree_build_fail);
  LWFW_PRINTF("    tree_search_fail: %u", g_lwfw_stats.tree_search_fail);
}
```

---

## 6. 速率限制统计

### 6.1 规则速率限制

规则级别的速率限制在 `lwfw_rule_rlimit_worker()` 中更新：

```c
void lwfw_rule_rlimit_worker(uint32_t interval)
{
  cdlist_iter_entry_safe(curr_rule, n_rule, header, next) {
    if (curr_rule->flags & LWFW_RULE_FLAGS_RATE_LIMIT) {
      // 清除 PPS 计数
      __atomic_store_n(&curr_rule->rlimit.rx_pps, 0, __ATOMIC_RELAXED);

      // 检查过期
      if (curr_rule->rlimit.expire != 0 &&
          curr_rule->rlimit.state == LWFW_RLIMIT_STATE_LIMIT) {
        // 恢复正常状态
        __atomic_store_n(&curr_rule->rlimit.state,
                         LWFW_RLIMIT_STATE_NORMAL, __ATOMIC_RELAXED);
      }
    }
  }
}
```

### 6.2 规则速率限制计数器

```c
// rate_limit_t 结构
typedef struct {
  char name[32];
  uint32_t burst;          // 突发量
  uint32_t rate;          // 速率
  uint32_t expire;        // 过期时间
  uint8_t event_mode;     // 事件模式
  uint8_t state;          // 状态

  // 运行时计数器
  uint32_t rx_pps;        // 当前 PPS
  uint32_t time;          // 持续时间
  uint32_t drops;         // 丢弃计数
  uint32_t occurs;       // 触发次数
} rate_limit_t;
```

---

## 7. 事件 FIFO 统计

### 7.1 事件队列

```c
static struct lwfw_event_fifo *lwfw_event_fifo = NULL;

struct lwfw_event_fifo {
  uint32_t size;
  uint32_t count;
  uint32_t head;
  uint32_t tail;
  lwfw_event_t events[LWFW_EVENT_NUM];
};
```

### 7.2 事件统计

```c
// 事件入队
if (lwfw_event_fifo_is_full()) {
  LWFW_STATICS_INC(g_lwfw_stats.drop_events);
  return LWFW_ERR_FIFO_FULL;
}
```

---

## 8. 总结

### 8.1 统计分类

| 类型 | 用途 | 更新频率 |
|------|------|----------|
| **total_rx_drop** | RX 丢弃包数 | 每个丢弃的包 |
| **total_tx_drop** | TX 丢弃包数 | 每个丢弃的包 |
| **total_event_cnt** | 安全事件总数 | 每个事件 |
| **throttled_logs** | 被限流的日志 | 每秒 |
| **throttled_events** | 被限流的事件 | 每秒 |

### 8.2 关键设计

1. **全局统计**: `g_lwfw_stats` 是全局变量，所有 filter 调用都更新同一个结构
2. **原子操作**: 使用 `__atomic_fetch_add` 进行原子递增
3. **条件编译**: `LWFW_STATS_SUPPORT` 控制是否启用统计
4. **导出接口**: `lwfw_dump_fw_info()` 用于导出所有统计

### 8.3 与 Linux 对比

| 特性 | LWFW | Linux iptables |
|------|------|----------------|
| **计数器位置** | 全局结构体 | /proc/net/stat |
| **包计数** | 通过 stats_filter | 通过 iptables -v |
| **速率限制** | 每规则 | 通过 limit 模块 |
| **事件统计** | FIFO + 通知线程 | 通过 ulogd |
