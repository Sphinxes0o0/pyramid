# NSv Event Loop 分析 — T-090

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: NSv 主事件循环、select/poll 实现、socket 事件分发

---

## 1. 概述

NSv (Network Service virtualized) 是 SafeOS 的**用户态网络服务**，运行在 seL4 微内核之上：

1. 处理应用的网络 socket 系统调用
2. 与 lwIP 协议栈交互
3. 管理 select/poll 事件

### 1.1 NSv 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Application                                   │
│                    (iperf, ping, lwfwcfg)                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼ seL4 IPC (syscall)
┌─────────────────────────────────────────────────────────────────────┐
│                          NSv Network Server                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     event_loop()                             │   │
│  │                   (主事件循环)                               │   │
│  │  seL4_Recv(svc_ep, &badge) → 处理系统调用                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   select_thread()                           │   │
│  │                 (lwip_select 监控)                         │   │
│  │  fast_select_wait() → 等待 socket 事件                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                  │                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      lwIP Stack                             │   │
│  │              (运行在 tcpip_thread)                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       seL4 Microkernel                              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       NIC Driver (PFE)                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 线程结构

### 2.1 NSv 线程列表

| 线程 | 优先级 | 功能 |
|------|--------|------|
| **event_loop** | SYS_PRIO_NETSVR | 处理系统调用 |
| **select_thread** | SYS_PRIO_NETSVR | 监控 socket 事件 |
| **nic_rx_thread** | SYS_PRIO_NETSVR | 接收 NIC 数据包 |
| **nic_tx_thread** | - | 发送数据包 (通过 sel4_signal) |

### 2.2 线程通信

```
event_loop          select_thread         nic_rx_thread
    │                    │                     │
    │                    │                     │
    │◄─── mbox ─────────┤                     │
    │      (work)       │                     │
    │                    │                     │
    │              seL4 IPC                 │
    │◄───────────────────┘                     │
    │                                         │
    │◄─────────────────────────────────────────┘
    │              (pbuf via shm)
```

---

## 3. event_loop 主事件循环

**文件**: `main.c:3400-3600`

### 3.1 核心循环

```c
static void event_loop(void)
{
    sel4_msg_info_t info;
    sel4_word badge;
    sel4_word label;

    while (1) {
        // 接收 seL4 消息
        info = seL4_Recv(svc_ep, &badge);
        label = sel4_msg_info_get_label(info);

        // 获取 PID
        pid_t pid = sys_get_pid_from_badge(badge);

        // 分发系统调用
        switch (label) {
            case SYS_NET_SOCKET:
                sys_socket_nb(info, badge);
                break;
            case SYS_NET_CONNECT:
                sys_connect_nb(info, badge);
                break;
            case SYS_NET_BIND:
                sys_bind_nb(info, badge);
                break;
            case SYS_NET_ACCEPT:
                sys_accept(info, badge);
                break;
            case SYS_NET_SENDTO:
                sys_sendto_nb(info, badge);
                break;
            case SYS_NET_RECVFROM:
                sys_recvfrom_nb(info, badge);
                break;
            // ... 其他系统调用
        }
    }
}
```

### 3.2 系统调用分发

| 系统调用 | 处理函数 | 说明 |
|---------|---------|------|
| `SYS_NET_SOCKET` | `sys_socket_nb()` | 创建 socket |
| `SYS_NET_BIND` | `sys_bind_nb()` | 绑定地址端口 |
| `SYS_NET_LISTEN` | `sys_listen_nb()` | 监听 |
| `SYS_NET_ACCEPT` | `sys_accept()` | 接受连接 |
| `SYS_NET_CONNECT` | `sys_connect_nb()` | 发起连接 |
| `SYS_NET_SENDTO` | `sys_sendto_nb()` | 发送数据 |
| `SYS_NET_RECVFROM` | `sys_recvfrom_nb()` | 接收数据 |
| `SYS_NET_SENDMSG` | `sys_sendmsg_nb()` | 发送消息 |
| `SYS_NET_RECVMSG` | `sys_recvmsg_nb()` | 接收消息 |

---

## 4. select_thread 事件监控

**文件**: `main.c:5560-5700`

### 4.1 select 实现架构

