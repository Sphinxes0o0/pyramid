# TCP PCB 结构与 Timer 管理分析 — T-033

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: TCP PCB 结构、timer 管理、重传队列、状态转换

---

## 1. 概述

TCP PCB (Protocol Control Block) 是 lwIP 维护每个 TCP 连接状态的核心数据结构。

### 1.1 PCB 类型

| PCB 类型 | 描述 | 所在链表 |
|---------|------|---------|
| **tcp_listen_pcbs** | LISTEN 状态的 PCB (监听socket) | 单独链表 |
| **tcp_bound_pcbs** | 已绑定但未连接的 PCB | 单独链表 |
| **tcp_active_pcbs** | 活跃的 PCB (ESTABLISHED等) | 活跃链表 |
| **tcp_tw_pcbs** | TIME-WAIT 状态的 PCB | 时间等待链表 |

---

## 2. TCP PCB 结构

**文件**: `include/lwip/tcp.h:244-400`

```c
struct tcp_pcb {
    // ============================================
    // 基础字段
    // ============================================
    IP_PCB;                    // IP 地址 + port
    TCP_PCB_COMMON;           // state, next, errf

    u16_t remote_port;        // 远端端口 (主机序)

    // ============================================
    // TCP Flags
    // ============================================
    tcpflags_t flags;
#define TF_ACK_DELAY   0x01U   // 延迟 ACK
#define TF_ACK_NOW    0x02U   // 立即发送 ACK
#define TF_INFR        0x04U   // 快速恢复中
#define TF_CLOSEPEND   0x08U   // 等待发送 FIN
#define TF_RXCLOSED    0x10U   // 接收已关闭
#define TF_FIN         0x20U   // 本地已发送 FIN
#define TF_NODELAY     0x40U   // 禁用 Nagle 算法
#define TF_NAGLEMEMERR 0x80U   // Nagle + 内存错误
#define TF_WND_SCALE   0x0100U // 窗口缩放
#define TF_BACKLOGPEND 0x0200U // backlog 待处理
#define TF_TIMESTAMP   0x0400U // 时间戳选项
#define TF_RTO         0x0800U // RTO 定时器已触发
#define TF_SACK        0x1000U // SACK 选项

    // ============================================
    // Timer 相关
    // ============================================
    u8_t polltmr, pollinterval;  // 轮询定时器
    u8_t last_timer;              // 上次处理时间
    u32_t tmr;                    // 最后活动时间
    u32_t recv_tmr_start;         // 接收计时开始

    // ============================================
    // 接收方变量
    // ============================================
    u32_t rcv_nxt;           // 期望接收的下一个 seq
    tcpwnd_size_t rcv_wnd;   // 接收窗口大小
    tcpwnd_size_t rcv_ann_wnd;  // 宣告的接收窗口
    u32_t rcv_ann_right_edge;   // 宣告的窗口右边界

    // ============================================
    // 重传定时器
    // ============================================
    s16_t rtime;              // RTO 计时器 (当前值)
    u16_t mss;                // 最大段大小
    u32_t rttest;             // RTT 估计 (500ms ticks)
    u32_t rtseq;              // 被计时的 seq 编号
    s16_t sa, sv;             // Van Jacobson RTT 估计
    s16_t rto;                // RTO 值 (slow ticks)
    u8_t nrtx;                // 重传次数

    // ============================================
    // 快速重传/恢复
    // ============================================
    u8_t dupacks;            // 重复 ACK 数量
    u32_t lastack;           // 最高确认的 seq

    // ============================================
    // 拥塞控制
    // ============================================
    tcpwnd_size_t cwnd;      // 拥塞窗口
    tcpwnd_size_t ssthresh;   // 慢启动阈值

    u32_t rto_end;           // 最后一个 RTO 字节

    // ============================================
    // 发送方变量
    // ============================================
    u32_t snd_nxt;           // 下一个要发送的 seq
    u32_t snd_wl1, snd_wl2; // 上次窗口更新的 seq/ack
    u32_t snd_lbb;           // 下一个要缓冲的字节 seq
    tcpwnd_size_t snd_wnd;   // 发送窗口
    tcpwnd_size_t snd_wnd_max; // 对端宣告的最大窗口
    tcpwnd_size_t snd_buf;   // 发送缓冲区大小
    u16_t snd_queuelen;      // 发送队列中的 pbuf 数量

    // ============================================
    // Segment 队列
    // ============================================
    struct tcp_seg *unsent;  // 未发送的 segment
    struct tcp_seg *unacked;  // 已发送未确认的 segment
    struct tcp_seg *ooseq;   // 接收到的乱序 segment

    struct pbuf *refused_data; // 被上层拒绝的数据

    // ============================================
    // 回调函数
    // ============================================
    tcp_sent_fn sent;        // 发送完成回调
    tcp_recv_fn recv;        // 接收数据回调
    tcp_accept_fn accept;     // 连接接受回调
    void *callback_arg;      // 回调参数

    // ============================================
    // 本地/远端地址
    // ============================================
    struct ip_addr_info local_ip;  // 本地 IP
    struct ip_addr_info remote_ip; // 远端 IP
};
```

