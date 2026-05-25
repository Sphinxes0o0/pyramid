---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Notification Mechanism

## 定义

LWFW 事件通知机制通过**双线程架构** (通知线程 + 定时器线程) 和**共享内存 FIFO**，将防火墙事件从内核态传递到用户态 `lwfw_agent` 守护进程。

## 整体架构

```
lwfw (lwIP 进程)                              lwfw_agent (守护进程)
      │                                              │
      │ sys_dspace_create(SHM)                       │
      │────────── 共享内存 lwfw_event_fifo ──────────►│
      │                                              │
      │ sys_svc_wait(NSV_LWFW_AGENT_SVC_NAME)       │
      │────────── 获取 agent endpoint ───────────────►│
      │                                              │
      │ lwfw_notification_thread ──────────────────┐  │
      │   ├─ lwfw_notification_timer_thread        │  │
      │   │     └─ sel4_call(SYS_LWFW_AGENT_NOTIFY)│  │
      │   └─ lwfw_notification_thread_loop          │  │
      │         └─ sync_sem_wait() 阻塞等事件      │  │
      │──────────────────────────────────────────────┘  │
      │                                              │
      │ ←─────── JSON 日志文件 (events_*.log) ───────┘
```

## 事件推送

```c
err_t lwfw_event_push(struct lwfw_event *event)
{
  next_put = (put_idx + 1) % queue_size;
  if (next_put == get_idx) return ERR_MEM;

  memcpy(&events[next_put], event, sizeof(*event));
  dmb(ish);  // ARM 内存屏障
  put_idx = next_put;

  if (lwfw_event_fifo_is_almost_full())
    sync_sem_post(&lwfw_wait_for_event);  // 信号通知线程

  return ERR_OK;
}
```

## 双线程

| 线程 | 职责 |
|------|------|
| `lwfw_notification_thread` | 主线程，初始化 + 循环处理事件 |
| `lwfw_notification_timer_thread` | 定时器线程，定期检查 FIFO + 触发热重载 |

## 定时器线程任务

```c
static void lwfw_notification_timer_thread()
{
  while (1) {
    sys_time_sleep(1s);

    // 热重载检查
    if (cfg_in_reloading) {
      lwfw_p->policy->rule_tables[IN_TABLE].state = DISABLE;
      lwfw_config_reload_manifest(cfg_path);  // ← P0: 热重载窗口期无防护
      lwfw_p->policy->rule_tables[IN_TABLE].state = ENABLE;
    }

    // 定期事件通知
    if (timer >= event_notify_interval && FIFO not empty)
      sel4_call(SYS_LWFW_AGENT_NOTIFY);

    // 速率限制 worker
    if (timer >= pkt_rlimit_interval)
      lwfw_rule_rlimit_worker();
  }
}
```

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 热重载期间 Ingress 无防护 | P0 | 重载时先 DISABLE 防火墙 |
| 定时器线程退出无重启 | P1 | sleep 失败后线程直接退出 |
| 同步等待无超时 | P2 | `sync_sem_wait` 永久阻塞风险 |
| 几乎满阈值粒度问题 | P3 | `event_notify_interval` 最小粒度 1 秒 |

## 相关概念

- [[entities/linux/lwfw/lwfw-ipc-mechanism]] — IPC 机制
- [[entities/linux/lwfw/lwfw-agent]] — Agent 守护进程
- [[entities/linux/lwfw/lwfw-hotswap-analysis]] — 热切换问题
