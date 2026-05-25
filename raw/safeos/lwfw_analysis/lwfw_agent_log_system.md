# lwfw_agent 日志系统深度分析

> 代码路径: `os-framework/servers/daemons/lwfw_agent/src/event_handler.c`
> 头文件: `os-framework/servers/daemons/lwfw_agent/include/`

---

## 1. 日志系统架构

### 1.1 整体数据流

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

### 1.2 日志文件路径

```c
#define LWFW_EVENT_DIR "/var/log/lwfw"
#define LWFW_EVENT_FILE_PREFIX "lwfw-event"
```

---

## 2. 事件合并机制

### 2.1 O(n²) 合并算法

```c
// event_handler.c:561-576
static int lwfw_agent_handle_evt(uint32_t event_len, uint32_t event_num,
                                 struct lwfw_event *events)
{
    // O(n²) 合并
    for (i = 0; i < event_num; i++) {
        if (events[i].merged)
            continue;

        events[i].count = 1;  // 至少 1 个

        for (j = i + 1; j < event_num; j++) {
            if (events[i].event_type == events[j].event_type &&
                events[i].rule_id == events[j].rule_id) {
                events[i].count++;
                events[j].merged = true;
            }
        }

        // 写入 JSON 日志
        json_format_eventlog(&events[i], json_str);
    }
}
```

### 2.2 合并键

| 字段 | 类型 | 作用 |
|------|------|------|
| `event_type` | uint32_t | 事件类型 (LOG/RLIMIT/etc) |
| `rule_id` | uint16_t | 匹配的规则 ID |

**合并逻辑**: 具有相同 `event_type` 和 `rule_id` 的事件被合并，`count` 累加。

### 2.3 性能问题

| 事件数 | 比较次数 | 估计 CPU 时间 |
|--------|----------|--------------|
| 64 | 2,016 | ~2ms |
| 256 | 32,640 | ~33ms |
| 512 | 130,816 | ~130ms |

---

## 3. JSON 格式化

### 3.1 JSON 格式

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

### 3.2 格式化代码

```c
// event_handler.c:json_format_eventlog()
static int json_format_eventlog(struct lwfw_event *event, char *json_str)
{
    char src_ip_str[16];
    char dst_ip_str[16];

    // IP 地址转换
    format_ip(src_ip_str, event->data.src_ip);
    format_ip(dst_ip_str, event->data.dst_ip);

    // 格式化 JSON
    sprintf(json_str,
        "{"
        "\"event_id\": %u, "
        "\"rule_id\": %u, "
        "\"action\": \"%s\", "
        "\"proto\": \"%s\", "
        "\"src_ip\": \"%s\", "
        "\"dst_ip\": \"%s\", "
        "\"src_port\": %u, "
        "\"dst_port\": %u, "
        "\"if_name\": \"%s\", "
        "\"count\": %u, "
        "\"timestamp\": %lu"
        "}\n",
        event->hdr.event_id,
        event->data.rule_id,
        action_to_string(event->data.action),
        proto_to_string(event->data.proto),
        src_ip_str,
        dst_ip_str,
        event->data.src_port,
        event->data.dst_port,
        event->data.if_name,
        event->hdr.count,
        event->hdr.timestamp
    );
}
```

### 3.3 sprintf 安全问题

```c
// 潜在缓冲区溢出风险
sprintf(json_str, "...");  // 没有长度检查
```

**建议**: 使用 `snprintf` 并检查返回值。

---

## 4. 日志轮转

### 4.1 轮转条件

```c
// 触发轮转的条件
if (lwfw_event_file_size >= g_params.max_event_file_size) {
    rotate_event_files();  // 文件大小超限
}

if (now - lwfw_file_rotate_timeout >= g_params.event_file_rotate_time) {
    rotate_event_files();  // 时间间隔超限
}
```

### 4.2 轮转参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_event_file_size` | 1MB | 单文件最大大小 |
| `event_file_rotate_time` | 600s | 轮转时间间隔 |
| `event_file_count` | 50 | 保留文件数量 |

### 4.3 轮转算法

```c
static int rotate_event_files()
{
    // 1. 关闭当前文件
    if (lwfw_curr_event_fd != -1) {
        close(lwfw_curr_event_fd);
        lwfw_curr_event_fd = -1;
    }

    // 2. 扫描日志目录
    n = scandir(LWFW_EVENT_DIR, &namelist, filter, alphasort);
    // 按文件名排序，最旧的在前

    // 3. 删除超过数量的旧文件
    while (n > g_params.event_file_count) {
        unlink(namelist[0]->d_name);
        free(namelist[0]);
        n--;
    }

    // 4. 重命名当前文件
    new_name = gen_timestamp_filename();
    rename(LWFW_EVENT_FILE_CURRENT, new_name);

    // 5. 创建新文件
    lwfw_curr_event_fd = open(LWFW_EVENT_FILE_CURRENT,
                               O_CREAT | O_WRONLY | O_TRUNC,
                               0644);
    lwfw_event_file_size = 0;
}
```

