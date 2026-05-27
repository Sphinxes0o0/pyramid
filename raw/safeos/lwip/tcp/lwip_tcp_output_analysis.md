# tcp_output 分析 — T-031

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: tcp_output 函数：拥塞控制、慢启动、快重传、nagle 算法、segment 发送

---

## 1. 概述

`tcp_output()` 是 lwIP 的 **TCP 发送主函数**，负责：
1. 计算可用发送窗口 (snd_wnd, cwnd)
2. 应用 Nagle 算法
3. 发送窗口内的 segment
4. 管理重传定时器

### 1.1 调用链

```
tcp_write() — 应用层数据写入发送缓冲区
    │
    ▼
tcp_output() — 发送数据
    │
    ├─► tcp_route() — 路由查找
    ├─► Nagle 算法检查
    ├─► tcp_output_segment() — 发送单个 segment
    │     └─► ip_output_if() → ip4_output_if()
    │           └─► LWFW egress_filter
    │
    └─► 管理 unacked 队列
```

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/core/tcp_out.c:1248`

### 2.1 tcp_output — 主函数

```c
err_t
tcp_output(struct tcp_pcb *pcb)
{
    struct tcp_seg *seg, *useg;
    u32_t wnd, snd_nxt;
    err_t err;
    struct netif *netif;

    // ============================================
    // Step 1: 避免递归调用
    // ============================================
    if (tcp_input_pcb == pcb) {
        return ERR_OK;  // 如果是被 input 处理调用，不输出
    }

    // ============================================
    // Step 2: 计算发送窗口
    // ============================================
    wnd = LWIP_MIN(pcb->snd_wnd, pcb->cwnd);  // 拥塞窗口 vs 接收窗口

    seg = pcb->unsent;  // 未发送的 segment 队列

    // ============================================
    // Step 3: 无数据可发送
    // ============================================
    if (seg == NULL) {
        if (pcb->flags & TF_ACK_NOW) {
            return tcp_send_empty_ack(pcb);  // 发送纯 ACK
        }
        goto output_done;
    }

    // ============================================
    // Step 4: 路由查找
    // ============================================
    netif = tcp_route(pcb, &pcb->local_ip, &pcb->remote_ip);
    if (netif == NULL) {
        return ERR_RTE;  // 无路由
    }

    // ============================================
    // Step 5: 检查 segment 是否适合窗口
    // ============================================
    if (lwip_ntohl(seg->tcphdr->seqno) - pcb->lastack + seg->len > wnd) {
        // segment 不在窗口内
        if (wnd == pcb->snd_wnd && pcb->unacked == NULL && pcb->persist_backoff == 0) {
            pcb->persist_backoff = 1;  // 启动 persist timer
        }
        if (pcb->flags & TF_ACK_NOW) {
            return tcp_send_empty_ack(pcb);
        }
        goto output_done;
    }

    // ============================================
    // Step 6: 发送循环
    // ============================================
    while (seg != NULL &&
           lwip_ntohl(seg->tcphdr->seqno) - pcb->lastack + seg->len <= wnd) {

        // 6.1 Nagle 算法检查
        if ((tcp_do_output_nagle(pcb) == 0) &&
            ((pcb->flags & (TF_NAGLEMEMERR | TF_FIN)) == 0)) {
            break;  // Nagle 阻止发送
        }

        // 6.2 设置 ACK 标志
        if (pcb->state != SYN_SENT) {
            TCPH_SET_FLAG(seg->tcphdr, TCP_ACK);
        }

        // 6.3 发送 segment
        err = tcp_output_segment(seg, pcb, netif);
        if (err != ERR_OK) {
            tcp_set_flags(pcb, TF_NAGLEMEMERR);
            return err;
        }

        // 6.4 移动到 unacked 队列
        pcb->unsent = seg->next;
        snd_nxt = lwip_ntohl(seg->tcphdr->seqno) + TCP_TCPLEN(seg);
        if (TCP_SEQ_LT(pcb->snd_nxt, snd_nxt)) {
            pcb->snd_nxt = snd_nxt;
        }

        // 6.5 加入 unacked 队列
        if (TCP_TCPLEN(seg) > 0) {
            seg->next = NULL;
            if (pcb->unacked == NULL) {
                pcb->unacked = seg;
            } else {
                // 按序列号排序插入
                ...
            }
        } else {
            tcp_seg_free(seg);  // 空 segment 释放
        }

        seg = pcb->unsent;
    }

output_done:
    tcp_clear_flags(pcb, TF_NAGLEMEMERR);
    return ERR_OK;
}
```

### 2.2 tcp_output_segment — 发送单个 segment

**文件**: `tcp_out.c:1473-1661`

```c
static err_t
tcp_output_segment(struct tcp_seg *seg, struct tcp_pcb *pcb, struct netif *netif)
{
    // ============================================
    // Step 1: 填充 ACK 编号
    // ============================================
    seg->tcphdr->ackno = lwip_htonl(pcb->rcv_nxt);
    seg->tcphdr->wnd = lwip_htons(TCPWND_MIN16(pcb->rcv_ann_wnd));

    // ============================================
    // Step 2: 添加 TCP Options
    // ============================================
    // MSS, Timestamp, SACK, Window Scale
    ...

    // ============================================
    // Step 3: 启动 RTT 计时器
    // ============================================
    if (pcb->rtime < 0) {
        pcb->rtime = 0;  // 启动重传定时器
    }
    if (pcb->rttest == 0) {
        pcb->rttest = tcp_ticks;
        pcb->rtseq = lwip_ntohl(seg->tcphdr->seqno);  // RTT 采样
    }

    // ============================================
    // Step 4: 计算 TCP Checksum
    // ============================================
    #if CHECKSUM_GEN_TCP
    IF__NETIF_CHECKSUM_ENABLED(netif, NETIF_CHECKSUM_GEN_TCP) {
        acc = ip_chksum_pseudo_partial(seg->p, IP_PROTO_TCP, ...);
        seg->tcphdr->chksum = (u16_t)~FOLD_U32T(acc);
    }
    #endif

    // ============================================
    // Step 5: 发送到 IP 层
    // ============================================
    NETIF_SET_HINTS(netif, &(pcb->netif_hints));
    err = ip_output_if(seg->p, &pcb->local_ip, &pcb->remote_ip,
                       pcb->ttl, pcb->tos, IP_PROTO_TCP, netif);
    NETIF_RESET_HINTS(netif);

    return err;
}
```

---

## 3. 拥塞控制

### 3.1 发送窗口计算

```c
wnd = LWIP_MIN(pcb->snd_wnd, pcb->cwnd);
//       接收窗口              拥塞窗口
```

### 3.2 慢启动 vs 拥塞避免

```
cwnd < ssthresh: 慢启动
    cwnd += min(acked, MSS)  // 每个 ACK cwnd 增加一个 MSS

