---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW IPC Mechanism

## 定义

LWFW 使用 **共享内存 FIFO** + **seL4 IPC** 实现内核态防火墙 (`lwfw`) 与用户态守护进程 (`lwfw_agent`) 之间的高效事件传递。

## 共享内存 FIFO 架构

```
lwfw (内核进程)                          lwfw_agent (用户态进程)
      │                                          │
      │ sys_dspace_create(SHM)                  │
      │────────── 共享内存 lwfw_event_fifo ──────►│
      │                                          │
      │ sys_svc_wait(NSV_LWFW_AGENT_SVC_NAME)  │
      │────────── 获取 agent endpoint ────────────►│
      │                                          │
      │ lwfw_notification_thread ────────────────┐│
      │   └─ lwfw_event_push(event)              ││
      │──────────────────────────────────────────┘│
      │                                          │
      │ ←─────── seL4 IPC NOTIFY ────────────────┘
```

## FIFO 数据结构

```c
struct lwfw_event_fifo {
  sync_mutex_t prod_lock;         // 多生产者锁
  lwfw_agent_parameters_t params; // 配置参数
  volatile uint32_t get_idx;     // 消费者索引 (agent 写)
  volatile uint32_t put_idx;     // 生产者索引 (lwfw 写)
  uint32_t queue_size;          // 事件槽数量 (512)
  lwfw_event events[1];          // 变长数组
};
```

## 生产者 (lwfw)

```c
err_t lwfw_event_push(struct lwfw_event *event)
{
  next_put = (put_idx + 1) % queue_size;
  if (next_put == get_idx) return ERR_MEM;  // FIFO 满

  memcpy(&events[next_put], event, sizeof(*event));
  dmb(ish);              // ARM 内存屏障
  put_idx = next_put;    // 发布数据
  return ERR_OK;
}
```

## 消费者 (lwfw_agent)

```c
sel4_msg_info_t lwfw_agent_evt_consume(info, pid, ctx)
{
  // 批量读取
  get_idx = lwfw_event_fifo->get_idx;
  put_idx = lwfw_event_fifo->put_idx;
  event_num = (put_idx - get_idx) % queue_size;

  // 复制到本地堆内存
  memcpy(lwfw_events, &events[get_idx+1], event_num * event_len);
  dmb(ish);
  lwfw_event_fifo->get_idx = put_idx;  // 批量确认

  lwfw_agent_handle_evt(event_num, lwfw_events);
  return 0;
}
```

## 初始化握手

```c
// lwfw: 创建共享内存
fifo_size = ALIGN_UP(sizeof(lwfw_event_fifo) + event_num * sizeof(event), 4KB);
sys_dspace_create(fifo_size, SHM|WRITE|CACHE, &addr, &lwfw_dspace_id);
sys_dspace_grant(lwfw_dspace_id, lwfw_agent_ep, ...);
sel4_set_mr(0, lwfw_dspace_id);
sel4_call_mrs(SYS_LWFW_AGENT_INIT, ...);

// agent: 映射共享内存
sys_dspace_map(dsid, &vaddr, &size, &attr);
lwfw_event_fifo = (void*)vaddr;
```

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 事件合并 O(n²) | P1 | 每批512事件需130k次比较 |
| 定时器线程退出无重启 | P2 | sleep 失败后线程直接退出 |
| 批量确认可能重复消费 | P3 | agent 重启后 get_idx 丢失 |

## 相关概念

- [[entities/linux/lwfw/lwfw-notif]] — 通知机制
- [[entities/linux/lwfw/lwfw-agent]] — Agent 守护进程
- [[entities/linux/lwfw/lwfw-agent-log-system]] — 日志系统
