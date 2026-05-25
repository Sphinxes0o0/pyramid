# lwfw_agent 守护进程分析

> 代码路径: `os-framework/servers/daemons/lwfw_agent/`
> 入口: `src/main.c` → `src/event_handler.c`

---

## 1. 整体架构

```
lwfw_agent (守护进程)
  │
  ├─ lwfw_agent_init()                    ← 初始化
  │     ├─ wait_for_fs_mount()           ← 等 /var 分区挂载
  │     ├─ mkdir(LWFW_EVENT_DIR)          ← 创建日志目录
  │     ├─ rotate_event_files()           ← 清空旧日志
  │     ├─ setup_proc_event_thread()     ← 注册进程退出通知
  │     └─ timer_thread_init()           ← 启动定时器线程
  │
  ├─ lwfw_agent_svc_evt()                 ← IPC 消息处理
  │     ├─ SYS_LWFW_AGENT_INIT:          ← 映射共享内存
  │     ├─ SYS_LWFW_AGENT_NOTIFY:         ← 消费事件
  │     └─ SYS_LWFW_AGENT_IOCTL:          ← IOCTL 控制
  │
  ├─ timer_thread()                        ← 定时任务
  │     ├─ reload_config()                ← 重载配置文件软链接
  │     ├─ rotate_event_files()            ← 日志轮转
  │     └─ 清除日志计数
  │
  └─ proc_evt_thread()                     ← 进程退出监听
        └─ 关闭文件句柄，等待关机
```

---

## 2. 初始化流程

### 2.1 主入口

```c
int main(int argc, char **argv)
{
  struct sel4_svc_ctx ctx = {
    .name    = NSV_LWFW_AGENT_SVC_NAME,
    .evt_hdlr = lwfw_agent_svc_evt,
    .init    = lwfw_agent_init,
  };
  return sel4_svc_run(&ctx);
}
```

### 2.2 初始化

```c
int lwfw_agent_init(ctx, unused)
{
  sync_mutex_new(&lwfw_event_file_lock);     // 文件锁

  wait_for_fs_mount();                       // 等 /var 分区
  mkdir(LWFW_EVENT_DIR);                     // 创建 /var/log/lwfw

  rotate_event_files();                       // 每次启动清空旧日志

  setup_proc_event_thread();                  // 注册进程通知
  timer_thread_init();                        // 启动定时器线程
}
```

---

## 3. IPC 消息处理

### 3.1 三类消息

```c
sel4_msg_info_t lwfw_agent_svc_evt(badge, info, ctx)
{
  switch (label) {
    case SYS_LWFW_AGENT_INIT:
      return lwfw_agent_evt_consumer_init();  // 映射共享内存

    case SYS_LWFW_AGENT_NOTIFY:
      return lwfw_agent_evt_consume();        // 消费事件

    case SYS_LWFW_AGENT_IOCTL:
      return lwfw_agent_ioctl();              // 控制命令
  }
}
```

### 3.2 INIT - 共享内存映射

```c
static sel4_msg_info_t lwfw_agent_evt_consumer_init(info, pid, ctx)
{
  // 1. 从消息寄存器获取 dspace ID
  dsid = sel4_get_mr(0);

  // 2. 映射到本进程虚拟地址空间
  sys_dspace_map(dsid, &vaddr, &size, &attr);
  lwfw_event_fifo = (struct lwfw_event_fifo *)vaddr;

  // 3. 分配本地事件缓冲区
  lwfw_events = malloc(sizeof(lwfw_event) * queue_size);

  // 4. 初始化参数
  g_params = lwfw_event_fifo->params;

  __atomic_store_n(&is_lwfw_agent_ready, true, __ATOMIC_RELEASE);
}
```

### 3.3 NOTIFY - 事件消费

```c
static sel4_msg_info_t lwfw_agent_evt_consume(info, pid, ctx)
{
  if (!is_lwfw_agent_ready) return EAGAIN;
  if (threads_count >= LIMIT) return EAGAIN;

  // 1. 从共享内存 FIFO 读取事件
  get_idx = lwfw_event_fifo->get_idx;
  put_idx = lwfw_event_fifo->put_idx;
  event_num = (put_idx - get_idx) % queue_size;

  // 2. 复制到本地堆内存
  if (get_idx <= put_idx) {
    memcpy(lwfw_events, &events[get_idx+1], event_num * event_len);
  } else {
    // 环形缓冲区跨边界复制
    size1 = (queue_size - get_idx) * event_len;
    memcpy(lwfw_events, &events[get_idx+1], size1);
    memcpy(lwfw_events+size1, &events[0], event_num*event_len - size1);
  }
  dmb(ish);
  lwfw_event_fifo->get_idx = put_idx;  // 更新消费者索引

  // 3. 处理事件
  lwfw_agent_handle_evt(event_num, lwfw_events);

  __atomic_fetch_sub(&threads_count, 1, __ATOMIC_ACQ_REL);
  return 0;
}
```

---

## 4. 事件处理

### 4.1 事件合并

```c
static int lwfw_agent_handle_evt(event_num, events)
{
  // O(n²) 合并相同类型事件
  for (i = 0; i < event_num; i++) {
    if (events[i].merged) continue;
    events[i].count = 1;

    for (j = i+1; j < event_num; j++) {
      if (same event_type && same rule_id) {
        events[i].count++;
        events[j].merged = true;
      }
    }
  }
}
```

**问题**: O(n²) 复杂度，事件数量多时CPU开销大。

