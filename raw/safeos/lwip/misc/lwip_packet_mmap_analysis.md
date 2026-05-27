# packet_mmap 实现分析 — T-091

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: AF-PACKET mmap 实现、ring buffer、零拷贝机制

---

## 1. 概述

SafeOS NSv 通过 `packet_mmap` 实现 Linux 兼容的 AF-PACKET mmap 接口，允许应用通过共享内存直接访问网络数据包，绕过内核拷贝。

**与 Linux 的主要差异**：
- 使用自定义 DSPACE 机制替代 Linux 的 `mlock` 共享内存
- 基于 seL4 IPC 进行控制平面通信
- TPACKET_V1 实现，仅支持 RX Ring

---

## 2. 核心组件

### 2.1 文件结构

| 文件 | 位置 | 职责 |
|------|------|------|
| `packet_mmap.c` | `servers/net/src/` | Ring 设置、tpacket_recv 回调、event callback |
| `packet_mmap.h` | `servers/net/include/nsv/` | 数据结构、常量定义 |
| `ringbuffer.h/c` | `libs/os_libs/libcore/src/` | 无锁 ring buffer 实现 |
| `dspace.c` | `libs/os_libs/libcore/src/` | DSPACE 系统调用实现 |

### 2.2 数据结构

**packet_mmap_info** — 存储在共享内存中：

```c
struct packet_mmap_info {
    pid_t               pid;                // 应用 PID
    int                 socket;            // AF_PACKET socket
    unsigned int        tp_frame_size;      // 每帧大小 (默认 2048)
    uint16_t            rx_buf_offset;      // ringbuf 偏移
    volatile uint16_t   loop_recevent;      // 本次 process_packet 接收计数
    volatile uint16_t   recved_packets;     // FIFO 中总包数
    void               *netconn;           // lwIP netconn 指针
};
```

**packet_mmap_dspace_node** — NSv 侧管理节点：

```c
struct packet_mmap_dspace_node {
    pid_t                         pid;              // 应用 PID
    int                           socket;           // socket 描述符
    struct dspace_info            dspace_info;      // dspace 映射信息
    struct slist                  list;             // 链表节点
};
```

### 2.3 常量定义

```c
#define DEFAULT_TP_FRAME_SIZE    2048    // 每帧大小
#define DEFAULT_TP_FRAME_NR       1024    // 帧总数
#define DEFAULT_TP_BLOCK_SIZE     4096    // block 大小
#define DEFAULT_DSPACE_SIZE       0x400000 // 4MB DSPACE 大小
```

---

## 3. DSPACE 内存布局

```
DSPACE (4MB)
├─ packet_mmap_info (32 bytes)     ← 元数据
├─ ringbuf (管理读写索引)           ← ring buffer 头部
└─ TPACKET 帧循环队列              ← 数据区 (2048 × 1024 bytes)
```

### 3.1 TPACKET 帧格式

```
┌────────────┬─────────────────────────────────┬────────────────────────────┐
│ tpacket_hdr│      Ethernet Frame             │         Padding            │
│  (24 B)    │     (tp_len bytes)              │ (2048-24-tp_len bytes)    │
└────────────┴─────────────────────────────────┴────────────────────────────┘
```

**tpacket_hdr 结构**：

```c
struct tpacket_hdr {
    volatile uint64_t    tp_timestamp;   // 时间戳
    uint32_t             tp_snaplen;     // 捕获长度
    uint32_t             tp_len;         // 包长度
    uint32_t             tp_net;         // 帧头长度 (sizeof(tpacket_hdr))
    volatile uint16_t    tp_vlan_tci;    // VLAN TCI
    volatile uint16_t    tp_vlan_tp;     // VLAN TPID
    volatile uint32_t    tp_status;      // 帧状态 (TP_STATUS_USER=1)
};
```

---

## 4. 关键流程

### 4.1 packet_mmap_set_ring — Ring 初始化

**文件**: `packet_mmap.c:265-330`