---

## 3. TCP 状态转换

### 3.1 TCP 状态

```c
enum tcp_state {
    CLOSED = 0,
    LISTEN,
    SYN_SENT,
    SYN_RCVD,
    ESTABLISHED,
    CLOSE_WAIT,
    FIN_WAIT_1,
    CLOSING,
    LAST_ACK,
    FIN_WAIT_2,
    TIME_WAIT
};
```

### 3.2 状态转换图

```
                              +--------+
         +------------------->| CLOSED |<------------------+
         |                     +--------+                     |
         |                                          |
         |  Passive Open                     +------------+
         |  +------------+                   | LISTEN    | (server)
         +->|            |                   +------------+
            |            |       +---------+       |
            +------------+------>|         |<------+ Active Open
            |                   |         |       |
            |  +------------+  |         |  +---------+
            |  |            |  |         |  |         |
            +->| SYN_SENT   |--+         +->| SYN_    |
               |            |             | RCVD    |
               +----------+-+              +---------+
                          |
            +--------------+---------------+
            |              |               |
       SYN  |         SYN/ACK         SYN/ACK
            |              |               |
            v              v               v
      +----------+   +-----------+   +-----------+
      |          |   |           |   |           |
      |          |   | ESTABLISHED|   |           |
      |          |   |           |   |           |
      +----------+   +-----------+   +-----------+
            |              |               |
            |     FIN      |       FIN    |
            v              |               v
      +----------+   +-----------+   +-----------+
      |          |   |           |   |           |
      |FIN_WAIT_1|   |CLOSE_WAIT|   | CLOSING   |
      |          |   |           |   |           |
      +----------+   +-----------+   +-----------+
            |              |               |
     FIN    |       FIN    |               |
            v              |               v
      +----------+   +-----------+   +-----------+
      |          |   |           |   |           |
      |FIN_WAIT_2|   | LAST_ACK  |   |TIME_WAIT  |
      |          |   |           |   |           |
      +----------+   +-----------+   +-----------+
            |              |               |
            |   ACK        |       timeout |
            +------------->+<--------------+
                           |
                           v
                     +-----------+
                     |           |
                     | TIME_WAIT |
                     |           |
                     +-----------+
```

---

## 4. Timer 管理

### 4.1 定时器概述

| 定时器 | 周期 | 作用 |
|--------|------|------|
| **tcp_fasttmr** | 25ms | 发送延迟 ACK、处理 FIN、处理被拒绝数据 |
| **tcp_slowtmr** | 50ms | 重传超时、keepalive、FIN-WAIT-2 超时 |

### 4.2 tcp_tmr — 主定时器

**文件**: `core/tcp.c:234-244`

```c
void tcp_tmr(void)
{
    // 每 25ms 调用一次 fast timer
    tcp_fasttmr();

    // 每 50ms (每两次调用一次) 调用 slow timer
    if (++tcp_timer & 1) {
        tcp_slowtmr();
    }
}
```

