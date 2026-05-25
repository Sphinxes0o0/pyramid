---
type: entity
tags: [linux, lwip, network, tcp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP TCP Receive Queue

## 定义

TCP 接收队列管理确保数据可靠传输，包括 rcv_wnd (接收窗口)、rcv_nxt (期望序列号)、ooseq (乱序队列)、backlog (listen 队列)。

## 接收窗口字段

```c
u32_t rcv_nxt;              /* next seqno expected */
tcpwnd_size_t rcv_wnd;       /* receiver window available */
tcpwnd_size_t rcv_ann_wnd;   /* receiver window to announce */
u32_t rcv_ann_right_edge;    /* announced right edge of window */
```

## Out-of-Order 队列 (ooseq)

当收到乱序的分段时：

```c
// tcp_receive() 中处理 ooseq
if (seqno == pcb->rcv_nxt) {
    // 顺序到达
    pcb->rcv_nxt += TCP_TCPLEN(seg);
    recv_data = inseg.p;  // 传递给应用
    tcp_ack(pcb);  // 发送 ACK
} else {
    // 乱序到达 → 加入 ooseq 队列
    tcp_oos_insert_segment(...);
    tcp_send_empty_ack(pcb);
}
```

## Zero-Window 机制

当接收缓冲区满时：
```c
// 收到 zero-window probe，回复当前 RCV.NXT
tcp_ack_now(pcb);
tcp_output(pcb);
```

## Backlog (Listen 队列)

```c
struct tcp_pcb_listen {
    u8_t backlog;          /* backlog 值，accept 前最大连接数 */
    u8_t accepts_pending;  /* 已接受但未调用的连接数 */
};
```

## 接收数据处理

```
接收到的分段
    │
    ▼
tcp_input()
    │
    ▼
┌─────────────────────────────────────────┐
│  序列号 == rcv_nxt?                      │
│    ├─ YES: 处理数据，更新 rcv_nxt        │
│    │           └─► 尝试合并 ooseq       │
│    └─ NO: 添加到 ooseq 队列            │
└─────────────────────────────────────────┘
    │
    ▼
应用调用 tcp_recved()
    │
    ▼
更新 rcv_wnd，发送 ACK
```

## 相关概念

- [[entities/linux/lwip/lwip-tcp-input]] — tcp_receive 处理 rcv_nxt/ooseq
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 中的 rcv_nxt/rcv_wnd/ooseq 字段
- [[entities/linux/lwip/lwip-tcp-socket]] — backlog 在 listen 时设置
