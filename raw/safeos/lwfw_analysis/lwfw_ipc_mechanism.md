# 事件 FIFO 与 IPC 机制深度分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw_notif.c`
> Agent 代码: `os-framework/servers/daemons/lwfw_agent/src/event_handler.c`

---

## 1. 共享内存 FIFO 架构

### 1.1 数据结构布局

```c
struct lwfw_event_fifo {
    sync_mutex_t prod_lock;           // 多生产者锁 (可选)
    lwfw_agent_parameters_t params;    // 配置参数
    lwfw_agent_data_t data;          // 统计信息
    volatile uint32_t get_idx;        // 消费者索引 (agent 写，lwfw 读)
    volatile uint32_t put_idx;        // 生产者索引 (lwfw 写，agent 读)
    uint32_t queue_size;             // 事件槽数量
    uint32_t user;                   // 用户标识
    uint32_t event_len;              // 每个事件长度
    lwfw_event events[1];          // 变长数组
};
```

**内存布局**:

```
+------------------+ <- 基址
| prod_lock        |  同步锁
+------------------+
| params           |  配置参数
+------------------+      实际数据
| data             |  统计信息
+------------------+
| get_idx          |  消费者索引
+------------------+      (双字对齐)
| put_idx          |  生产者索引
+------------------+
| queue_size       |  队列大小
+------------------+
| user             |  用户标识
+------------------+
| event_len        |  事件长度
+------------------+      4KB 对齐边界
| events[0]        |  事件槽 0
+------------------+
| events[1]        |  事件槽 1
+------------------+
| ...              |
+------------------+ <- 基址 + fifo_size
```

### 1.2 内存分配公式

```c
fifo_size = ALIGN_UP(
    sizeof(lwfw_event_fifo) + (queue_size - 1) * sizeof(lwfw_event),
    4096  // 4KB 对齐
);
```

**典型值**:
- `queue_size` = 512 (默认 `LWFW_EVENT_NUM`)
- `sizeof(lwfw_event)` ≈ 64 bytes
- 总大小 ≈ 4KB + 511 × 64 ≈ 36KB

### 1.3 环形缓冲区原理

```
        put_idx                      put_idx
          │                            │
          ▼                            ▼
    ┌─────────────────────────────────────┐
    │ [0] [1] [2] [3] ... [510] [511]    │  events[queue_size]
    └─────────────────────────────────────┘
          ▲                            ▲
          │                            │
      get_idx                      get_idx
          │                            │
    (数据未消费)               (数据已消费,等待覆写)
```

**关键不变式**:
- 缓冲区永远不会真正满：留一个空槽
- 索引使用模运算：`idx = idx % queue_size`（但用 % 会慢，实际用 &mask 优化）

---

## 2. 生产者-消费者同步

### 2.1 无锁设计

```c
// lwfw (生产者) - 在数据包处理上下文调用
err_t lwfw_event_push(struct lwfw_event *event)
{
    uint32_t next_put = (put_idx + 1) % queue_size;

    if (next_put == get_idx) {
        return ERR_MEM;  // FIFO 满，丢弃
    }

    memcpy(&events[next_put], event, sizeof(*event));
    dmb(ish);              // 数据内存屏障
    put_idx = next_put;   // 发布数据

    return ERR_OK;
}
```

```c
// agent (消费者) - 在独立线程运行
static sel4_msg_info_t lwfw_agent_evt_consume(info, pid, ctx)
{
    // 1. 读取索引
    get_idx = lwfw_event_fifo->get_idx;
    put_idx = lwfw_event_fifo->put_idx;

    // 2. 计算可用事件数
    if (put_idx >= get_idx) {
        event_num = put_idx - get_idx;
    } else {
        event_num = queue_size - get_idx + put_idx;
    }

    // 3. 复制事件到本地缓冲区
    // 4. 更新 get_idx
    dmb(ish);
    lwfw_event_fifo->get_idx = put_idx;  // 批量确认

    return 0;
}
```

### 2.2 内存屏障 (Memory Barrier)

```c
dmb(ish);  // ARM inner shareable barrier
```

**作用**:
1. 确保 `memcpy` 完成后再更新 `put_idx`
2. 防止 CPU 指令重排导致消费者看到部分写入的数据
3. 在多核系统中确保跨核可见性

### 2.3 批量确认优化

```c
// agent 不是每处理一个事件就更新 get_idx
// 而是批量读取后统一更新
lwfw_event_fifo->get_idx = put_idx;
```

**优点**: 减少原子操作次数
**风险**: 如果 agent 崩溃重启，`get_idx` 丢失，可能重复消费事件

---

## 3. seL4 IPC 通信

### 3.1 服务发现

```c
// lwfw 等待 agent 服务就绪
sys_svc_wait(NSV_LWFW_AGENT_SVC_NAME, SVC_WAIT_EXACT, &lwfw_agent_ep);
```

**机制**: seL4 CSpace 根目录查找，获取 agent 的 endpoint capability

### 3.2 消息格式

```c
// 通知消息
sel4_msg_info_t info = sel4_msg_info_set_label(info, SYS_LWFW_AGENT_NOTIFY);
info = sel4_msg_info_set_length(info, 0);  // 无额外数据
sel4_call(lwfw_agent_ep, info);  // 同步调用
```

