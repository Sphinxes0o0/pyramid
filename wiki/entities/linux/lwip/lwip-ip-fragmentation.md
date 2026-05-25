---
type: entity
tags: [linux, lwip, network, ip, fragmentation]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP IP Fragmentation Analysis

## 定义

当 IP 数据包大于 MTU 时，需要进行**分片 (Fragmentation)** 和**重组 (Reassembly)**。

## MTU 限制

| 网络类型 | MTU |
|---------|-----|
| Ethernet | 1500 bytes |
| PPPoE | 1492 bytes |
| VLAN | 1500 - 4 = 1496 bytes |
| Tunnel (GRE) | 1476 bytes |
| Loopback | 65535 bytes |

## 分片结构

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
```

## TX: ip4_frag — 分片发送

```c
err_t ip4_frag(struct pbuf *p, struct netif *netif, const ip4_addr_t *dest) {
    u16_t nfb = (netif->mtu - IP_HLEN) / 8;  // non-fragment bytes
    u16_t left = p->tot_len - IP_HLEN;

    while (left) {
        fragsize = LWIP_MIN(left, nfb * 8);
        rambuf = pbuf_alloc(PBUF_IP, fragsize, PBUF_RAM);
        pbuf_copy_partial(p, rambuf->payload, fragsize, poff);
        pbuf_add_header(rambuf, IP_HLEN);
        // 设置分片偏移和 MF 标志
        if (left > fragsize) {
            IPH_OFFSET_SET(iphdr, (offset / 8) | IP_MF);
        } else {
            IPH_OFFSET_SET(iphdr, (offset / 8));
        }
        netif->output(netif, rambuf, dest);
        left -= fragsize;
        offset += fragsize;
    }
}
```

## RX: ip4_reass — 重组

```c
struct pbuf *ip4_reass(struct pbuf *p) {
    // 查找或创建重组数据报
    for (ipr = reassdatagrams; ipr != NULL; ipr = ipr->next) {
        if (IP_ADDRESSES_ID_PROTO_MATCH(&ipr->iphdr, fraghdr)) {
            break;  // 找到
        }
    }

    if (ipr == NULL) {
        ipr = ip_reass_enqueue_new_datagram(fraghdr);
    }

    // 链入分片
    is_last = (IPH_OFFSET(fraghdr) & IP_MF) == 0;
    valid = ip_reass_chain_frag_into_datagram_and_validate(ipr, p, is_last);

    // 检查是否完成
    if (valid == IP_REASS_VALIDATE_TELEGRAM_FINISHED) {
        return ip_reass_remove(ipr);  // 返回完整数据报
    }
    return NULL;  // 继续等待更多分片
}
```

## 重组超时

```c
void ip_reass_tmr(void) {
    for (r = reassdatagrams; r != NULL; r = r->next) {
        if (r->timer > 0) {
            r->timer--;
        } else {
            ip_reass_free_complete_datagram(r);  // 超时删除
        }
    }
}

#define IP_REASS_MAXAGE 30  // 重组超时 (秒)
```

## 相关概念

- [[entities/linux/lwip/lwip-ip4-output]] — 调用 ip4_frag 的位置
- [[entities/linux/lwip/lwip-ip4-input]] — 调用 ip4_reass 的位置
- [[entities/linux/lwip/lwip-pbuf]] — 分片使用 pbuf 分配