### 4.3 tcp_fasttmr — 快速定时器

**文件**: `core/tcp.c:1557-1600`

```c
void tcp_fasttmr(void)
{
    ++tcp_timer_ctr;

    for (pcb = tcp_active_pcbs; pcb != NULL; pcb = pcb->next) {
        if (pcb->last_timer != tcp_timer_ctr) {
            pcb->last_timer = tcp_timer_ctr;

            // 发送延迟的 ACK
            if (pcb->flags & TF_ACK_DELAY) {
                tcp_ack_now(pcb);
                tcp_output(pcb);
                tcp_clear_flags(pcb, TF_ACK_DELAY | TF_ACK_NOW);
            }

            // 发送待处理的 FIN
            if (pcb->flags & TF_CLOSEPEND) {
                tcp_clear_flags(pcb, TF_CLOSEPEND);
                tcp_close_shutdown_fin(pcb);
            }

            // 处理被上层拒绝的数据
            if (pcb->refused_data != NULL) {
                tcp_process_refused_data(pcb);
            }
        }
    }
}
```

### 4.4 tcp_slowtmr — 慢速定时器

**文件**: `core/tcp.c:1211-1500`

```c
void tcp_slowtmr(void)
{
    ++tcp_ticks;
    ++tcp_timer_ctr;

    for (pcb = tcp_active_pcbs; pcb != NULL; pcb = pcb->next) {
        pcb->last_timer = tcp_timer_ctr;

        // ============================================
        // Step 1: 重传超时处理
        // ============================================
        if (pcb->rtime >= pcb->rto) {
            // 触发重传
            tcp_rexmit_rto_prepare(pcb);

            // 指数退避
            if (pcb->state != SYN_SENT) {
                pcb->rto = ((pcb->sa >> 3) + pcb->sv) << tcp_backoff[nrtx];
            }

            // 拥塞控制
            pcb->ssthresh = min(cwnd, snd_wnd) >> 1;
            pcb->cwnd = mss;  // 进入慢启动

            tcp_rexmit_rto_commit(pcb);
        } else {
            ++pcb->rtime;  // RTO 计时器递增
        }

        // ============================================
        // Step 2: SYN_SENT 超时 (最大重传次数)
        // ============================================
        if (pcb->state == SYN_SENT && pcb->nrtx >= TCP_SYNMAXRTX) {
            // 移除 PCB，连接超时
        }

        // ============================================
        // Step 3: DATA 超时 (最大重传次数)
        // ============================================
        if (pcb->nrtx >= TCP_MAXRTX) {
            // 移除 PCB，连接中断
        }

        // ============================================
        // Step 4: FIN-WAIT-2 超时
        // ============================================
        if (pcb->state == FIN_WAIT_2) {
            if (tcp_ticks - pcb->tmr > TCP_FIN_WAIT_TIMEOUT) {
                // 超时，移除 PCB
            }
        }

        // ============================================
        // Step 5: Keepalive
        // ============================================
        if (pcb->state == ESTABLISHED || pcb->state == CLOSE_WAIT) {
            if (tcp_ticks - pcb->tmr > keep_idle + keep_cnt * keep_intvl) {
                // 发送 keepalive 或终止连接
            }
        }

        // ============================================
        // Step 6: OOSEQ 超时 (乱序队列)
        // ============================================
#if TCP_QUEUE_OOSEQ
        if (pcb->ooseq != NULL &&
            tcp_ticks - pcb->tmr >= pcb->rto * TCP_OOSEQ_TIMEOUT) {
            // 丢弃乱序数据
            tcp_free_ooseq(pcb);
        }
#endif
    }
}
```

---

## 5. 重传机制

### 5.1 RTT 估计 (Van Jacobson 算法)

**文件**: `tcp.c:1308-1311`

```c
// RTO = (sa >> 3) + sv
// sa: 平滑 RTT 估计
// sv: RTT 偏差
//
// 每次 RTT 采样后:
// new_srtt = (7/8) * old_srtt + (1/8) * sample
// new_rttvar = (3/4) * old_rttvar + (1/4) * |sample - srtt|
// RTO = srtt + 4 * rttvar
```

