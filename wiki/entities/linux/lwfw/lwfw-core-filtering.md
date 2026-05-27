---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Core Filtering

## 定义

`lwfw_core_filtering` 是 LWFW 的核心过滤逻辑模块，包含 Ingress/Egress 过滤入口、包信息解析、规则链表遍历、速率限制和热切换机制。

## 过滤入口

### Ingress: ip4_filter_dispatch_incoming

```c
static int ip4_filter_dispatch_incoming(const struct pbuf *p, const struct netif *inp)
{
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_DISABLE)
    return ERR_OK;

  ret = ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE);

  if ((ret & LWFW_ACTION_CODE_DENY) == LWFW_ACTION_CODE_DENY) {
    LWFW_STATICS_INC(g_lwfw_stats.total_rx_drop);
    return ERR_VAL;  // 丢弃
  }
  return ERR_OK;
}
```

### Egress: ip4_filter_dispatch_outgoing

流程与 Ingress 基本一致，区别在于 `dir = LWFW_OUT_TABLE`，且 Egress 方向 L2 字段不填充。

## 包解析

```c
lwfw_pkt_info_constructor(p, inp, pkt_info, dir)
{
  ip_hdr = (struct ip_hdr *)p->payload;
  trans_hdr = p->payload + IPH_HL_BYTES(ip_hdr) * 4;

  // L3
  l3->src_ip = lwip_ntohl(ip_hdr->src.addr);
  l3->dst_ip = lwip_ntohl(ip_hdr->dest.addr);
  l3->proto  = ip_hdr->_proto;

  // L4
  switch (proto) {
    case TCP:  src = TCP_HDR.src;  dst = TCP_HDR.dst;  break;
    case UDP:  src = UDP_HDR.src;  dst = UDP_HDR.dst;  break;
  }

  // L2 (Ingress only)
  if (dir == LWFW_IN_TABLE) {
    eth_hdr = (struct eth_hdr *)((uint8_t *)p + SIZEOF_STRUCT_PBUF);
    l2->vlan = VLAN_ID(vlan_hdr);
  }

  // 连接状态
  pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);
}
```

## 规则匹配 check_rule

```c
static bool check_rule(rule, info, dir)
{
  // 1. CT_STATE 匹配
  if (flags & CT_STATE && rule->ct_state != info->ct_state)
    return false;

  // 2. NETIF 接口匹配
  if (flags & NETIF && strncmp(...) != 0)
    return false;

  // 3. L2 字段匹配 (需 LWFW_ADVANCED_FUNC_L2)
  if (!check_lwfw_l2_info(rule, &info->l2))
    return false;

  // 4. L3 字段匹配
  if (!check_lwfw_l3_info(rule, &info->l3))
    return false;

  // 5. L4 字段匹配
  if (!check_lwfw_l4_info(rule, &info->l4))
    return false;

  return true;
}
```

## 速率限制

```
状态转换:
NORMAL ──(rx_pps >= burst)──► LIMIT ──(time >= expire)──► NORMAL

限速逻辑:
1. rx_pps++ (原子)
2. LIMIT 状态下:
   - 如果 action 不是 DENY 且 rx_pps > rate → 拒绝
3. NORMAL 状态下: 不上报 event
```

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 深拷贝阻塞持锁 | P1 | 规则数量大时持锁数百毫秒 |
| 静态 parser state | P0 | 多线程并发解析时冲突 |
| L2 解析默认关闭 | P2 | VLAN/MAC 过滤默认不可用 |

## 相关概念
- [[entities/linux/safeos/safeos-nsv]]
- [[entities/linux/safeos/safeos-lwip-lwfw-plan]]
- [[entities/linux/lwip/lwip-lwfw-filter-hooks]]

- [[entities/linux/lwfw/lwfw-architecture]] — 引擎抽象架构
- [[entities/linux/lwfw/lwfw-hook-injection]] — Hook 注入点
- [[entities/linux/lwfw/lwfw-list-search]] — 线性扫描引擎
- [[entities/linux/lwfw/lwfw-tree-search]] — 树搜索引擎
- [[entities/linux/lwfw/lwfw-hotswap-analysis]] — 热切换机制
- [[entities/linux/lwfw/lwfw-parser-concurrency]] — 解析器并发问题
