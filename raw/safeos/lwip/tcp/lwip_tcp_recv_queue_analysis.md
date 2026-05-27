# TCP 接收队列分析 — T-034

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: TCP 接收队列、backlog、zero-window、out-of-order 处理

---

## 1. 概述

TCP 接收队列管理是确保数据可靠传输的关键机制，包括：
1. **rcv_wnd**: 接收窗口，告知对方可发送的数据量
2. **rcv_nxt**: 期望接收的下一个序列号
3. **ooseq**: out-of-order 队列，存放乱序到达的分段
4. **backlog**: listen 队列中未接受的连接数

---

## 2. TCP PCB 中的接收相关字段

### 2.1 接收窗口字段

**文件**: `include/lwip/tcp.h:291-295`

```c
/* receiver variables */
u32_t rcv_nxt;              /* next seqno expected */
tcpwnd_size_t rcv_wnd;     /* receiver window available */
tcpwnd_size_t rcv_ann_wnd; /* receiver window to announce */
u32_t rcv_ann_right_edge;   /* announced right edge of window */
```

### 2.2 字段解释

| 字段 | 说明 |
|------|------|
| `rcv_nxt` | 期望接收的下一个字节序列号 |
| `rcv_wnd` | 当前接收窗口大小 (可用缓冲) |
| `rcv_ann_wnd` | 通告给对方的窗口大小 |
| `rcv_ann_right_edge` | 已通告的窗口右边界 |

### 2.3 发送窗口通告

```c
// TCP header 中的窗口字段
TCPH_HDRLEN_OFFSET_SET(tcphdr, TCP_HLEN / 4);
tcphdr->wnd = lwip_htons(TCPWND_MIN16(pcb->rcv_ann_wnd));
```

---

## 3. Out-of-Order 队列 (ooseq)

### 3.1 ooseq 数据结构

**文件**: `core/tcp_in.c`

```c
struct tcp_seg {
    struct tcp_seg *next;    /* 链表指针 */
    struct pbuf *p;          /* 包含数据和 TCP header 的 pbuf */
    u16_t len;               /* TCP 长度 */
    u8_t  flags;             /* 标志 (TF_SEG_OPTS_*) */
    struct tcp_hdr *tcphdr;  /* TCP header 指针 */
};

/* 在 tcp_pcb 中 */
struct tcp_pcb {
    // ...
    struct tcp_seg *ooseq;   /* out-of-order segment queue */
    // ...
};
```

### 3.2 ooseq 队列管理

**文件**: `core/tcp_in.c:1385-1420`

当收到乱序的分段时：

```c
// tcp_receive() 中处理 ooseq
if (TCP_SEQ_BETWEEN(seqno, pcb->rcv_nxt, pcb->rcv_nxt + pcb->rcv_wnd)) {
    // 序列号在接收窗口内但不是下一个期望的序列号
    // 添加到 ooseq 队列

    // 找到正确的插入位置 (按序列号排序)
    struct tcp_seg *prev = NULL;
    struct tcp_seg *seg = pcb->ooseq;
    while (seg && TCP_SEQ_LT(seqno, seg->tcphdr->seqno)) {
        prev = seg;
        seg = seg->next;
    }

    // 插入分段
    if (prev) {
        prev->next = new_seg;
    } else {
        pcb->ooseq = new_seg;
    }
    new_seg->next = seg;
}
```

### 3.3 ooseq 释放

```c
void tcp_free_ooseq(struct tcp_pcb *pcb)
{
    struct tcp_seg *seg = pcb->ooseq;
    while (seg) {
        struct tcp_seg *next = seg->next;
        tcp_seg_free(seg);
        seg = next;
    }
    pcb->ooseq = NULL;
}
```

---

## 4. Zero-Window

### 4.1 Zero-Window 机制

当接收缓冲区满时，`rcv_wnd` 可以变为 0，表示对方不能发送数据。

```c
// 当应用没有及时读取数据时
if (rcv_wnd == 0) {
    // 通告对方窗口为 0
    // 对方停止发送数据
}
```

### 4.2 Zero-Window Probe

**文件**: `core/tcp_in.c:429`

```c
/* this is a zero-window probe, we respond to it with current RCV.NXT */
if (TCPH_HDRLEN_FLAGS_OFFSET == TCP_ACK) {
    // 收到 zero-window probe，回复当前 RCV.NXT
    tcp_ack_now(pcb);
    tcp_output(pcb);
}
```

### 4.3 Zero-Window 恢复

```c
// tcp_recved() 被调用后
void tcp_recved(struct tcp_pcb *pcb, u16_t len)
{
    // 增加窗口
    pcb->rcv_wnd += len;
    if (pcb->rcv_wnd > TCP_WND_MAX(pcb)) {
        pcb->rcv_wnd = TCP_WND_MAX(pcb);
    }

    // 如果窗口变化足够大，发送 ACK 更新对方窗口
    if (wnd_inflation >= TCP_WND_UPDATE_THRESHOLD) {
        tcp_ack_now(pcb);
        tcp_output(pcb);
    }
}
```

---

## 5. Backlog (Listen 队列)