### 4.2 JSON 格式化

```c
json_format_eventlog(event, json_str)
  ├─ sprintf(json, "{")
  ├─ sprintf("event_id": %d, ...)
  ├─ sprintf("rule_id": %d, ...)
  ├─ sprintf("src_ip": "%d.%d.%d.%d", ...)
  ├─ sprintf("timestamp": %lu, ...)
  └─ sprintf("count": %d)  // 合并计数
```

**输出路径**: `/var/log/lwfw/lwfw-event_<timestamp>_<timestamp>.log`

---

## 5. 日志轮转

### 5.1 轮转条件

| 条件 | 阈值 |
|------|------|
| 文件大小 | `max_event_file_size` (默认 1MB) |
| 时间间隔 | `event_file_rotate_time` (默认 600s) |
| 文件数量 | 超过 `event_file_count` (默认 50) 时删最旧 |

### 5.2 轮转算法

```c
static int rotate_event_files()
{
  // 1. 如果当前文件打开，先关闭
  if (lwfw_curr_event_fd != -1) {
    close(lwfw_curr_event_fd);
    lwfw_curr_event_fd = -1;
  }

  // 2. scandir 扫描现有日志文件
  scandir(LWFW_EVENT_DIR, &namelist, lwfw-event*, alphasort);
  // 按文件名排序，最旧的在前

  // 3. 如果文件数超限，删除最旧的
  while (curr_file_count > event_file_count) {
    unlink(namelist[0]);
    删除最旧文件;
  }

  // 4. 重命名当前文件 (如果存在)
  new_name = lwfw-event_<timestamp>.log;
  rename(old_name, new_name);

  // 5. 创建新文件
  lwfw_curr_event_fd = open(new_name, O_CREAT|O_WRONLY);
  lwfw_event_file_size = 0;
}
```

---

## 6. 配置重载

### 6.1 软链接机制

```c
// 配置文件路径
LWFW_CONFIG_LINK_FILE = "/tmp/etc/lwfw/vdf_firewall_policy.yaml"
LWFW_DEF_CONFIG_PATH  = "/etc/lwfw/vdf_firewall_policy.yaml"
```

### 6.2 重载流程

```c
// timer_thread 中
if (!lwfw_reload_cfg_done && retry_cnt < MAX_RETRY) {
  reload_config(LWFW_CONFIG_LINK_FILE);
}

static int reload_config(filename)
{
  // 尝试打开软链接
  fd = open(filename, O_RDONLY);
  if (fd < 0) {
    retry_cnt++;
    return;
  }

  // 读取配置文件内容
  // 通知 lwfw 通过 IOCTL 更新 (TODO)
  close(fd);
}
```

**注意**: 当前 `reload_config` 只打开了文件，没有实际解析和通知 lwfw。

---

## 7. 进程退出处理

```c
proc_evt_thread()
{
  while (1) {
    seL4_Wait(evt_ntfn, &badge);
    event = evts->data[evts->head];

    if (event == EVENT_POWER_OFF) {
      close(lwfw_curr_event_fd);  // 关闭日志文件
      sys_pm(PM_SHUTDOWN_RESP, 1);  // 回应电源管理
    }
  }
}
```

---

## 8. 已知问题

### 8.1 事件合并 O(n²)

```c
for (i = 0; i < event_num; i++) {
  for (j = i+1; j < event_num; j++) {
    // ...
  }
}
```

每批事件最多512个，O(512²) = 262144 次比较，可接受但不够优雅。

### 8.2 scandir 每次轮转都遍历目录

`rotate_event_files()` 每次调用都执行 `scandir`，在高频率轮转时开销较大。

### 8.3 重试计数器无上限

```c
if (!lwfw_reload_cfg_done && g_agent_stats.retry_cfg_link < LWFW_MAX_RETRY)
```

`retry_cfg_link` 最大值 1000，之后即使软链接变为有效也不会再尝试。

### 8.4 没有 VDRS 上报

```c
// TODO: when VDRS host is ready, we shall send all events to cloud
```

代码注释说未来会上报到云端，但当前只写入本地文件。

### 8.5 timer_thread 退出无重启

如果 `sys_time_sleep` 失败，线程直接退出并返回，没有重连机制。

---

## 9. 统计信息

```c
struct stats_agent {
  uint32_t err_log_cnt;
  uint32_t warn_log_cnt;
  uint32_t throttled_logs;
  uint32_t invalid_ioctl;
  uint32_t no_event_file;
  uint32_t dir_create_fail;
  uint32_t file_rename_fail;
  uint32_t file_open_fail;
  uint32_t file_write_fail;
  uint32_t time_rotate_fail;
  uint32_t dir_rotate_fail;
  uint32_t filesize_rotate_fail;
  uint32_t total_rotate_fail;
  uint32_t total_rotate_success;
  uint32_t handle_events;
  uint32_t handle_events_err;
  uint32_t agent_not_ready;
  uint32_t ipc_notify;
  uint32_t ipc_ioctl;
  uint32_t ipc_err1;      // threads_count >= LIMIT
  uint32_t ipc_err2;        // ctx exited or fifo null
  uint32_t idx_err;        // 索引异常
  uint32_t invalid_msg;
  uint32_t invalid_msglen;
  uint32_t json_fail;
  uint32_t retry_cfg_link;
  uint32_t reload_cfg_done;
  uint32_t reload_cfg_too_long;
  uint32_t reload_call_svc_fail;
  uint32_t reload_cfg_parse_fail;
};
```