### 3.3 初始化握手

```c
// lwfw: 创建共享内存
fifo_size = ALIGN_UP(sizeof(lwfw_event_fifo) + event_num * sizeof(event), 4KB);
sys_dspace_create(fifo_size, SHM|WRITE|CACHE, &addr, &lwfw_dspace_id);

// lwfw: 授予 agent 访问权限
sys_dspace_grant(lwfw_dspace_id, lwfw_agent_ep, ...);

// lwfw: 通知 agent 映射
sel4_set_mr(0, lwfw_dspace_id);  // dspace ID 通过消息寄存器传递
sel4_call_mrs(SYS_LWFW_AGENT_INIT, ...);
```

### 3.4 共享内存映射

```c
// agent: 映射共享内存到本进程虚拟地址空间
sys_dspace_map(dsid, &vaddr, &size, &attr);
lwfw_event_fifo = (struct lwfw_event_fifo *)vaddr;
```

**权限**: seL4 授予 `READ | WRITE` 权限，lwfw 和 agent 都可以读写

---

## 4. 事件合并机制 (Agent 端)

### 4.1 O(n²) 合并算法

```c
// event_handler.c:561-576
static int lwfw_agent_handle_evt(event_num, events)
{
    // O(n²) 合并
    for (i = 0; i < event_num; i++) {
        if (events[i].merged) continue;
        events[i].count = 1;

        for (j = i+1; j < event_num; j++) {
            if (same_event_type(events[i], events[j]) &&
                same_rule_id(events[i], events[j])) {
                events[i].count++;
                events[j].merged = true;
            }
        }
    }
}
```

### 4.2 性能分析

| 事件数 | 比较次数 | 时间 (假设 1M ops/ms) |
|--------|----------|---------------------|
| 64 | 2,016 | ~2ms |
| 512 | 130,816 | ~130ms |
| 1024 | 523,776 | ~524ms |

**问题**: 事件多时 CPU 占用高

### 4.3 优化方案

```c
// 优化: O(n) 哈希表合并
typedef struct {
    uint32_t event_type;
    uint32_t rule_id;
    uint32_t count;
} merged_event_t;

GHashTable *merged = g_hash_table_new(
    g_direct_hash,
    (GEqualFn)same_event_key
);

// 单次遍历 O(n)
for (i = 0; i < event_num; i++) {
    key = events[i].event_type << 16 | events[i].rule_id;
    entry = g_hash_table_lookup(merged, key);
    if (entry) {
        entry->count++;
    } else {
        g_hash_table_insert(merged, key, &events[i]);
    }
}
```

---

## 5. 定时通知机制

### 5.1 定时器线程问题

```c
// lwfw_notif.c:94-102
if (MAX_TIMER_VAL_PER_SECOND * loops++ >= event_notify_interval) {
    loops = 1;
    // 触发通知
}
```

**问题**: `event_notify_interval` 单位不一致导致粒度问题

### 5.2 几乎满阈值

```c
// lwfw_notif.c:75
if (lwfw_event_fifo_is_almost_full())
```

**几乎满定义**: `(put_idx + queue_size - get_idx) % queue_size > (queue_size * 3 / 4)`

---

## 6. 稳定性问题

### 6.1 定时器线程退出无重启

```c
// lwfw_notif.c:76-80
err = sys_time_sleep(tmsv_get_svc_ep(), &tv);
if (err) {
    LWFW_PRINTF_ERROR("timer thread sleep fail");
    break;  // 线程直接退出，无重启机制
}
```

**影响**: 热重载、事件通知永久失效

### 6.2 同步等待无超时

```c
// lwfw_notif.c:75
sync_sem_wait(&lwfw_wait_for_event);  // 永久阻塞
```

**影响**: 如果信号量出问题，线程永久阻塞

### 6.3 get_idx 丢失风险

```c
// 如果 agent 重启，get_idx 重置为 0
// 但 lwfw 不知道，以为还有旧事件
lwfw_event_fifo->get_idx = 0;  // agent 重启后
```

**缓解**: lwfw 通过 `LWFW_AGENT_IOCTL_GET_STATS` 查询 agent 状态

---

## 7. 优化建议汇总

| 优先级 | 问题 | 建议 | 复杂度 |
|--------|------|------|--------|
| **P1** | 事件合并 O(n²) | 使用哈希表 O(n) 合并 | 中 |
| **P2** | 定时器线程退出无重启 | 添加线程监控和重启机制 | 中 |
| **P2** | 同步等待无超时 | 添加超时机制 | 低 |
| **P3** | 批量确认可能重复消费 | 添加幂等消费 | 高 |
| **P3** | 定时粒度问题 | 修复 event_notify_interval 单位 | 低 |

---

## 8. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `lwfw_notif.c` | `lwfw_event_fifo_init` | FIFO 初始化 |
| `lwfw_notif.c` | `lwfw_event_push` | 事件推送 |
| `lwfw_notif.c` | `lwfw_notification_timer_thread` | 定时器线程 |
| `lwfw_notif.c` | `lwfw_notification_thread_loop` | 主通知循环 |
| `event_handler.c` | `lwfw_agent_evt_consume` | 事件消费 |
| `event_handler.c` | `lwfw_agent_handle_evt` | O(n²) 合并 |
