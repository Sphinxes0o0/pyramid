---
type: source
source-type: github
created: 2026-05-27
title: "SafeOS lwIP 防火墙实现深度分析"
date: 2026-04-13
size: medium
path: raw/safeos/docs/lwip_firewall_analysis.md
summary: "lwfw无状态包过滤+lwct连接跟踪+cBPF socket级过滤三层架构：hook精确位置(ip4_input/ip4_output)、过滤函数调用链、lwfw_pkt_info解析、规则匹配(check_rule/check_l2/l3/l4_info)"
tags: [safeos, lwfw, lwip, firewall, cBPF, connection-tracking, packet-filter, seL4]
sources: []
---

# SafeOS lwIP 防火墙实现深度分析

> 文档版本: 1.0 | 更新日期: 2026/04/13

## 三层防护架构

```
┌─────────────────────────────────────────────────────────────┐
│ Ingress (RX): NIC → used_rx_buf_ring → rx_callback()       │
│   → ethernet_input() → ip4_input()                          │
│     → lwfw_ops.ingress_filter()  ←── lwfw 无状态过滤        │
│       → ip4_filter_dispatch_incoming()                    │
│         → ip4_filter() → list_search_do_filter()           │
│           → check_rule() → check_l2/l3/l4_info()          │
│     → raw_input() → raw_afpacket_input()                   │
│       → lwip_run_socket_filter()  ←── cBPF socket 过滤      │
│         → bpf_filter_run() → cbpf_execute_interpreter()    │
│     → tcp_input() / udp_input() / icmp_input()            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Egress (TX): App → sys_sendto_nb() → lwip_sendto()        │
│   → tcp_output() / udp_output() / raw_output()             │
│   → ip4_output_if()                                        │
│     → lwfw_ops.egress_filter()  ←── lwfw 无状态过滤        │
│       → ip4_filter_dispatch_outgoing()                     │
│     → netif->linkoutput = ethif_link_output()              │
│       → pending_tx_buf_ring → NIC                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Hook 精确位置

### Ingress Hook — `ip4_input()` 入口

```c
// ip4.c:743-770
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
      pbuf_free(p);
      IP_STATS_INC(ip.drop);
      return ERR_OK;  // 丢弃包
    }
  }
#endif
```

### Egress Hook — `ip4_output_if()` 出口

```c
// ip4.c:1096-1122
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
      return ERR_FW;  // 丢弃包
    }
  }
#endif
  return netif->linkoutput(netif, p);
```

---

## 过滤函数调用链

### Ingress

```
ip4_input()
 └─ lwfw_p->ops->ingress_filter(p, inp)
     └─ ip4_filter_dispatch_incoming()     [lwfw.c:802]
         └─ ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE)  [lwfw.c:724]
             ├─ lwfw_pkt_info_constructor()  [lwfw.c:329]
             │   ├─ 从 pbuf 解析 IP 头
             │   ├─ 从 pbuf 解析 TCP/UDP 头 (端口)
             │   └─ 填充 lwfw_pkt_l3_info_t, lwfw_pkt_l4_info_t
             └─ filter_engine->do_filter()
                 └─ list_search_do_filter()  [lwfw.c:1884]
                     └─ check_rule()  [lwfw.c:565]
                         ├─ 连接状态匹配 (lwct)
                         ├─ 网卡接口匹配
                         ├─ check_lwfw_l2_info()  [lwfw.c:383]
                         ├─ check_lwfw_l3_info()  [lwfw.c:447]
                         └─ check_lwfw_l4_info()  [lwfw.c:498]
