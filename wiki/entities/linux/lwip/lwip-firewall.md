---
type: entity
tags: [linux, lwip, network, firewall, lwfw, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# SafeOS LWFW Firewall — Lightweight Firewall

## 定义

LWFW (Lightweight Firewall) 是 SafeOS lwIP 的**三层安全防护体系**：
1. **lwfw** — 无状态包过滤 (L2/L3/L4 规则匹配)
2. **lwct** — 连接追踪 (状态ful 过滤)
3. **cBPF** — socket 级过滤 (AF-PACKET capture)

## 防火墙架构

```
Ingress (RX):
NIC → used_rx_buf_ring → rx_callback → ethernet_input → ip4_input
    → lwfw_ops.ingress_filter() ←── lwfw 无状态过滤
        → ip4_filter_dispatch_incoming()
            → ip4_filter()
                → list_search_do_filter()
                    → check_rule() → check_l2/l3/l4_info()
    → raw_input() → raw_afpacket_input()
        → lwip_run_socket_filter() ←── cBPF socket 过滤

Egress (TX):
App → sys_net_sendto() → lwip_sendto()
    → tcp_output() / udp_output() → ip4_output_if()
        → lwfw_ops.egress_filter() ←── lwfw 无状态过滤
            → ip4_filter_dispatch_outgoing()
                → ip4_filter() ...
        → netif->linkoutput = ethif_link_output()
```

## 防火墙 Hook 位置

### Ingress Hook — ip4_input() 入口
**文件**: `ip4.c:743-770`
```c
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
    pbuf_free(p); IP_STATS_INC(ip.drop);
    return ERR_OK;  // 丢弃包
  }
#endif
```

### Egress Hook — ip4_output_if() 出口
**文件**: `ip4.c:1096-1122`
```c
#ifdef NIO_LWIP_LWFW
  err_t ret = lwfw_p->ops->egress_filter(p, netif);
  if (ret != ERR_OK) {
    return ERR_FW;  // 丢弃包
  }
#endif
```

## 规则匹配流程

```
ip4_input() → lwfw_p->ops->ingress_filter()
    → ip4_filter_dispatch_incoming()
        → ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE)
            → lwfw_pkt_info_constructor()  // 解析 packet → lwfw_pkt_info_t
            → filter_engine->do_filter()
                → list_search_do_filter()
                    → check_rule()
                        ├─ check_lwfw_l2_info()  // EtherType, VLAN, MAC
                        ├─ check_lwfw_l3_info()  // IP, Protocol
                        └─ check_lwfw_l4_info()  // Ports
```

## 匹配字段

| 层次 | 字段 | 说明 |
|------|------|------|
| L2 | EtherType | ETH_P_IP / ETH_P_ARP / ETH_P_ALL |
| L2 | VLAN ID | VID 匹配 |
| L2 | Src/Dst MAC | MAC 地址 + mask |
| L3 | Protocol | TCP/UDP/ICMP |
| L3 | Src/Dst IP | IP 地址 + CIDR prefix |
| L4 | Src/Dst Port | 端口/端口范围 |

## 动作定义

```c
typedef enum lwfw_action {
  LWFW_ACTION_CODE_DENY     = BIT(0),  // 拒绝
  LWFW_ACTION_CODE_EVENT    = BIT(1),  // 生成安全事件
  LWFW_ACTION_CODE_LOGGING  = BIT(2),  // 记录日志
} lwfw_action_t;
// 支持组合: DENY | EVENT
```

## cBPF Socket 过滤

```
raw_afpacket_input() → lwip_run_socket_filter()
    → bpf_filter_run()
        → cbpf_execute_interpreter()
            → bpf_filter_with_aux_data()
```

支持 SO_ATTACH_FILTER setsockopt 附加 cBPF 程序。

## 与 Linux netfilter 对比

| 方面 | Linux netfilter | SafeOS lwfw |
|------|----------------|-------------|
| 架构 | 内核态 hook | 用户态 lwIP 函数调用 |
| 规则组织 | 链表 / ipset | 单向链表 (list) 或树 (tree) |
| 连接跟踪 | 内核 conntrack | 用户态 lwct (可选) |
| 匹配字段 | 完整 L1-L4 + mark | L2(LWFW ADVANCED) + L3 + L4 |
| 动作 | ACCEPT/DROP/REJECT | DENY / EVENT / LOGGING |

## 相关概念

- [[entities/linux/lwip/lwip-lwfw-filter-hooks]] — Hook 精确位置
- [[entities/linux/lwip/lwip-raw-socket]] — RAW socket / cBPF
- [[entities/linux/lwip/lwip-sel4-function]] — 整体 lwIP 调用链

## 来源详情

- [[sources/safeos-lwip-extensions]]
