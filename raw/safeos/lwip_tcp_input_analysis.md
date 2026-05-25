# tcp_input 分析 — T-030

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: tcp_input 函数：TCP 状态机、segment 分片重组、out-of-order 处理、拥塞控制

---

## 1. 概述

`tcp_input()` 是 lwIP 的 **TCP 层入口函数**，负责：
1. TCP header 解析和 checksum 校验
2. demultiplex — 找到对应的 PCB (Protocol Control Block)
3. TCP 状态机处理 (tcp_process)
4. 数据接收和 ACK 处理 (tcp_receive)

### 1.1 调用链

```
ip4_input()
    │
    ▼
tcp_input(p, inp)
    │
    ├─► TCP Header 解析
    │     - 检查长度、checksum
    │     - 提取 src/dst port, seqno, ackno, flags
    │
    ├─► Demultiplex — 找 PCB
    │     - tcp_active_pcbs (已连接)
    │     - tcp_tw_pcbs (TIME_WAIT)
    │     - tcp_listen_pcbs (监听中)
    │
    ├─► tcp_process() — 状态机
    │     │
    │     ├─► RST 处理
    │     ├─► SYN 处理
    │     ├─► tcp_receive() — 数据接收
    │     └─► tcp_output() — 发送响应
    │
    └─► 回调通知 application
```

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/core/tcp_in.c:118`

### 2.1 tcp_input — 主入口

```c
// tcp_in.c:118-595
void
tcp_input(struct pbuf *p, struct netif *inp)
{
    struct tcp_pcb *pcb, *prev;
    struct tcp_pcb_listen *lpcb;

    // ============================================
    // Step 1: TCP Header 解析
    // ============================================
    tcphdr = (struct tcp_hdr *)p->payload;

    // 检查长度
    if (p->len < TCP_HLEN) {
        goto dropped;
    }

    // 丢弃广播/多播
    if (ip_addr_isbroadcast(ip_current_dest_addr(), ip_current_netif()) ||
        ip_addr_ismulticast(ip_current_dest_addr())) {
        goto dropped;
    }

    // ============================================
    // Step 2: Checksum 校验
    // ============================================
    #if CHECKSUM_CHECK_TCP
    IF__NETIF_CHECKSUM_ENABLED(inp, NETIF_CHECKSUM_CHECK_TCP) {
        u16_t chksum = ip_chksum_pseudo(p, IP_PROTO_TCP, ...);
        if (chksum != 0) {
            goto dropped;  // checksum 失败
        }
    }
    #endif

    // ============================================
    // Step 3: 解析 TCP Header 字段
    // ============================================
    hdrlen_bytes = TCPH_HDRLEN_BYTES(tcphdr);
    tcphdr->src = lwip_ntohs(tcphdr->src);
    tcphdr->dest = lwip_ntohs(tcphdr->dest);
    seqno = tcphdr->seqno = lwip_ntohl(tcphdr->seqno);
    ackno = tcphdr->ackno = lwip_ntohl(tcphdr->ackno);
    flags = TCPH_FLAGS(tcphdr);

    // ============================================
    // Step 4: Demultiplex — 找 PCB
    // ============================================

    // 4.1 先查找 active_pcbs (已连接)
    for (pcb = tcp_active_pcbs; pcb != NULL; pcb = pcb->next) {
        // 检查 netif 绑定
        if ((pcb->netif_idx != NETIF_NO_INDEX) &&
            (pcb->netif_idx != netif_get_index(ip_data.current_input_netif))) {
            continue;
        }
        // 匹配 4-tuple
        if (pcb->remote_port == tcphdr->src &&
            pcb->local_port == tcphdr->dest &&
            ip_addr_cmp(&pcb->remote_ip, ip_current_src_addr()) &&
            ip_addr_cmp(&pcb->local_ip, ip_current_dest_addr())) {
            // 移动到链表头部 ( locality optimization )
            if (prev != NULL) {
                prev->next = pcb->next;
                pcb->next = tcp_active_pcbs;
                tcp_active_pcbs = pcb;
            }
            break;
        }
        prev = pcb;
    }

    // 4.2 TIME_WAIT 状态
    if (pcb == NULL) {
        for (pcb = tcp_tw_pcbs; pcb != NULL; pcb = pcb->next) {
            // 类似匹配逻辑...
            tcp_timewait_input(pcb);
            pbuf_free(p);
            return;
        }
    }

    // 4.3 LISTEN 状态 (SYN 监听)
    if (pcb == NULL) {
        for (lpcb = tcp_listen_pcbs.listen_pcbs; lpcb != NULL; lpcb = lpcb->next) {
            if (lpcb->local_port == tcphdr->dest) {
                // 匹配 IP (any, exact, or specific)
                tcp_listen_input(lpcb);
                pbuf_free(p);
                return;
            }
        }
    }

    // ============================================
    // Step 5: 找到匹配的 PCB
    // ============================================
    if (pcb != NULL) {
        inseg.next = NULL;
        inseg.len = p->tot_len;
        inseg.p = p;
        inseg.tcphdr = tcphdr;

        err = tcp_process(pcb);  // ← 状态机处理

        // 处理 RESET
        if (recv_flags & TF_RESET) {
            TCP_EVENT_ERR(pcb->state, pcb->errf, pcb->callback_arg, ERR_RST);
            tcp_pcb_remove(&tcp_active_pcbs, pcb);
            tcp_free(pcb);
        }

        // 处理发送窗口更新
        if (recv_acked > 0) {
            TCP_EVENT_SENT(pcb, (u16_t)acked16, err);
        }

        // 处理接收数据
        if (recv_data != NULL) {
            TCP_EVENT_RECV(pcb, recv_data, ERR_OK, err);
            if (err != ERR_OK) {
                pcb->refused_data = recv_data;  // 应用层拒绝，保存
            }
        }

        // 处理 FIN
        if (recv_flags & TF_GOT_FIN) {
            TCP_EVENT_CLOSED(pcb, err);
        }

        tcp_output(pcb);  // ← 发送响应
    } else {
        // 无匹配 PCB，发送 RST
        if (!(TCPH_FLAGS(tcphdr) & TCP_RST)) {
            tcp_rst(NULL, ackno, seqno + tcplen, ...);
        }
    }
}
```

### 2.2 tcp_process — TCP 状态机

**文件**: `tcp_in.c:789-1044`

```c
static err_t
tcp_process(struct tcp_pcb *pcb)
{
    // ============================================
    // Step 1: RST 处理
    // ============================================
    if (flags & TCP_RST) {
        if (pcb->state == SYN_SENT) {
            // SYN-SENT: ACK 确认后才接受 RST
            if (ackno == pcb->snd_nxt) {
                acceptable = 1;
            }
        } else {
            // 其他状态: seqno 必须匹配
            if (seqno == pcb->rcv_nxt) {
                acceptable = 1;
            }
        }
        if (acceptable) {
            recv_flags |= TF_RESET;
            return ERR_RST;
        }
    }

    // ============================================
    // Step 2: 解析 TCP Options (MSS, Window Scale, etc.)
    // ============================================
    tcp_parseopt(pcb);

    // ============================================
    // Step 3: 状态处理
    // ============================================
    switch (pcb->state) {
        case SYN_SENT:
            // 收到 SYN|ACK，三次握手完成
            if ((flags & TCP_ACK) && (flags & TCP_SYN)
                && (ackno == pcb->lastack + 1)) {
                pcb->state = ESTABLISHED;
                pcb->cwnd = LWIP_TCP_CALC_INITIAL_CWND(pcb->mss);  // 初始拥塞窗口
                TCP_EVENT_CONNECTED(pcb, ERR_OK, err);  // 回调
                tcp_ack_now(pcb);
            }
            break;

        case SYN_RCVD:
            // 收到 ACK，连接建立
            if (flags & TCP_ACK) {
                if (TCP_SEQ_BETWEEN(ackno, pcb->lastack + 1, pcb->snd_nxt)) {
                    pcb->state = ESTABLISHED;
                    TCP_EVENT_ACCEPT(pcb->listener, pcb, ...);  // accept 回调
                    tcp_receive(pcb);
                }
            }
            break;

        case ESTABLISHED:
        case CLOSE_WAIT:
            tcp_receive(pcb);  // ← 数据接收
            if (recv_flags & TF_GOT_FIN) {
                tcp_ack_now(pcb);
                pcb->state = CLOSE_WAIT;
            }
            break;

        case FIN_WAIT_1:
            tcp_receive(pcb);
            if (recv_flags & TF_GOT_FIN) {
                if ((flags & TCP_ACK) && ackno == pcb->snd_nxt && pcb->unsent == NULL) {
                    pcb->state = LWIP_TCP_TIME_WAIT;
                } else {
                    pcb->state = CLOSING;
                }
            }
            break;

        // ... 其他状态
    }
    return ERR_OK;
}
```

### 2.3 tcp_receive — 数据接收

**文件**: `tcp_in.c:1142-1895`

```c
static void
tcp_receive(struct tcp_pcb *pcb)
{
    // ============================================
    // Step 1: ACK 处理
    // ============================================
    if (flags & TCP_ACK) {
        // 1.1 窗口更新
        if (TCP_SEQ_LT(pcb->snd_wl1, seqno) || ...) {
            pcb->snd_wnd = SND_WND_SCALE(pcb, tcphdr->wnd);
        }

        // 1.2 重复 ACK 检测 (Fast Retransmit)
        if (tcplen == 0 &&
            pcb->snd_wl2 + pcb->snd_wnd == right_wnd_edge &&
            pcb->rtime >= 0 &&
            pcb->lastack == ackno) {
            // 3 个重复 ACK → 快速重传
            if (pcb->dupacks >= 3) {
                tcp_rexmit_fast(pcb);
            }
        }

        // 1.3 新数据 ACK
        if (TCP_SEQ_BETWEEN(ackno, pcb->lastack + 1, pcb->snd_nxt)) {
            // 拥塞控制
            if (pcb->cwnd < pcb->ssthresh) {
                // 慢启动
                pcb->cwnd += acked;
            } else {
                // 拥塞避免
                pcb->bytes_acked += acked;
                if (pcb->bytes_acked >= pcb->cwnd) {
                    pcb->cwnd += pcb->mss;
                }
            }

            // RTT 估计
            if (pcb->rttest && TCP_SEQ_LT(pcb->rtseq, ackno)) {
                m = tcp_ticks - pcb->rttest;
                // Van Jacobson 算法
                pcb->sa += m - (pcb->sa >> 3);
                pcb->sv += m - (pcb->sv >> 2);
                pcb->rto = (pcb->sa >> 3) + pcb->sv;
            }

            // 释放已确认的 segment
            pcb->unacked = tcp_free_acked_segments(pcb, pcb->unacked, ...);
            pcb->unsent = tcp_free_acked_segments(pcb, pcb->unsent, ...);
        }
    }

    // ============================================
    // Step 2: 数据接收
    // ============================================
    if ((tcplen > 0) && (pcb->state < CLOSE_WAIT)) {
        // 2.1 序列号边界检查
        if (TCP_SEQ_BETWEEN(pcb->rcv_nxt, seqno + 1, seqno + tcplen - 1)) {
            // 需要 trim 边界
        }

        // 2.2 序列号在窗口内
        if (TCP_SEQ_BETWEEN(seqno, pcb->rcv_nxt, pcb->rcv_nxt + pcb->rcv_wnd - 1)) {
            if (pcb->rcv_nxt == seqno) {
                // 2.3 顺序到达
                pcb->rcv_nxt = seqno + tcplen;
                recv_data = inseg.p;  // 传递给应用

                // OOSEQ 处理 (Out-Of-Sequence)
                // ...

                tcp_ack(pcb);  // 发送 ACK
            } else {
                // 2.4 乱序到达 → 加入 ooseq 队列
                tcp_oos_insert_segment(...);
                tcp_send_empty_ack(pcb);
            }
        }
    }
}
```

---

## 3. TCP 状态机

### 3.1 状态转换图

```
                                    +-------+
                                    | CLOSED|
                                    +---+---+
                                        │
                                        ↓
                                    +-------+
                                    |LISTEN |
                                    +---+---+
                                        │
                    SYN ----------------+
                    ↓
                +-------+
                |SYN_SENT|
                +---+---+
                    │ SYN|ACK
                    ↓
                +-------+
         +------>|SYN_RCVD|
         |       +---+---+
         |           │  ACK
         |           ↓
         |       +-----------+
         |       | ESTABLISHED|
         |       +-----+-----+
         |             │ FIN
         |             ↓
         |       +-----------+
         |       | CLOSE_WAIT|
         |       +-----------+
         | FIN                 | close()
         ↓                     ↓
    +-----------+         +--------+
    |FIN_WAIT_1 |         |CLOSING |
    +-----+-----+         +---+----+
          | FIN|ACK           │ ACK
          ↓    ↓             ↓
    +-----------+        +--------+
    |FIN_WAIT_2 |        |LAST_ACK|
    +-----+-----+        +---+----+
          | FIN              ↓
          ↓              +--------+
    +-----------+          |TIME_WAIT|
    |TIME_WAIT  |←←←←←←←←←--------+
    +-----------+
          | 2MSL timeout
          ↓
    +-------+
    |CLOSED|
    +-------+
