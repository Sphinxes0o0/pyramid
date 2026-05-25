# IP 分片与重组分析 — T-023/T-024

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: IP 分片重组、MTU 发现、ip4_frag 函数

---

## 1. 概述

当 IP 数据包大于 MTU (Maximum Transmission Unit) 时，需要进行**分片 (Fragmentation)** 和**重组 (Reassembly)**：

1. **分片 (TX)**: 大数据包被分成多个小片段
2. **重组 (RX)**: 接收端将分片重新组装

### 1.1 MTU 限制

| 网络类型 | MTU |
|---------|-----|
| Ethernet | 1500 bytes |
| PPPoE | 1492 bytes |
| VLAN | 1500 - 4 = 1496 bytes |
| Tunnel (GRE) | 1476 bytes |
| Loopback | 65535 bytes |

---

## 2. IP 分片结构

### 2.1 IP Header 中的分片字段

**文件**: `include/lwip/prot/ip4.h`

```c
struct ip_hdr {
    u8_t  _v_hl_tos;     // version (4) + header length (5) + TOS
    u16_t _len;           // total length
    u16_t _id;            // identification
    u16_t _offset;       // fragment offset + flags
    u8_t  _ttl;           // time to live
    u8_t  _proto;         // protocol
    u16_t _chksum;        // header checksum
    u32_t src;            // source IP
    u32_t dest;           // destination IP
};
```

### 2.2 分片偏移计算

```c
// IP Header 中的 _offset 字段格式:
// [13 bits: offset] [2 bits: flags] [1 bit: reserved]
//
// Flags:
//   IP_MF = 0x2000 (More Fragments)
//   IP_DF = 0x4000 (Don't Fragment)
//   IP_OFFMASK = 0x1FFF (offset mask)

// 计算偏移 (单位: 8 bytes)
offset = (iphdr->_offset & PP_NTOHS(IP_OFFMASK)) * 8

// 检查 MF (More Fragments)
is_last = (iphdr->_offset & PP_NTOHS(IP_MF)) == 0
```

---

## 3. TX: ip4_frag — 分片发送

**文件**: `core/ipv4/ip4_frag.c:741`

### 3.1 分片触发

```c
// ip4_output_if 中
#if IP_FRAG
if (netif->mtu && (p->tot_len > netif->mtu)) {
    return ip4_frag(p, netif, dest);
}
#endif
```

### 3.2 分片算法

```c
err_t ip4_frag(struct pbuf *p, struct netif *netif, const ip4_addr_t *dest)
{
    // 每个分片的数据量 (以 8 bytes 为单位)
    u16_t nfb = (netif->mtu - IP_HLEN) / 8;  // non-fragment bytes
    u16_t left = p->tot_len - IP_HLEN;       // 剩余数据

    while (left) {
        // 计算当前分片大小
        fragsize = LWIP_MIN(left, nfb * 8);

        // 分配分片 pbuf
        rambuf = pbuf_alloc(PBUF_IP, fragsize, PBUF_RAM);

        // 复制数据
        pbuf_copy_partial(p, rambuf->payload, fragsize, poff);

        // 添加 IP Header
        pbuf_add_header(rambuf, IP_HLEN);

        // 填充 IP Header
        iphdr = (struct ip_hdr *)rambuf->payload;
        SMEMCPY(iphdr, original_iphdr, IP_HLEN);

        // 设置分片偏移和 MF 标志
        if (left > fragsize) {
            // 还有更多分片
            IPH_OFFSET_SET(iphdr, (offset / 8) | IP_MF);
        } else {
            // 最后一个分片
            IPH_OFFSET_SET(iphdr, (offset / 8));
        }

        // 发送分片
        netif->output(netif, rambuf, dest);

        left -= fragsize;
        offset += fragsize;
        poff += fragsize;
    }
}
```

### 3.3 分片示例

```
原始数据: 4000 bytes (不含 IP Header)
MTU: 1500 bytes
IP Header: 20 bytes

分片 1: 1500 - 20 = 1480 bytes data, MF=1, offset=0
分片 2: 1500 - 20 = 1480 bytes data, MF=1, offset=1480
分片 3: 1060 bytes data, MF=0, offset=2960
```

---

## 4. RX: ip4_reass — 重组

**文件**: `core/ipv4/ip4_frag.c:504`

### 4.1 重组数据结构

```c
struct ip_reassdata {
    struct ip_reassdata *next;    // 链表指针
    struct pbuf *p;               // 分片 pbuf 链表
    struct ip_hdr iphdr;          // 保存原始 IP Header
    u16_t datagram_len;          // 重组后总长度
    u8_t flags;                  // IP_REASS_FLAG_LASTFRAG
    u8_t timer;                  // 超时计数器
};

static struct ip_reassdata *reassdatagrams;  // 重组队列
static u16_t ip_reass_pbufcount;            // pbuf 计数
```

### 4.2 重组流程