### 5.1 Backlog 数据结构

**文件**: `include/lwip/tcp.h:236-239`

```c
#if TCP_LISTEN_BACKLOG
    u8_t backlog;          /* backlog 值，accept 前最大连接数 */
    u8_t accepts_pending;  /* 已接受但未调用的连接数 */
#endif
```

### 5.2 Backlog 处理

**文件**: `core/tcp.c:832-855`

```c
struct tcp_pcb *tcp_listen_with_backlog(struct tcp_pcb *pcb, u8_t backlog)
{
    pcb->backlog = backlog;
    return tcp_listen(pcb);
}
```

### 5.3 延迟 Accept

**文件**: `core/tcp.c:284-317`

```c
// 延迟接受连接，用于控制 accept() 调用频率
tcp_backlog_delayed(struct tcp_pcb *pcb)
{
    if (pcb->accepts_pending > 0) {
        pcb->accepts_pending--;
    }
}

// 增加 backlog
tcp_backlog_accepted(struct tcp_pcb *pcb)
{
    if (pcb->backlog > 0) {
        pcb->accepts_pending++;
    }
}
```

### 5.4 SYN Queue 与 Accept Queue

```
TCP 连接建立 (三次握手):
  │
  ├── SYN (client → server)
  │
  ├── 服务器回复 SYN-ACK，连接进入 SYN Queue
  │
  └── 客户端回复 ACK，连接移到 Accept Queue
       │
       └── accept() 从 Accept Queue 取走连接
```

---

## 6. 数据接收流程

### 6.1 接收数据处理

**文件**: `core/tcp_in.c:1489-1620`

```c
static void tcp_receive(struct tcp_pcb *pcb)
{
    // 检查 ooseq 队列
    if (pcb->ooseq != NULL) {
        // ooseq 有数据，检查是否可以合并
        if (pcb->rcv_nxt == pcb->ooseq->tcphdr->seqno) {
            // 下一个期望的序列号正好是 ooseq 的第一个
            // 处理 ooseq 中的分段
            while (pcb->ooseq &&
                   pcb->ooseq->tcphdr->seqno == pcb->rcv_nxt) {
                // 提取分段
                struct tcp_seg *cseg = pcb->ooseq;
                pcb->ooseq = cseg->next;

                // 更新 rcv_nxt
                pcb->rcv_nxt += TCP_TCPLEN(cseg);

                // 发送数据给应用
                tcp_recved(pcb, cseg->len);
                tcp_seg_free(cseg);
            }
        }
    }

    // 处理当前到达的分段 (in-sequence)
    if (seqno == pcb->rcv_nxt) {
        // 是期望的下一个序列号
        pcb->rcv_nxt += TCP_TCPLEN(seg);

        // 发送给应用
        if (recv_flags & TCP_recv_FLAG_DATA) {
            // 调用应用的 recv 回调
            TCP_EVENT_RECV(pcb, seg->p, 0, err);
        }

        // 如果有 FIN，更新状态
        if (flags & TCP_FIN) {
            // 处理 FIN
        }
    }
}
```

### 6.2 rcv_wnd 更新

```c
// tcp_process() 中
if (TCP_SEQ_BETWEEN(seqno, pcb->rcv_nxt, pcb->rcv_nxt + pcb->rcv_wnd)) {
    // 数据在接收窗口内
    // ...
} else {
    // 数据不在窗口内，丢弃
    LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_process: unacceptable seqno %"U32_F" rcv_nxt %"U32_F"\n",
                                seqno, pcb->rcv_nxt));
}
```

---

## 7. TCP Window Scale

### 71. 窗口扩大因子

**文件**: `include/lwip/tcp.h:262-264`

```c
#if LWIP_WND_SCALE
#define TF_WND_SCALE   0x0100U /* Window Scale option enabled */
#endif
```

### 7.2 rcv_wnd 最大值

```c
// TCP_WND_MAX 计算
#define TCP_WND_MAX(pcb) ((tcpwnd_size_t)TCP_WND_SCALED(pcb->rcv_wnd_max))
```

---

## 8. 总结

### 8.1 接收队列架构

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

### 8.2 关键字段

| 字段 | 说明 |
|------|------|
| `rcv_nxt` | 期望的下一个序列号 |
| `rcv_wnd` | 当前接收窗口 |
| `rcv_ann_wnd` | 通告的窗口大小 |
| `ooseq` | 乱序分段队列 |

### 8.3 Zero-Window 流程

```
应用繁忙，来不及读取
    │
    ▼
rcv_wnd → 0
    │
    ▼
发送窗口通告 = 0
    │
    ▼
对方停止发送 (除 zero-window probe)
    │
    ▼
应用调用 tcp_recved()
    │
    ▼
rcv_wnd 恢复
    │
    ▼
发送窗口更新 ACK
```

### 8.4 Backlog 流程

```
socket() → bind() → listen(backlog=5)
    │
    ▼
客户端连接进入 SYN Queue
    │
    ▼
三次握手完成，移到 Accept Queue
    │
    ▼
accept() 从 Accept Queue 取走
```