```

### 3.2 状态说明

| 状态 | 说明 |
|------|------|
| **CLOSED** | 无连接 |
| **LISTEN** | 监听中，等待 SYN |
| **SYN_SENT** | 已发送 SYN，等待 ACK |
| **SYN_RCVD** | 收到 SYN 并发送 SYN+ACK，等待 ACK |
| **ESTABLISHED** | 连接建立，正常数据传输 |
| **CLOSE_WAIT** | 收到 FIN，本地还在等应用关闭 |
| **FIN_WAIT_1** | 已发送 FIN，等待 ACK |
| **FIN_WAIT_2** | 收到 ACK，等待对方 FIN |
| **CLOSING** | 同时关闭中 |
| **LAST_ACK** | 最后一次 ACK |
| **TIME_WAIT** | 2MSL 等待 |

---

## 4. 拥塞控制

### 4.1 变量说明

```c
struct tcp_pcb {
    // 拥塞窗口
    tcpwnd_size_t cwnd;       // 拥塞窗口大小
    tcpwnd_size_t ssthresh;   // 慢启动阈值

    // RTT 估计
    u16_t rttest;             // RTT 采样时间
    u16_t rtseq;              // RTT 采样时的 seqno
    s16_t sa, sv;             // RTT 平滑因子 (Van Jacobson)

