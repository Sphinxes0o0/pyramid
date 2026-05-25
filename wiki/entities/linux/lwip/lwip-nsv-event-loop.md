---
type: entity
tags: [linux, lwip, network, nsv, event-loop, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# NSv Event Loop — Network Server Event Loop

## 定义

NSv (Network Service virtualized) 是 SafeOS 的**用户态网络服务器**，运行在 seL4 微内核之上，通过 `event_loop()` 处理所有应用的网络 socket 系统调用。

## NSv 线程结构

| 线程 | 功能 |
|------|------|
| **event_loop** | 处理系统调用 (seL4 IPC) |
| **select_thread** | 监控 socket 事件 (lwip_select) |
| **nic_rx_thread** | 接收 NIC 数据包 |
| **nic_tx_thread** | 发送数据包 (通过 sel4_signal) |

## event_loop 主事件循环

**文件**: `main.c:3400-3600`

```c
while (1) {
    // 接收 seL4 消息
    info = seL4_Recv(svc_ep, &badge);
    label = sel4_msg_info_get_label(info);
    pid_t pid = sys_get_pid_from_badge(badge);

    switch (label) {
        case SYS_NET_SOCKET:     sys_socket_nb(info, badge);  break;
        case SYS_NET_CONNECT:    sys_connect_nb(info, badge);  break;
        case SYS_NET_BIND:       sys_bind_nb(info, badge);    break;
        case SYS_NET_ACCEPT:    sys_accept(info, badge);     break;
        case SYS_NET_SENDTO:    sys_sendto_nb(info, badge);  break;
        case SYS_NET_RECVFROM:   sys_recvfrom_nb(info, badge); break;
        // ...
    }
}
```

## Socket Syscall 分发

| 系统调用 | 处理函数 |
|---------|---------|
| `SYS_NET_SOCKET` | `sys_socket_nb()` |
| `SYS_NET_BIND` | `sys_bind_nb()` |
| `SYS_NET_LISTEN` | `sys_listen_nb()` |
| `SYS_NET_ACCEPT` | `sys_accept()` |
| `SYS_NET_CONNECT` | `sys_connect_nb()` |
| `SYS_NET_SENDTO` | `sys_sendto_nb()` |
| `SYS_NET_RECVFROM` | `sys_recvfrom_nb()` |
| `SYS_NET_CLOSE` | `sys_close_nb()` |

## select_thread 事件监控

```c
void *select_thread(void *arg)
{
    while (1) {
        int fsel_ret = fast_select_wait(&fselect_mbox, &fsel_entry, 0);
        if (fsel_ret < 0) continue;

        int fd = fsel_entry.s;
        evt_type = fsel_entry.evt_type;

        if (evt_type == FAST_SELECT_EVT_READ) {
            work = ht_get(&select_read_ht, fd);
            do_work(work);  // 执行工作
        } else if (evt_type == FAST_SELECT_EVT_WRITE) {
            work = ht_get(&select_write_ht, fd);
            do_work(work);
        }
    }
}
```

## Socket 事件流程 (RECV)

```
Application                      NSv                        lwIP
    │                             │                          │
    │ ─── recvfrom() ───────────►│                          │
    │                             │ ──► work 加入 mbox      │
    │                             │◄── return (pending)      │
    │◄── return (等待) ───────────┤                          │
    │                             │    select_thread         │
    │                             │    检测到 READ 事件     │
    │                             │ ──► do_work(work)       │
    │                             │     └─► sys_recvfrom_nb │
    │                             │                          │
    │                             │ ──► 复制数据到用户空间   │
    │ ─── data ──────────────────►│                          │
```

## 与 lwIP 的交互

NSv 使用 lwIP 的 **netconn API** 与 lwIP 交互：
```c
struct netconn *conn = netconn_new(NETCONN_TCP);
netconn_bind(conn, addr, port);
netconn_listen(conn);
netconn_accept(conn, &new_conn);
netconn_send(conn, data);
netconn_recv(conn, buf);
```

## 与 Linux 对比

| 特性 | NSv | Linux |
|------|-----|-------|
| syscall 处理 | 单独的 event_loop 线程 | 内核线程池 |
| socket 监控 | select_thread + lwip_select | epoll/poll/select |
| 协议栈 | lwIP (用户态) | Linux TCP/IP |
| 异步模型 | mbox + work item | 事件驱动 |

## 相关概念

- [[entities/linux/lwip/lwip-sel4-function]] — 整体 lwIP 调用链
- [[entities/linux/lwip/lwip-sys-net-socket-api]] — BSD Socket API
- [[entities/linux/lwip/lwip-sys-net-send-recv]] — send/recv 流程

## 来源详情

- [[sources/safeos-lwip-extensions]]