### 4.4 文件名格式

```
lwfw-event_20240422_143052.log
```

格式: `lwfw-event_YYYYMMDD_HHMMSS.log`

---

## 5. 写文件流程

### 5.1 带锁写入

```c
sys_mutex_lock(&lwfw_event_file_lock);

// 写入 JSON
len = strlen(json_str);
if (write(lwfw_curr_event_fd, json_str, len) != len) {
    LWFW_STATICS_INC(g_agent_stats.file_write_fail);
}

// 更新大小计数
lwfw_event_file_size += len;

// 检查轮转
if (lwfw_event_file_size >= g_params.max_event_file_size) {
    rotate_event_files();
}

sys_mutex_unlock(&lwfw_event_file_lock);
```

### 5.2 写入失败处理

```c
if (write(...) != len) {
    LWFW_STATICS_INC(g_agent_stats.file_write_fail);
    // 继续处理，不阻塞
}
```

**问题**: 写入失败后事件丢失，不持久化。

---

## 6. 配置重载机制

### 6.1 软链接机制

```c
#define LWFW_CONFIG_LINK_FILE "/tmp/etc/lwfw/vdf_firewall_policy.yaml"
#define LWFW_DEF_CONFIG_PATH  "/etc/lwfw/vdf_firewall_policy.yaml"
```

软链接允许在不重启进程的情况下更新配置文件。

### 6.2 重载流程

```c
// timer_thread 中
static int reload_config(const char *filename)
{
    fd = open(filename, O_RDONLY);
    if (fd < 0) {
        retry_cnt++;
        return -1;
    }

    // TODO: 实际解析和通知 lwfw
    // 当前只打开文件，没有更新

    close(fd);
    return 0;
}
```

### 6.3 TODO 问题

```c
// 代码注释说明
// TODO: when VDRS host is ready, we shall send all events to cloud
```

**当前状态**: 事件只写入本地文件，不上报云端。

---

## 7. 进程退出处理

### 7.1 关机处理

```c
proc_evt_thread()
{
    while (1) {
        seL4_Wait(evt_ntfn, &badge);
        event = seL4_GetEvent();

        if (event == EVENT_POWER_OFF) {
            // 关闭文件
            if (lwfw_curr_event_fd != -1) {
                close(lwfw_curr_event_fd);
            }
            // 回应关机
            sys_pm(PM_SHUTDOWN_RESP, 1);
        }
    }
}
```

---

## 8. 稳定性问题

### 8.1 scandir 每次遍历

```c
// rotate_event_files() 每次轮转都执行
n = scandir(LWFW_EVENT_DIR, &namelist, filter, alphasort);
```

**问题**: 目录文件多时性能差

### 8.2 重试计数器上限

```c
if (!lwfw_reload_cfg_done && g_agent_stats.retry_cfg_link < LWFW_MAX_RETRY)
```

**问题**: `LWFW_MAX_RETRY` 达到后，即使软链接有效也不再尝试

### 8.3 timer_thread 退出

```c
// event_handler.c:251
if (sys_time_sleep(...) < 0) {
    break;  // 线程退出，无重启
}
```

---

## 9. 优化建议

### 9.1 哈希表合并 O(n²) → O(n)

```c
// 使用哈希表按 (event_type, rule_id) 分桶
GHashTable *merged = g_hash_table_new_full(
    g_int_hash, g_int_equal, NULL, NULL);

for (i = 0; i < event_num; i++) {
    key = (event->event_type << 16) | event->rule_id;
    existing = g_hash_table_lookup(merged, key);
    if (existing) {
        existing->count += event->count;
    } else {
        g_hash_table_insert(merged, key, event);
    }
}
```

### 9.2 异步写入

```c
// 使用单独的线程处理写入
// 主线程只管读取 FIFO 和合并
// 写入线程从队列取数据批量写入
```

### 9.3 写入失败告警

```c
if (write(...) != len) {
    LWFW_STATICS_INC(g_agent_stats.file_write_fail);
    // 发送告警到监控系统
}
```

---

## 10. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `event_handler.c` | 561 | O(n²) 事件合并 |
| `event_handler.c` | 630 | JSON 格式化 |
| `event_handler.c` | 700 | 日志轮转 |
| `event_handler.c` | 241 | timer_thread |
| `event_handler.c` | 251 | sleep 失败退出 |
