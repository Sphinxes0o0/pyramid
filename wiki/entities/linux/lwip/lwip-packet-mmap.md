---
type: entity
tags: [linux, lwip, network, packet-mmap, af-packet, ring-buffer, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# packet_mmap — AF-PACKET MMAP Implementation

## 定义

SafeOS NSv 通过 `packet_mmap` 实现 Linux 兼容的 AF-PACKET mmap 接口，允许应用通过共享内存直接访问网络数据包，绕过内核拷贝。核心是 DSPACE 机制 + TPACKET_V1。

## TPACKET 帧格式

```
┌────────────┬─────────────────────────────────┬────────────────────────┐
│ tpacket_hdr│      Ethernet Frame             │        Padding        │
│  (24 B)   │     (tp_len bytes)             │ (2048-24-tp_len B)   │
└────────────┴─────────────────────────────────┴────────────────────────┘
```

### tpacket_hdr 结构
```c
struct tpacket_hdr {
    volatile uint64_t tp_timestamp;   // 时间戳
    uint32_t tp_snaplen;             // 捕获长度
    uint32_t tp_len;                // 包长度
    uint32_t tp_net;                // 帧头长度
    volatile uint16_t tp_vlan_tci;  // VLAN TCI
    volatile uint16_t tp_vlan_tp;   // VLAN TPID
    volatile uint32_t tp_status;     // 帧状态 (TP_STATUS_USER=1)
};
```

## Ring Buffer 写入 (tpacket_recv)

```c
static uint8_t tpacket_recv(void *arg, struct raw_pcb *pcb,
                           struct pbuf *p_buf, const ip_addr_t *addr)
{
    struct packet_mmap_info *packet_node = (struct packet_mmap_info *)arg;
    struct ringbuf *rx_buf_ring = packet_mmap_rbuf(packet_node);

    int write_avail = rb_write_avail(rx_buf_ring);
    if (write_avail > DEFAULT_TP_FRAME_SIZE) {
        rb_write_tpacket(rx_buf_ring, p_buf, packet_node->tp_frame_size);
        if (conn) {
            API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0);  // 唤醒 select/poll
        }
    }
    return 0;
}
```

## 收包完整路径

```
NIC DMA
    │
    ▼
used_rx_buf_ring → nic_rx_thread → rx_callback
    │
    ▼
vnet_if.input = ethernet_input
    │
    ▼
raw_afpacket_input() [raw.c]
    │
    ▼
tpacket_recv() [packet_mmap.c]
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
select_check_waiters() → 唤醒 App poll/select
    │
    ▼
rb_read() 从 ring buffer 读取数据包
```

## 零拷贝分析

| 阶段 | SafeOS NSv | Linux |
|------|-------------|-------|
| NIC → pbuf | DMA | DMA |
| pbuf → ring buffer | 1 copy | 0 copy (skb direct) |
| ring buffer → App | 0 copy (mmap) | 0 copy (mmap) |
| **总拷贝次数** | **1** | **0** |

SafeOS 在 `pbuf → ring buffer` 阶段需要一次拷贝，因为 pbuf 可能分散在多个内存块中。

## 与 Linux 对比

| 特性 | SafeOS NSv | Linux |
|------|-------------|-------|
| 共享内存机制 | DSPACE + seL4 IPC | mmap + 页表 |
| TPACKET 版本 | TPACKET_V1 | TPACKET_V1/V2/V3 |
| TX Ring | 不支持 | 支持 |
| VLAN tag | tp_vlan_tci/tp_vlan_tp | skb->vlan_tci |
| 零拷贝 | 部分 (pbuf → ring) | 完全 (skb → ring) |

## 相关概念

- [[entities/linux/lwip/lwip-raw-socket]] — RAW socket / AF-PACKET 绑定
- [[entities/linux/lwip/lwip-nsv-event-loop]] — NSv 事件循环
- [[entities/linux/lwip/lwip-elem-ring]] — 无锁 ring buffer

## 来源详情

- [[sources/safeos-lwip-extensions]]