```c
int packet_mmap_set_ring(int socket, seL4_Word badge,
                        const void *packet_mmap_param, unsigned int len,
                        struct packet_mmap_dspace_node *packet_mmap_list)
{
    // 1. 验证参数长度
    if (len != sizeof(struct packet_mmap_param)) {
        return -1;
    }

    // 2. 获取 socket 对应的 lwip_sock
    struct lwip_sock *sock = get_socket(socket);

    // 3. 添加 DSPACE 节点到链表
    struct packet_mmap_dspace_node *node;
    node = packet_mmap_add_node(app_ds, pid, socket, packet_mmap_list);

    // 4. 将应用 DSPACE 映射到 NSv 地址空间
    err = sys_dspace_map(app_ds, &(dspace_info->va), &(dspace_info->size), ...);

    // 5. 初始化 packet_mmap_info
    struct packet_mmap_info *packet_info = (struct packet_mmap_info *)(dspace_info->va);
    packet_info->tp_frame_size = req.tp_frame_size;
    packet_info->rx_buf_offset = RX_BUF_RING_OFFSET;
    packet_info->netconn = sock->conn;

    // 6. 初始化 ringbuf
    struct ringbuf *rbuf = packet_mmap_rbuf(packet_info);
    init_rx_buf_ring(rbuf_size, rbuf);

    // 7. 注册回调
    sock->conn->pcb.raw_afpacket->recv_arg = packet_info;
    sock->conn->pcb.raw_afpacket->recv = tpacket_recv;
    sock->conn->callback = packet_mmap_event_callback;
    sock->packet_info = packet_info;

    return 0;
}
```

### 4.2 tpacket_recv — 收包回调

**文件**: `packet_mmap.c:122-155`

当 lwIP 收到数据包时，`raw_afpacket_input` 会调用此回调：

```c
static uint8_t tpacket_recv(void *arg, struct raw_pcb *pcb,
                           struct pbuf *p_buf, const ip_addr_t *addr)
{
    struct packet_mmap_info *packet_node = (struct packet_mmap_info *)arg;
    struct ringbuf *rx_buf_ring = packet_mmap_rbuf(packet_node);

    // 检查 ring buffer 是否有足够空间
    int write_avail = rb_write_avail(rx_buf_ring);
    if (write_avail > DEFAULT_TP_FRAME_SIZE) {
        // 写入 TPACKET 帧
        bool res = rb_write_tpacket(rx_buf_ring, p_buf, packet_node->tp_frame_size);
        if (res && conn) {
            // 触发 select/poll 唤醒
            API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0);
        }
    } else {
        // ring buffer 满，丢弃包
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.rawafpacket_mbox_errs);
    }
    return 0;
}
```

### 4.3 rb_write_tpacket — Ring Buffer 写入

**文件**: `packet_mmap.c:104-119`

```c
static inline bool rb_write_tpacket(struct ringbuf *rbuf, struct pbuf *p_buf,
                                   unsigned int tp_frame_size)
{
    // 计算 padding
    int padding = tp_frame_size - sizeof(struct tpacket_hdr) - p_buf->tot_len;

    // 填充 TPACKET header
    struct tpacket_hdr packet_hdr = {0};
    packet_hdr.tp_status = TP_STATUS_USER;    // 标记为用户可读
    packet_hdr.tp_len = p_buf->tot_len;       // 包长度
    packet_hdr.tp_net = sizeof(struct tpacket_hdr);  // 网络头偏移

    // 写入 header + payload + padding
    rb_write(rbuf, &packet_hdr, sizeof(struct tpacket_hdr));
    rb_write(rbuf, p_buf->payload, p_buf->tot_len);
    rb_write(rbuf, rb_buf_padding, padding);

    return true;
}
```

### 4.4 packet_mmap_event_callback — 事件回调

**文件**: `packet_mmap.c:172-263`