```c
struct pbuf *ip4_reass(struct pbuf *p)
{
    struct ip_hdr *fraghdr;
    struct ip_reassdata *ipr;
    u16_t offset, len;

    fraghdr = (struct ip_hdr *)p->payload;
    offset = IPH_OFFSET_BYTES(fraghdr);  // 偏移 (bytes)
    len = lwip_ntohs(IPH_LEN(fraghdr)) - IPH_HL_BYTES(fraghdr);

    // ============================================
    // Step 1: 查找或创建重组数据报
    // ============================================
    for (ipr = reassdatagrams; ipr != NULL; ipr = ipr->next) {
        // 检查是否匹配 (src, dst, protocol, ID)
        if (IP_ADDRESSES_ID_PROTO_MATCH(&ipr->iphdr, fraghdr)) {
            break;  // 找到
        }
    }

    if (ipr == NULL) {
        // 创建新的重组数据报
        ipr = ip_reass_enqueue_new_datagram(fraghdr);
    }

    // ============================================
    // Step 2: 链入分片
    // ============================================
    is_last = (IPH_OFFSET(fraghdr) & IP_MF) == 0;
    valid = ip_reass_chain_frag_into_datagram_and_validate(ipr, p, is_last);

    // ============================================
    // Step 3: 检查是否完成
    // ============================================
    if (valid == IP_REASS_VALIDATE_TELEGRAM_FINISHED) {
        // 所有分片已收到
        p = ip_reass_remove(ipr);  // 提取完整数据报
        return p;
    }

    return NULL;  // 继续等待更多分片
}
```

---

## 5. 重组队列管理

### 5.1 ip_reass_enqueue_new_datagram

```c
static struct ip_reassdata *ip_reass_enqueue_new_datagram(struct ip_hdr *fraghdr)
{
    struct ip_reassdata *ipr;

    // 分配内存
    ipr = (struct ip_reassdata *)memp_malloc(MEMP_REASSDATA);
    if (ipr == NULL) {
        return NULL;
    }

    // 初始化
    memset(ipr, 0, sizeof(struct ip_reassdata));
    SMEMCPY(&ipr->iphdr, fraghdr, IP_HLEN);  // 保存 IP Header
    ipr->timer = IP_REASS_MAXAGE;  // 超时时间

    // 添加到队列
    ipr->next = reassdatagrams;
    reassdatagrams = ipr;

    return ipr;
}
```

### 5.2 ip_reass_chain_frag_into_datagram_and_validate

```c
static int ip_reass_chain_frag_into_datagram_and_validate(
    struct ip_reassdata *ipr, struct pbuf *new_p, int is_last)
{
    // 按偏移顺序插入分片
    // 检查是否有空洞或重叠
    // 更新 datagram_len
}
```

---

## 6. 超时管理

### 6.1 重组超时

**文件**: `core/ipv4/ip4_frag.c:128`

```c
void ip_reass_tmr(void)
{
    struct ip_reassdata *r, *prev = NULL;

    r = reassdatagrams;
    while (r != NULL) {
        if (r->timer > 0) {
            r->timer--;
            prev = r;
            r = r->next;
        } else {
            // 超时，删除重组数据报
            struct ip_reassdata *tmp = r;
            r = r->next;
            ip_reass_free_complete_datagram(tmp, prev);
        }
    }
}
```

### 6.2 超时参数

```c
#define IP_REASS_MAXAGE 30  // 重组超时 (秒)
#define IP_TMR_INTERVAL 1000  // 定时器间隔 (ms)
```

---

## 7. 分片限制

### 7.1 IP_REASS_MAX_PBUFS

```c
// 重组队列中最大 pbuf 数量
#define IP_REASS_MAX_PBUFS 100
```

### 7.2 内存压力处理

```c
if ((ip_reass_pbufcount + clen) > IP_REASS_MAX_PBUFS) {
#if IP_REASS_FREE_OLDEST
    // 删除最旧的重组数据报
    ip_reass_remove_oldest_datagram();
#endif
    // 如果还是太多，丢弃当前分片
}
```

---

## 8. 与 Linux 的对比

### 8.1 Linux IP Fragmentation

Linux 维护一个 **fragment cache**：

```bash
# 查看 fragment cache
cat /proc/net/sockstat

# Fragment reassembly timeout
sysctl -w net.ipv4.ipfrag_time=30
```

### 8.2 lwIP vs Linux

| 特性 | lwIP | Linux |
|------|------|-------|
| **重组位置** | 协议栈内 | 内核网络命名空间 |
| **分片位置** | 协议栈内 | 可配置 (ip_forward) |
| **超时** | 30 秒 | 30 秒 |
| **内存限制** | IP_REASS_MAX_PBUFS | sysctl ipfrag* |
| **重叠处理** | 丢弃重叠分片 | 支持重叠 |

---

## 9. 总结

### 9.1 分片 (TX)

```
ip4_output_if()
    │
    └─► if (p->tot_len > netif->mtu)
          └─► ip4_frag()
                │
                ├─► 计算 nfb = (mtu - IP_HLEN) / 8
                ├─► while (left > 0)
                │     ├─► 分配分片 pbuf
                │     ├─► 复制数据
                │     ├─► 设置 IP Header (offset, MF)
                │     └─► netif->output()
                └─► return
```

### 9.2 重组 (RX)

```
ip4_input()
    │
    └─► if (IP_MF set || offset > 0)
          └─► ip4_reass()
                │
                ├─► 查找/创建重组数据报
                │     └─► ip_reass_enqueue_new_datagram()
                ├─► 链入分片
                │     └─► ip_reass_chain_frag_into_datagram_and_validate()
                ├─► 检查是否完成
                │     └─► 如果 valid == TELEGRAM_FINISHED
                │           └─► 返回完整 pbuf
                └─► return NULL (等待更多分片)
```

### 9.3 关键设计

1. **偏移单位**: IP 偏移以 8 bytes 为单位
2. **MF 标志**: MF=1 表示还有更多分片，MF=0 表示最后一个分片
3. **ID 匹配**: 同一数据报的所有分片有相同的 (src, dst, protocol, ID)
4. **超时删除**: 重组超时删除未完成的分片
5. **内存限制**: IP_REASS_MAX_PBUFS 限制重组队列大小