```
┌─────────────────────────────────────────────────────────────┐
│                    select_thread()                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              fast_select_wait()                      │   │
│  │         (使用 lwip_select 内部实现)                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                 │
│         ┌─────────────┴─────────────┐                    │
│         ▼                           ▼                     │
│  ┌─────────────┐             ┌─────────────┐              │
│  │ read_set    │             │ write_set   │              │
│  │  (ht)       │             │  (ht)       │              │
│  └─────────────┘             └─────────────┘              │
│         │                           │                     │
│         ▼                           ▼                     │
│  ┌─────────────────────────────────────────────┐          │
│  │              do_work()                        │          │
│  │    处理 READ/WRITE/ACCEPT/CONNECT            │          │
│  └─────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 fast_select_wait

```c
void *select_thread(void *arg)
{
    fselect_evt_entry_t fsel_entry;
    struct lwip_work *work;

    while (1) {
        // 等待 socket 事件
        int fsel_ret = fast_select_wait(&fselect_mbox, &fsel_entry, 0);

        if (fsel_ret < 0) continue;

        int fd = fsel_entry.s;
        evt_type = fsel_entry.evt_type;

        if (evt_type == FAST_SELECT_EVT_NOTIFY) {
            // 新工作通知
            work = (struct lwip_work *)fsel_entry.user_data;
            fast_select_register(work->socket, &fselect_mbox, NULL);
            // 根据命令类型添加到 read/write hashtable
        } else if (evt_type == FAST_SELECT_EVT_READ) {
            // 读事件就绪
            work = ht_get(&select_read_ht, fd);
            do_work(work);  // 执行工作
        } else if (evt_type == FAST_SELECT_EVT_WRITE) {
            // 写事件就绪
            work = ht_get(&select_write_ht, fd);
            do_work(work);  // 执行工作
        }
    }
}
```

### 4.3 工作项结构

```c
struct lwip_work {
    int socket;          // socket fd
    int cmd;             // 命令类型
    void *conn;          // lwIP netconn
    void *buf;           // 数据缓冲区
    size_t len;          // 数据长度
    // ... 其他字段
};
```

---

## 5. Socket 事件流程

### 5.1 RECV 流程

```
Application                      NSv                        lwIP
    │                             │                          │
    │ ─── recvfrom() ────────────►│                          │
    │                             │                          │
    │                             │ ──► work 加入 mbox      │
    │                             │                          │
    │                             │◄── return (pending)      │
    │◄── return (等待) ───────────┤                          │
    │                             │                          │
    │                             │    select_thread         │
    │                             │    检测到 READ 事件     │
    │                             │                          │
    │                             │ ──► do_work(work)      │
    │                             │     └─► sys_recvfrom_nb │
    │                             │           └─► netconn_recv
    │                             │                          │
    │                             │                          │◄─── pbuf 数据
    │                             │                          │
    │                             │ ──► 复制数据到用户空间   │
    │                             │                          │
    │ ─── data ──────────────────►│                          │
    │◄── return ──────────────────┤                          │
```

### 5.2 ACCEPT 流程

```
Application                      NSv                        lwIP
    │                             │                          │
    │ ─── accept() ──────────────►│                          │
    │                             │                          │
    │                             │ ──► work 加入 mbox      │
    │                             │                          │
    │                             │◄── return (pending)     │
    │◄── return (等待) ───────────┤                          │
    │                             │                          │
    │                             │    select_thread         │
    │                             │    检测到 ACCEPT 事件    │
    │                             │                          │
    │                             │ ──► do_work(work)       │
    │                             │     └─► sys_accept      │
    │                             │           └─► netconn_accept
    │                             │                          │
    │                             │                          │◄─── TCP PCB 已建立
    │                             │                          │
    │                             │ ──► 返回新 socket fd     │
    │ ─── new_fd ────────────────►│                          │
    │◄── return ──────────────────┤                          │
```

---

## 6. mbox 机制

### 6.1 fast_select 内部 mailbox

```c
// 用于 select_thread 内部通信
struct fselect_mbox {
    void *msg[SYS_MAX_OPEN_FILES];  // 消息队列
    int put_idx;                     // 写入位置
    int get_idx;                     // 读取位置
    sys_mutex_t lock;               // 保护锁
};
```

### 6.2 工作分发

```c
// event_loop 中将工作加入 select_thread 的 mbox
int fast_select_register(int fd, struct fselect_mbox *mbox, void *user_data)
{
    mbox->msg[mbox->put_idx] = user_data;
    mbox->put_idx = (mbox->put_idx + 1) % SIZE;
}
```

---

## 7. 与 lwIP 的交互

### 7.1 netconn API

NSv 使用 lwIP 的 **netconn API** (顺序 socket API) 与 lwIP 交互：

```c
// socket 创建
struct netconn *conn = netconn_new(NETCONN_TCP);

// bind
netconn_bind(conn, addr, port);

// listen
netconn_listen(conn);

// accept
netconn_accept(conn, &new_conn);

// send/recv
netconn_send(conn, data);
netconn_recv(conn, buf);
```

### 7.2 异步处理

NSv 使用 **lwip_select** 实现异步 socket：

```c
// lwIP select 实现
int lwip_select(int maxfdp, fd_set *readset, fd_set *writeset,
                fd_set *exceptset, struct timeval *timeout)
{
    // 检查 socket 状态
    // 如果没有就绪，阻塞等待
    // 返回就绪的 socket 数量
}
```

---

## 8. 性能特征

### 8.1 延迟来源

| 阶段 | 延迟 |
|------|------|
| seL4 IPC (syscall) | ~1-5 μs |
| mbox 入队/出队 | ~100 ns |
| lwip_select 等待 | ~10-100 μs |
| 数据复制 (pbuf → user) | ~100-500 ns |

### 8.2 瓶颈

1. **event_loop 单线程**: 所有 syscall 串行处理
2. **lwip_select**: O(n) 遍历所有 socket
3. **数据复制**: pbuf → 用户空间需要复制

---

## 9. 总结

### 9.1 NSv 架构

```
event_loop (syscall)          select_thread (socket 监控)
       │                              │
       │◄─── mbox (work)            │
       │                              │
       └──────────────────────────────┤
                                      │
                                      ▼
                              do_work()
                                      │
                                      ▼
                              lwIP netconn API
                                      │
                                      ▼
                              tcpip_thread
```

### 9.2 关键设计

1. **双线程**: event_loop 处理 syscall，select_thread 监控 socket
2. **lwip_select**: 基于 lwIP 的 select 实现
3. **mbox 通信**: select_thread 和 do_work 通过 mbox 通信
4. **无锁**: 分离 syscall 和 socket 监控减少锁竞争

### 9.3 与 Linux 对比

| 特性 | NSv | Linux |
|------|-----|-------|
| **syscall 处理** | 单独的 event_loop 线程 | 内核线程池 |
| **socket 监控** | select_thread + lwip_select | epoll/poll/select |
| **协议栈** | lwIP (内核态) | Linux TCP/IP |
| **异步模型** | mbox + work item | 事件驱动 |