```c
void packet_mmap_event_callback(struct netconn *conn, enum netconn_evt evt, u16_t len)
{
    struct lwip_sock *sock = get_socket(s);
    SYS_ARCH_PROTECT(lev);

    switch (evt) {
        case NETCONN_EVT_RCVPLUS:
            sock->rcvevent++;
            if (sock->packet_info != NULL) {
                struct packet_mmap_info *info = sock->packet_info;
                info->recved_packets++;  // 更新接收计数
            }
            break;
        case NETCONN_EVT_RCVMINUS:
            sock->rcvevent--;
            break;
    }

    if (sock->select_waiting && check_waiters) {
        // 唤醒等待的 select/poll
        select_check_waiters(s, has_recvevent, has_sendevent, has_errevent);
    }

    SYS_ARCH_UNPROTECT(lev);
}
```

---

## 5. Ring Buffer 实现

### 5.1 ringbuf 结构

**文件**: `libs/os_libs/libcore/src/ringbuffer.h`

```c
struct ringbuf {
    uint32_t    size;       // 总大小
    uint32_t    head;       // 读指针
    uint32_t    tail;       // 写指针
    uint8_t     data[0];    // 数据区
};
```

### 5.2 无锁设计原理

ringbuf 是单生产者/单消费者无锁环形缓冲区：

| 操作 | 执行者 | 同步机制 |
|------|--------|----------|
| **写入** | NSv (tpacket_recv) | 仅生产者写入 head |
| **读取** | App (rb_read) | 仅消费者读取 tail |

```c
// 写入 (packet_mmap.c 侧)
rb_write(rbuf, data, len) {
    // head 指针原子写入
    memcpy(rbuf->data + rbuf->head, data, len);
    rbuf->head = (rbuf->head + len) % rbuf->size;
}

// 读取 (App 侧)
rb_read(rbuf, data, len) {
    // tail 指针原子读取
    memcpy(data, rbuf->data + rbuf->tail, len);
    rbuf->tail = (rbuf->tail + len) % rbuf->size;
}
```

### 5.3 可用空间计算

```c
rb_write_avail(rbuf) {
    if (rbuf->head >= rbuf->tail) {
        return rbuf->size - rbuf->head + rbuf->tail;
    } else {
        return rbuf->tail - rbuf->head;
    }
}
```

---

## 6. 收包完整路径

```
NIC DMA
    │
    ▼
used_rx_buf_ring → nic_rx_thread → rx_callback
    │
    ▼
vnet_if.input = ethernet_input
    │
    ├─► LWIP_ARP_FILTER_NETIF (VLAN 分发)
    │
    ▼
raw_afpacket_input() [raw.c]
    │
    ├─► 检查 socket 绑定的 protocol/type
    │
    ▼
tpacket_recv() [packet_mmap.c:122]
    │
    ├─► rb_write_avail() 检查空间
    │
    ├─► rb_write_tpacket() 写入 ring buffer
    │     ├─ tpacket_hdr (24B)
    │     ├─ pbuf->payload (包数据)
    │     └─ padding
    │
    ▼
API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)
    │
    ▼
packet_mmap_event_callback()
    │
    ├─► sock->rcvevent++
    ├─► packet_info->recved_packets++
    │
    ▼
select_check_waiters() → 唤醒 App poll/select
    │
    ▼
App: poll() 返回 POLLIN
    │
    ▼
rb_read() 从 ring buffer 读取数据包
```

---

## 7. 与 Linux 对比

| 特性 | SafeOS NSv | Linux |
|------|-------------|-------|
| **共享内存机制** | DSPACE + seL4 IPC | mmap + 页表 |
| **Ring Buffer 类型** | 自定义 ringbuf | 泛化的 ringbuffer |
| **TPACKET 版本** | TPACKET_V1 | TPACKET_V1/V2/V3 |
| **TX Ring** | 不支持 | 支持 |
| **VLAN tag** | tp_vlan_tci/tp_vlan_tp | skb->vlan_tci |
| **时间戳** | 固定 0 | 硬件/软件时间戳 |
| **零拷贝** | 部分 (pbuf → ring) | 完全 (skb → ring) |

### 7.1 架构差异