### 5.2 拥塞控制

```c
// 初始拥塞窗口 (IW)
cwnd = min(4 * MSS, max(2 * MSS, 4380))

// 慢启动
if (cwnd < ssthresh) {
    cwnd += MSS;  // 每个 ACK，cwnd 增加一个 MSS
}

// 拥塞避免
if (cwnd >= ssthresh) {
    cwnd += MSS * MSS / cwnd;  // 每个 ACK，cwnd 增加一小部分
}
```

### 5.3 快速重传

```c
// tcp_receive() 中
if (dupacks >= 3) {
    // 快速重传丢失的 segment
    tcp_rexmit_fast(pcb);
}
```

---

## 6. Segment 队列

### 6.1 unsent 队列

```
unsent (未发送)
    │
    ├── seg[0]: len=1000, seq=1000
    ├── seg[1]: len=800, seq=2000
    └── NULL

等待发送窗口，发送时移到 unacked
```

### 6.2 unacked 队列

```
unacked (已发送未确认)
    │
    ├── seg[0]: len=1000, seq=1000 (已发送，等待 ACK)
    └── NULL

收到 ACK 时移除，RTO 超时则重传
```

### 6.3 ooseq 队列 (out-of-sequence)

```
ooseq (乱序接收)
    │
    ├── seg[0]: len=1000, seq=3000 (比期望的 seq=2000 提前)
    └── NULL

填充后交给上层
```

---

## 7. PCB 链表管理

### 7.1 链表列表

```c
// tcp.c:181-182
struct tcp_pcb **const tcp_pcb_lists[] = {
    &tcp_listen_pcbs.pcbs,  // LISTEN 状态
    &tcp_bound_pcbs,        // 已绑定未连接
    &tcp_active_pcbs,       // 活跃连接
    &tcp_tw_pcbs            // TIME-WAIT 状态
};
```

### 7.2 PCB 注册

```c
// 注册到活跃链表
TCP_REG(&tcp_active_pcbs, pcb);

// 注册到 LISTEN 链表
TCP_REG(&tcp_listen_pcbs.pcbs, (struct tcp_pcb *)lpcb);
```

---

## 8. 总结

### 8.1 TCP PCB 核心字段

```
TCP PCB
    │
    ├─► 标识
    │     ├─► local_ip, local_port
    │     ├─► remote_ip, remote_port
    │     └─► state (TCP 状态)
    │
    ├─► 序列号
    │     ├─► snd_nxt, lastack (发送)
    │     └─► rcv_nxt (接收)
    │
    ├─► 窗口
    │     ├─► snd_wnd, cwnd, ssthresh (发送)
    │     └─► rcv_wnd (接收)
    │
    ├─► Timer
    │     ├─► rtime (重传)
    │     ├─► tmr (活动)
    │     └─► rto (超时值)
    │
    ├─► 队列
    │     ├─► unsent (待发送)
    │     ├─► unacked (已发未确认)
    │     └─► ooseq (乱序)
    │
    └─► 回调
          ├─► sent (发送完成)
          ├─► recv (接收数据)
          └─► accept (连接接受)
```

### 8.2 Timer 机制

```
tcp_tmr() [25ms]
    │
    ├─► tcp_fasttmr() [25ms]
    │     ├─► 延迟 ACK
    │     ├─► 待发送 FIN
    │     └─► 拒绝的数据
    │
    └─► tcp_slowtmr() [50ms]
          ├─► 重传超时
          ├─► SYN/DATA 最大重传
          ├─► FIN-WAIT-2 超时
          ├─► Keepalive
          └─► OOSEQ 超时
```

### 8.3 关键设计

1. **双链表**: 分离 LISTEN/ACTIVE/TIME-WAIT 便于管理
2. **RTT 估计**: Van Jacobson 算法
3. **拥塞控制**: 慢启动 + 拥塞避免 + 快速重传
4. **指数退避**: RTO 超时后指数增加等待时间