```

---

## 规则匹配 — check_rule

```c
// lwfw.c:565-613
static bool check_rule(const lwfw_rule_t *rule, const lwfw_pkt_info_t *info, ...)
{
    // 1. 连接状态匹配 (lwct)
    #ifdef NIO_LWIP_LWCT
        if ((rule->flags & LWFW_RULE_FLAGS_CT_STATE) &&
            rule->ct_state != info->ct_state) return 0;
    #endif

    // 2. 网卡接口匹配
    if ((rule->flags & LWFW_RULE_FLAGS_NETIF) &&
        strncmp(rule->interface.if_name, info->interface.if_name, ...) != 0) return 0;

    // 3. L2 层匹配 (EtherType, VLAN, MAC)
    #ifdef LWFW_ADVANCED_FUNC_L2
        if (!check_lwfw_l2_info(rule, &info->l2)) return 0;
    #endif

    // 4. L3 层匹配 (IP, Protocol, CIDR)
    if (!check_lwfw_l3_info(rule, &info->l3)) return 0;

    // 5. L4 层匹配 (端口范围/列表)
    if (!check_lwfw_l4_info(rule, &info->l4)) return 0;

    return 1;  // 全部匹配
}
```

---

## L3 字段匹配 — check_lwfw_l3_info

```c
// lwfw.c:447-488
static bool check_lwfw_l3_info(const lwfw_rule_t *rule, const lwfw_pkt_l3_info_t *packet_info)
{
    // 协议匹配 (IP/ICMP/TCP/UDP)
    if ((rule->flags & LWFW_RULE_FLAGS_PROTOCOL) &&
        rule_l3_info->proto != LWFW_RULE_PROTO_IP &&
        rule_l3_info->proto != packet_info->proto) return 0;

    // 源 IP + CIDR 掩码匹配
    if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK_LEN) {
        rule_mask = ~((1UL << (32 - masklen)) - 1);
        if ((packet_info->src_ip & rule_mask) != (rule_l3_info->src_ip & rule_mask)) return 0;
    }

    // 目标 IP + CIDR 掩码匹配
    if (rule->flags & LWFW_RULE_FLAGS_DST_IP_MASK_LEN) {
        rule_mask = ~((1UL << (32 - masklen)) - 1);
        if ((packet_info->dst_ip & rule_mask) != (rule_l3_info->dst_ip & rule_mask)) return 0;
    }
    return 1;
}
```

---

## cBPF Socket 级过滤

### raw_afpacket_input — AF_PACKET 分发

```c
// raw.c:282-345
raw_input_state_t raw_afpacket_input(struct pbuf *p, struct netif *inp, u16_t type)
{
    while (pcb != NULL) {
        // 1. 协议匹配
        skip_this_frame = (PP_HTONS(pcb->protocol) != type);

        // 2. PCB 状态检查
        if (pcb->state == AF_PACKET_STATE_INIT) skip_this_frame = 1;

        // 3. 网卡绑定检查
        if ((pcb->netif_idx != AF_PACKET_NOBIND) &&
            (pcb->netif_idx != netif_get_index(inp))) skip_this_frame = 1;

        // 4. cBPF 过滤器匹配
        if (!skip_this_frame && pcb->recv != NULL &&
            lwip_run_socket_filter(pcb->conn, p, inp) != 0) {
            pcb->recv(pcb->recv_arg, pcb, p, NULL);  // tpacket_recv 回调
        }
        pcb = pcb->next;
    }
}
```

### bpf_filter_run — cBPF 执行

```c
// bpf_filter.c:548-562
int bpf_filter_run(struct bpf_program *fprog, struct pbuf *pb)
{
    if (fprog == NULL) return 1;  // 无过滤器，接受所有
    if (pb == NULL) return 0;
    pkt_len = cbpf_execute_interpreter(fprog, pb);
    return pkt_len ? 1 : 0;  // 1=匹配(捕获), 0=不匹配(跳过)
}
```

---

## 数据结构

### lwfw_rule — 防火墙规则

```c
struct lwfw_rule {
    struct cdlist next;              // 链表节点
    uint16_t index;                  // 规则唯一索引
    uint16_t state;                 // LWFW_STATE_ENABLE / DISABLE
    uint16_t ct_state;              // 连接跟踪状态
    uint32_t flags;                  // 匹配标志位图
    lwfw_netif_t interface;         // 网卡接口名
    lwfw_rule_l2_info_t l2;        // L2: EtherType, VLAN, MAC
    lwfw_rule_l3_info_t l3;        // L3: src/dst IP, Protocol
    lwfw_rule_l4_info_t l4;        // L4: src/dst Port
    lwfw_action_t action;            // 动作: DENY / EVENT / LOGGING
    rate_limit_t rlimit;            // 速率限制
    uint32_t hit_cnt;               // 命中计数
};
```

### lwfw_pkt_info_t — 数据包信息

```c
typedef struct lwfw_pkt_info {
    lwfw_netif_t interface;
    lwfw_pkt_l2_info_t l2;   // EtherType, VLAN, MAC
    lwfw_pkt_l3_info_t l3;   // src_ip, dst_ip, proto
    lwfw_pkt_l4_info_t l4;   // src_port, dst_port
    lwfw_direction_t dir;    // LWFW_DIR_RX / LWFW_DIR_TX
    #ifdef NIO_LWIP_LWCT
        uint16_t ct_state;
    #endif
} lwfw_pkt_info_t;
```

### 匹配标志位

| 标志 | 说明 |
|------|------|
| `LWFW_RULE_FLAGS_ETHER_TYPE` | EtherType 匹配 |
| `LWFW_RULE_FLAGS_VLAN` | VLAN ID 匹配 |
| `LWFW_RULE_FLAGS_SRC_MAC` | 源 MAC + 掩码 |
| `LWFW_RULE_FLAGS_DST_MAC` | 目标 MAC + 掩码 |
| `LWFW_RULE_FLAGS_PROTOCOL` | L3 协议匹配 |
| `LWFW_RULE_FLAGS_SRC_IP_MASK_LEN` | 源 IP + CIDR |
| `LWFW_RULE_FLAGS_DST_IP_MASK_LEN` | 目标 IP + CIDR |
| `LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE` | 源端口范围 |
| `LWFW_RULE_FLAGS_DST_L4_PORT_RANGE` | 目标端口范围 |
| `LWFW_RULE_FLAGS_NETIF` | 网卡接口匹配 |
| `LWFW_RULE_FLAGS_CT_STATE` | 连接状态匹配 |
| `LWFW_RULE_FLAGS_RATE_LIMIT` | 速率限制 |

---

## 两种过滤引擎

| 引擎 | 特点 | 时间复杂度 |
|------|------|-----------|
| 链表搜索引擎 (默认) | 无索引，按顺序遍历，First-Match | O(n) |
| 树搜索引擎 (Hyperscan-like) | 多维索引，支持批量预编译 | O(log n) |

---

## 与 Linux netfilter 的关键差异

| 方面 | Linux netfilter | SafeOS lwfw |
|------|----------------|-------------|
| 架构 | 内核态 hook (NF_HOOK) | 用户态 lwIP 函数调用 |
| 规则组织 | 链表 / ipset / nftable | 单向链表或树 |
| 连接跟踪 | 内核 conntrack | 用户态 lwct (可选) |
| 匹配字段 | 完整 L1-L4 + mark/connmark | L2 + L3 + L4 |
| 动作 | ACCEPT/DROP/REJECT/LOG | DENY/EVENT/LOGGING |

---

## 关键文件清单

| 文件 | 职责 |
|------|------|
| `ip4.c:743,1096` | 防火墙 hook 调用点 |
| `lwfw.c` | ip4_filter, check_rule, list_search_do_filter |
| `lwfw.h` | lwfw_rule, lwfw_policy, lwfw_rule_table |
| `lwfw_common.h` | lwfw_pkt_info_t, 动作/标志枚举 |
| `lwfw_parser.c` | YAML 规则解析 |
| `lwfw_notif.c` | lwfw_event_fifo 实现 |
| `bpf_filter.c` | cBPF 解释器 |
| `raw.c:282` | raw_afpacket_input, cBPF 调用点 |
| `sockets.c:580,4434` | lwip_run_socket_filter, SO_ATTACH_FILTER |

---

## 相关页面

- [[sources/safeos-lwfw]] — LWFW 防火墙完整分析
- [[lwfw-index]] — LWFW 模块索引
- [[lwip-index]] — lwIP 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引
