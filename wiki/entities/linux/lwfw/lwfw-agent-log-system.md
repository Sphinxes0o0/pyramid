---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Agent Log System

## 定义

`lwfw_agent` 日志系统负责将安全事件 **JSON 格式化**后写入 `/var/log/lwfw/` 目录，支持按大小和时间轮转。

## 整体数据流

```
lwfw (内核进程)
    │
    │ lwfw_event_push(event)
    │   └─ 写入共享内存 FIFO
    │
    ▼
lwfw_event_fifo (共享内存)
    │
    │ SYS_LWFW_AGENT_NOTIFY (seL4 IPC)
    │
    ▼
lwfw_agent (用户态守护进程)
    │
    │ lwfw_agent_evt_consume()
    │   └─ 从 FIFO 读取事件
    │
    ▼
lwfw_agent_handle_evt()
    ├─ 事件合并 (O(n²))
    └─ json_format_eventlog()
          │
          ▼
    /var/log/lwfw/lwfw-event_<timestamp>.log
```

## 事件合并 O(n²)

```c
// event_handler.c:561-576
for (i = 0; i < event_num; i++) {
    if (events[i].merged) continue;
    events[i].count = 1;

    for (j = i + 1; j < event_num; j++) {
        if (events[i].event_type == events[j].event_type &&
            events[i].rule_id == events[j].rule_id) {
            events[i].count++;
            events[j].merged = true;
        }
    }
    json_format_eventlog(&events[i], json_str);
}
```

**性能问题**:

| 事件数 | 比较次数 | 估计 CPU 时间 |
|--------|----------|--------------|
| 64 | 2,016 | ~2ms |
| 512 | 130,816 | ~130ms |

## JSON 格式

```json
{
  "event_id": 1,
  "rule_id": 42,
  "action": "DENY",
  "proto": "TCP",
  "src_ip": "192.168.1.100",
  "dst_ip": "10.0.0.1",
  "src_port": 54321,
  "dst_port": 443,
  "if_name": "eth0",
  "count": 5,
  "timestamp": 1712345678901
}
```

## 日志轮转

| 条件 | 阈值 |
|------|------|
| 文件大小 | `max_event_file_size` (默认 1MB) |
| 时间间隔 | `event_file_rotate_time` (默认 600s) |
| 文件数量 | 超过 `event_file_count` (默认 50) 时删最旧 |

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| O(n²) 合并算法 | P1 | 事件多时 CPU 占用高 |
| sprintf 溢出风险 | P2 | 无长度检查 |
| scandir 每次遍历 | P2 | 目录文件多时性能差 |
| 写入失败事件丢失 | P2 | 不持久化 |

## 优化建议

```c
// 优化: O(n) 哈希表合并
GHashTable *merged = g_hash_table_new(g_direct_hash, same_event_key);
for (i = 0; i < event_num; i++) {
    key = events[i].event_type << 16 | events[i].rule_id;
    existing = g_hash_table_lookup(merged, key);
    if (existing) {
        existing->count += events[i].count;
    } else {
        g_hash_table_insert(merged, key, &events[i]);
    }
}
```

## 相关概念

- [[entities/linux/lwfw/lwfw-agent]] — Agent 守护进程
- [[entities/linux/lwfw/lwfw-ipc-mechanism]] — 共享内存 FIFO
- [[entities/linux/lwfw/lwfw-notif]] — 事件生成与通知