cwnd >= ssthresh: 拥塞避免
    cwnd += MSS * MSS / cwnd  // 每个 ACK cwnd 增加一小部分
```

### 3.3 初始拥塞窗口 (IW)

```c
// RFC 2581
cwnd = LWIP_MIN(4 * MSS, max(2 * MSS, 4380))
```

---

## 4. Nagle 算法

### 4.1 算法描述

Nagle 算法**减少小包数量**：

```
如果:
  1. 有未确认的数据 (unacked 不为空)
  2. 有未发送的小数据 (< MSS)

则: 等待，直到 ACK 到达
```

### 4.2 tcp_do_output_nagle

```c
// tcp_out.c 中的 nagle 检查
if ((tcp_do_output_nagle(pcb) == 0) &&
    ((pcb->flags & (TF_NAGLEMEMERR | TF_FIN)) == 0)) {
    break;  // 阻止发送
}

// nagle 条件: 小数据 + 有未确认数据
```

---

## 5. 重传机制

### 5.1 重传定时器

```c
// 启动
if (pcb->rtime < 0) {
    pcb->rtime = 0;
}

// RTO 计算 (Van Jacobson 算法)
rto = (sa >> 3) + sv;

// sa: 平滑 RTT 估计
// sv: RTT 偏差
```

### 5.2 快速重传

当收到 **3 个重复 ACK** (dupacks >= 3)：

```c
// tcp_receive() 中
if (pcb->dupacks >= 3) {
    tcp_rexmit_fast(pcb);  // 快速重传丢失的 segment
}
```

---

## 6. 与其他模块的关系

### 6.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **tcp_input** | `tcp_output()` | 处理完输入后发送 ACK |
| **tcp_write** | `tcp_output()` | 写入数据后触发发送 |
| **应用** | 直接/间接 | 通过 socket API |

### 6.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **ip_output_if** | `tcp_output_segment()` | 发送到 IP 层 |
| **LWFW** | egress_filter | 在 ip_output_if 中调用 |

---

## 7. 性能特征

### 7.1 复杂度

```
tcp_output: O(n) — 遍历 unsent 队列
tcp_output_segment: O(1) — 发送单个 segment
```

### 7.2 瓶颈

```
1. Nagle 算法: 小包延迟发送
2. cwnd 限制: 拥塞窗口小则发送受限
3. 重传开销: 丢包导致重传
```

---

## 8. 总结

### 8.1 tcp_output 的核心作用

```
发送 TCP 数据
    │
    ├─► 计算发送窗口 min(snd_wnd, cwnd)
    │
    ├─► 路由查找 (tcp_route)
    │
    ├─► Nagle 算法检查
    │
    ├─► 循环发送 segment
    │     ├─► 设置 ACK 标志
    │     ├─► tcp_output_segment()
    │     │     └─► ip_output_if() → LWFW egress_filter
    │     └─► 移动到 unacked 队列
    │
    └─► 返回 ERR_OK
```

### 8.2 关键设计

1. **窗口限制**: min(snd_wnd, cwnd) 防止发送过多
2. **Nagle 算法**: 减少小包，提高网络效率
3. **RTT 采样**: 在第一个 segment 发送时记录
4. **unacked 管理**: 发送后移到 unacked，等待 ACK

### 8.3 SafeOS 特供

无明显特供修改，tcp_output 保持标准 lwIP 实现。
