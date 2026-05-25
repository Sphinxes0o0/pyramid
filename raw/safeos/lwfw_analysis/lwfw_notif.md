# lwfw 事件通知机制分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw_notif.c`

---

## 1. 整体架构

```
lwfw (lwIP 进程)                              lwfw_agent (守护进程)
      │                                              │
      │ sys_dspace_create(SHM)                       │
      │────────── 共享内存 lwfw_event_fifo ──────────►│
      │                                              │
      │ sys_svc_wait(NSV_LWFW_AGENT_SVC_NAME)       │
      │────────── 获取 agent endpoint ───────────────►│
      │                                              │
      │ lwfw_notification_thread ──────────────────┐ │
      │   ├─ lwfw_notification_timer_thread         │ │
      │   │     └─ sel4_call(SYS_LWFW_AGENT_NOTIFY)  │ │
      │   └─ lwfw_notification_thread_loop           │ │
      │         └─ sync_sem_wait() 阻塞等事件         │ │
      │                                              │ │
      │ lwfw_event_push(event)                      │ │
      │   └─ put_idx++ (无锁)                        │ │
      │──────────────────────────────────────────────┘ │
      │                                              │
      │ ←─────── JSON 日志文件 (events_*.log) ─────────┘
```

---

## 2. 共享内存 FIFO

### 2.1 数据结构

```c
struct lwfw_event_fifo {
#if LWFW_FIFO_MULTI_PRODUCER_ON
  sync_mutex_t prod_lock;           // 多生产者时加锁
#endif
  lwfw_agent_parameters_t params;   // 配置参数
  lwfw_agent_data_t data;           // 统计信息
  uint32_t queue_size;              // 事件槽数量 (默认512)
  volatile uint32_t get_idx;        // 消费者索引
  volatile uint32_t put_idx;        // 生产者索引
  uint32_t user;
  uint32_t event_len;
  lwfw_event events[1];            // 变长数组
};
```

### 2.2 环形缓冲区算法

```c
// 判满
((put_idx + 1) % queue_size) == get_idx

// 判空
put_idx == get_idx

// 推进
next_put_idx = (put_idx + 1) % queue_size;
memcpy(&events[next_put_idx], event, sizeof(event));
__dmb(ish);                         // 内存屏障，确保数据先于索引更新
put_idx = next_put_idx;
```

### 2.3 内存屏障

```c
dmb(ish);  // ARM inner shareable barrier
// 确保:
// 1. 事件数据写入完成后再更新 put_idx
// 2. 防止指令重排导致的消费者看到部分写入的数据
```

---

## 3. 事件通知线程

### 3.1 两个线程

| 线程 | 职责 |
|------|------|
| `lwfw_notification_thread` | 主线程，初始化 + 循环处理事件 |
| `lwfw_notification_timer_thread` | 定时器线程，定期检查 FIFO + 触发热重载 |

### 3.2 主线程循环

```c
static void lwfw_notification_thread_loop()
{
  while (1) {
    sync_sem_wait(&lwfw_wait_for_event);  // 阻塞等信号
    sys_mutex_lock(&lwfw_event_offload_lock);

    // FIFO 快要满了，触发通知
    if (lwfw_event_fifo_is_almost_full()) {
      lwfw_event_write_timestamp();
      should_notify = true;
    }
    sys_mutex_unlock(&lwfw_event_offload_lock);

    if (should_notify) {
      sel4_call(SYS_LWFW_AGENT_NOTIFY);    // 通知 agent
    }
  }
}
```

### 3.3 定时器线程

```c
static void lwfw_notification_timer_thread()
{
  while (1) {
    sys_time_sleep(1s);

    // === 热重载检查 ===
    if (cfg_in_reloading) {
      lwfw_p->policy->rule_tables[IN_TABLE].state = DISABLE;
      lwfw_config_reload_manifest(cfg_path);
      lwfw_p->policy->rule_tables[IN_TABLE].state = ENABLE;
      cfg_in_reloading = false;
    }

    // === 定期事件通知 ===
    if (timer >= event_notify_interval) {
      if (FIFO not empty) {
        lwfw_event_write_timestamp();
        sel4_call(SYS_LWFW_AGENT_NOTIFY);  // 通知 agent
      }
    }

    // === 清除事件 PPS 计数 ===
    event_pps = 0;

    // === 速率限制工作 ===
    if (timer >= pkt_rlimit_interval)
      lwfw_rule_rlimit_worker();

    // === 清除日志计数 ===
    g_lwfw_curr_log_cnt = 0;
  }
}
```

---

## 4. 事件推送流程