    // 重传
    s16_t rtime;              // 重传定时器
    u8_t nrtx;                // 重传次数
};
```

### 4.2 慢启动 vs 拥塞避免

```
cwnd < ssthresh: 慢启动
    cwnd += min(acked, MSS)

cwnd >= ssthresh: 拥塞避免
    cwnd += MSS * MSS / cwnd
```

---

## 5. 与其他模块的关系

### 5.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **ip4_input** | `tcp_input()` | IP 层分发 |

### 5.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **tcp_output** | tcp_output() | 发送响应 |
| **tcp_receive** | tcp_process() | 数据接收 |
| **tcp_parseopt** | tcp_process() | 解析 TCP options |

### 5.3 数据结构

| 结构 | 说明 |
|------|------|
| **tcp_pcb** | TCP 协议控制块 (连接) |
| **tcp_pcb_listen** | 监听中的 PCB |
| **tcp_seg** | TCP segment (存储在 unacked/ooseq) |

---

## 6. 性能特征

### 6.1 Demultiplex 复杂度

```
O(n) — 遍历 tcp_active_pcbs 链表找匹配的 PCB
```

### 6.2 瓶颈分析

```
tcp_input 瓶颈:
1. PCB 链表遍历 — O(n)，无哈希表
2. tcp_receive 中 ooseq 队列操作 — O(n)
3. RTT 计算每次都做 — 一些冗余计算
```

### 6.3 与 Linux 对比

| 特性 | SafeOS lwIP | Linux |
|------|-------------|-------|
| **PCB 查找** | 链表遍历 O(n) | 哈希表 O(1) |
| **ooseq 队列** | 链表 | 红黑树 |
| **SACK** | 支持 | 支持 |
| **RTT采样** | 每次 ACK | 仅首次 |

---

## 7. 总结

### 7.1 tcp_input 的核心作用

```
收到 TCP Segment
    │
    ├─► 解析 Header (checksum, options)
    │
    ├─► Demultiplex — 找 PCB
    │     - active_pcbs → ESTABLISHED
    │     - tw_pcbs → TIME_WAIT
    │     - listen_pcbs → LISTEN
    │
    ├─► tcp_process() — 状态机
    │     ├─► RST/SYN 处理
    │     ├─► tcp_receive() — 数据 + ACK
    │     └─► tcp_output() — 响应
    │
    └─► 回调通知 Application
```

### 7.2 关键设计

1. **全局变量状态**: tcp_input 使用大量全局变量 (seqno, ackno, flags, inseg) 存储当前 segment 状态
2. **locality optimization**: 匹配的 PCB 移动到链表头部
3. **ooseq 队列**: 乱序 segment 存储在链表队列中
4. **refused_data**: 应用层繁忙时，数据暂存在 pcb->refused_data

### 7.3 SafeOS 特供

无明显特供修改，tcp_input 保持标准 lwIP 实现。