**SafeOS NSv**：
```
App                              NSv
 │                                 │
 ├─ sys_dspace_create(4MB) ──────┼─ PSv 创建 dspace_t
 │                                 │
 ├─ grant_shm_to_net() ────────────┼─ seL4 IPC 授权
 │                                 │
 ├─ mmap() 映射同一 DSPACE ─────────┼─ (通过 seL4 共享页)
 │                                 │
 ▼                                 ▼
ring buffer                       ring buffer
(App 读取)                        (NSv 写入)
```

**Linux**：
```
App                    Kernel               NIC Driver
 │                       │                      │
 ├─ mmap() ──────────────┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
 │                       │                      │
 │                       ├─ skb → ring buffer   │
 │                       │                      │
 ▼                       ▼                      ▼
ring buffer             ring buffer           DMA
(App 读取)              (kernel 写入)        (NIC)
```

---

## 8. 性能考虑

### 8.1 零拷贝分析

| 阶段 | SafeOS NSv | Linux |
|------|-------------|-------|
| NIC → pbuf | DMA | DMA |
| pbuf → ring buffer | 1 copy | 0 copy (skb direct) |
| ring buffer → App | 0 copy (mmap) | 0 copy (mmap) |
| **总拷贝次数** | **1** | **0** |

SafeOS NSv 在 `pbuf → ring buffer` 阶段需要一次拷贝，因为 pbuf 可能分散在多个内存块中。

### 8.2 瓶颈分析

1. **tcpip_thread 锁**：LWFW ingress_filter 在 `LOCK_TCPIP_CORE()` 下执行，tpacket_recv 作为 raw_pcb 回调也在锁内
2. **ring buffer 竞争**：多 socket 场景下，ring buffer 写入可能成为瓶颈
3. **seL4 IPC 开销**：DSPACE 映射需要 IPC 调用

---

## 9. 资源管理

### 9.1 packet_mmap_free_resources — 释放资源

```c
int packet_mmap_free_resources(int socket, pid_t pid,
                              struct packet_mmap_dspace_node *packet_mmap_list)
{
    // 1. 查找节点
    struct packet_mmap_dspace_node *node;
    node = packet_mmap_get_node(pid, socket, packet_mmap_list);

    // 2. 解除 DSPACE 映射
    sys_dspace_unmap(node->dspace_info.ds, node->dspace_info.va);

    // 3. 从链表移除
    packet_mmap_remove_node(pid, socket, packet_mmap_list);

    return 0;
}
```

### 9.2 packet_mmap_free_for_poll — poll 后清理

```c
void packet_mmap_free_for_poll(int nfds, struct sel4_pollfd *spfd)
{
    for (int i = 0; i < nfds; i++, spfd++) {
        if (sock->packet_info != NULL) {
            // 清理本次 poll 接收的包计数
            sock->rcvevent -= packet_info->loop_recevent;
            packet_info->recved_packets -= packet_info->loop_recevent;
            packet_info->loop_recevent = 0;
        }
    }
}
```

---

## 10. 总结

### 10.1 关键设计

1. **DSPACE 机制**：使用 seL4 DSPACE 实现应用与 NSv 的共享内存
2. **单生产者/消费者**：ringbuf 无锁设计，仅需考虑边界条件
3. **TPACKET_V1**：简化实现，仅支持 RX Ring
4. **回调链**：raw_pcb.recv → tpacket_recv → API_EVENT → select 唤醒

### 10.2 数据流

```
NIC DMA → pbuf → raw_afpacket_input → tpacket_recv
    → rb_write_tpacket → ring buffer (mmap)
    → App rb_read → 用户空间
```

### 10.3 待优化项

| 问题 | 影响 | 优化方向 |
|------|------|----------|
| TPACKET_V1 only | 无超时回收机制 | 实现 TPACKET_V3 |
| 无 TX Ring | 无法做发送环 | 按需实现 |
| 时间戳为 0 | 无法做时间分析 | 使用实际时间戳 |
| pbuf → ring copy | 性能开销 | 考虑零拷贝方案 |