```c
err_t lwfw_event_push(struct lwfw_event *event)
{
#if LWFW_FIFO_MULTI_PRODUCER_ON
  sync_mutex_lock(&prod_lock);  // 多生产者时加锁
#endif

  // 环缓冲区写
  next_put = (put_idx + 1) % queue_size;
  memcpy(&events[next_put], event, sizeof(*event));
  dmb(ish);
  put_idx = next_put;

#if LWFW_FIFO_MULTI_PRODUCER_ON
  sync_mutex_unlock(&prod_lock);
#endif

  // 几乎满时，信号通知线程
  if (lwfw_event_fifo_is_almost_full())
    sync_sem_post(&lwfw_wait_for_event);

  return ERR_OK;
}
```

---

## 5. 与 Agent 的 IPC

### 5.1 初始化

```c
static err_t lwfw_event_fifo_init()
{
  // 1. 等待 agent 服务就绪
  sys_svc_wait(NSV_LWFW_AGENT_SVC_NAME, ..., &lwfw_agent_ep);

  // 2. 创建共享内存 dspace
  fifo_size = ALIGN_UP(sizeof(lwfw_event_fifo) + event_num * sizeof(event), 4KB);
  sys_dspace_create(fifo_size, SHM|WRITE|CACHE, &addr, &lwfw_dspace_id);

  // 3. 将 dspace 授予 agent
  sys_dspace_grant(lwfw_dspace_id, lwfw_agent_ep, ...);

  // 4. 初始化 FIFO
  lwfw_event_fifo = (void*)addr;
  lwfw_event_fifo->put_idx = 0;
  lwfw_event_fifo->get_idx = 0;

  // 5. 通知 agent 映射该内存
  sel4_call_mrs(SYS_LWFW_AGENT_INIT, &lwfw_dspace_id);
}
```

### 5.2 Agent IOCTL

```c
void lwfw_agent_ioctl(opcode, p1, p2)
{
  sel4_msg_info_t info;
  info = sel4_msg_info_set_label(info, SYS_LWFW_AGENT_IOCTL);
  sel4_set_mr(0, opcode);
  sel4_set_mr(1, p1);
  sel4_set_mr(2, p2);
  sel4_call(lwfw_agent_ep, info);
}
```

支持的 opcode:
- `LWFW_AGENT_IOCTL_SET_LOG_LEVEL`
- `LWFW_AGENT_IOCTL_SET_LOG_THROTTLING`
- `LWFW_AGENT_IOCTL_GET_STATS`

---

## 6. 事件格式

### 6.1 事件头

```c
struct lwfw_event_hdr {
  uint32_t version;
  uint32_t flag;           // LWFW_EVENT_FLAGS_MERGED 等
  uint32_t event_type;     // LOG_KIND_EVENT
  uint32_t count;          // 事件计数
  uint32_t reserved;
  unsigned long timestamp; // 微秒级时间戳
};
```

### 6.2 事件数据

```c
struct lwfw_event_data {
  uint32_t event_id;       // LWFW_SEV_GENERIC_FILTER / RATELIMIT / ...
  uint16_t rule_id;
  uint8_t action;
  uint8_t proto;
  char if_name[32];
  uint8_t src_mac[6];
  uint8_t dst_mac[6];
  uint16_t ether_type;
  uint16_t vlan;
  uint16_t pdu_len;
  uint32_t src_ip;
  uint32_t dst_ip;
  uint16_t src_port;
  uint16_t dst_port;
};
```

---

## 7. 已知问题

### 7.1 定时器线程依赖

`lwfw_notification_timer_thread` 中既有定时任务又有热重载逻辑，职责不单一。如果 `sys_time_sleep` 失败，线程直接退出，没有任何重连或重启机制。

### 7.2 几乎满阈值

```c
// lwfw_notification_timer_thread:97
if (MAX_TIMER_VAL_PER_SECOND * loops++ >= event_notify_interval)
```

这里的 `event_notify_interval` 单位是毫秒，但循环计数器 `loops` 每秒只增加1。这意味着 `event_notify_interval` 实际最小粒度是1秒，无法支持毫秒级通知。

### 7.3 同步等待信号量

```c
sync_sem_wait(&lwfw_wait_for_event);
```

在 `lwfw_notification_thread_loop` 中使用信号量等待，但信号量没有超时。如果 FIFO 满了但 agent 一直不通知，线程会永久阻塞。

### 7.4 热重载时 Ingress 被短暂禁用

```c
if (lwfw_p->policy->rule_tables[IN_TABLE].state == LWFW_STATE_ENABLE) {
  lwfw_p->policy->rule_tables[IN_TABLE].state = LWFW_STATE_DISABLE;
  lwfw_config_reload_manifest(fw_ctrl_p->cfg_path);  // 这期间 Ingress 无防火墙
  lwfw_p->policy->rule_tables[IN_TABLE].state = LWFW_STATE_ENABLE;
}
```

热重载期间 Ingress 防火墙被关闭，如果重载耗时较长（如文件IO阻塞），这段时间内数据包无法被过滤。建议使用双策略切换而非单策略修改。
