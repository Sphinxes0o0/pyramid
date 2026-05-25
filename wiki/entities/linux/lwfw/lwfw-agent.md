---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Agent

## 定义

`lwfw_agent` 是 LWFW 的**用户态守护进程**，负责从共享内存 FIFO 消费安全事件、将事件 JSON 格式化写入日志文件、处理配置文件热重载的软链接切换。

## 整体架构

```
lwfw_agent (守护进程)
  │
  ├─ lwfw_agent_init()                    ← 初始化
  │     ├─ wait_for_fs_mount()
  │     ├─ mkdir(LWFW_EVENT_DIR)
  │     ├─ rotate_event_files()
  │     └─ timer_thread_init()
  │
  ├─ lwfw_agent_svc_evt()                 ← IPC 消息处理
  │     ├─ SYS_LWFW_AGENT_INIT:          ← 映射共享内存
  │     ├─ SYS_LWFW_AGENT_NOTIFY:         ← 消费事件
  │     └─ SYS_LWFW_AGENT_IOCTL:          ← IOCTL 控制
  │
  ├─ timer_thread()                        ← 定时任务
  │     ├─ reload_config()
  │     ├─ rotate_event_files()
  │     └─ 清除日志计数
  │
  └─ proc_evt_thread()                     ← 进程退出监听
```

## IPC 消息处理

```c
sel4_msg_info_t lwfw_agent_svc_evt(badge, info, ctx)
{
  switch (label) {
    case SYS_LWFW_AGENT_INIT:
      return lwfw_agent_evt_consumer_init();

    case SYS_LWFW_AGENT_NOTIFY:
      return lwfw_agent_evt_consume();

    case SYS_LWFW_AGENT_IOCTL:
      return lwfw_agent_ioctl();
  }
}
```

## 事件消费

```c
sel4_msg_info_t lwfw_agent_evt_consume(info, pid, ctx)
{
  // 1. 从共享内存 FIFO 读取事件
  get_idx = lwfw_event_fifo->get_idx;
  put_idx = lwfw_event_fifo->put_idx;
  event_num = (put_idx - get_idx) % queue_size;

  // 2. 复制到本地堆内存
  memcpy(lwfw_events, &events[get_idx+1], event_num * event_len);
  dmb(ish);
  lwfw_event_fifo->get_idx = put_idx;  // 批量确认

  // 3. 处理事件
  lwfw_agent_handle_evt(event_num, lwfw_events);
}
```

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 事件合并 O(n²) | P1 | 每批512事件需130k次比较 |
| scandir 每次遍历 | P2 | 目录文件多时性能差 |
| 重试计数器上限 | P3 | `LWFW_MAX_RETRY` 达到后不再尝试 |
| timer_thread 退出 | P1 | sleep 失败后线程直接退出 |

## 相关概念

- [[entities/linux/lwfw/lwfw-agent-log-system]] — 日志系统详解
- [[entities/linux/lwfw/lwfw-ipc-mechanism]] — IPC 机制
- [[entities/linux/lwfw/lwfw-notif]] — 通知机制
