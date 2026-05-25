---
type: entity
tags: [linux, lwip, network, firewall, lwfw, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# LWFW Filter Hooks — Ingress/Egress Integration Points

## 定义

LWFW Filter Hooks 是 LWFW 防火墙在 lwIP IP 层的精确集成点：Ingress Filter 在 `ip4_input()` 中、Ingress/Egress Filter 在 `ip4_output_if()` 中，作为函数调用而非内核 hook。

## Hook 位置总览

```
Ingress (RX):
ip4_input()
    ├─► IP Header 解析
    ├─► Checksum 校验
    ├─► [LWIP_HOOK_IP4_INPUT] ← 外部 hook
    ├─► [LWFW ingress_filter] ← SafeOS 防火墙 ★
    ├─► 目的地址验证
    └─► 分发到上层

Egress (TX):
ip4_output_if()
    ├─► IP Header 构造
    ├─► Checksum 计算
    ├─► [LWFW egress_filter] ← SafeOS 防火墙 ★
    ├─► [分片处理]
    └─► netif->output
```

## Ingress Hook — ip4_input()

**文件**: `ip4.c:743-770`

```c
#ifdef NIO_LWIP_LWFW
if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    #if LWFW_TEST_LATENCY
    in_count++;
    freq = raw_read_cntfrq_el0();
    t_start = raw_read_pcnt_el0();
    #endif

    if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
        pbuf_free(p);
        IP_STATS_INC(ip.drop);
        MIB2_STATS_INC(mib2.ipindiscards);
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipinlwfwdrops);
        return ERR_OK;  // 丢弃包
    }
    #if LWFW_TEST_LATENCY
    t_end = raw_read_pcnt_el0();
    delta_ns = (t_end - t_start) * 1000000000 / freq;
    #endif
}
#endif
```

## Egress Hook — ip4_output_if()

**文件**: `ip4.c:1096-1122`

```c
#ifdef NIO_LWIP_LWFW
if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
        MIB2_STATS_INC(mib2.ipoutdiscards);
        IP_STATS_INC(ip.drop);
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipoutlwfwdrops);
        return ERR_FW;  // 丢弃包
    }
}
#endif
```

## lwfw_firewall_ops 函数指针表

```c
static const lwfw_firewall_ops_t lwfw_ops = {
    .firewall_ioctl = lwfw_firewall_ioctl,
    .ingress_filter = ip4_filter_dispatch_incoming,
    .egress_filter = ip4_filter_dispatch_outgoing,
};
```

## 规则表

```c
lwfw_rule_table_t rule_tables[LWFW_MAX_COUNT_TABLE];
// LWFW_IN_TABLE  = 0  (Ingress)
// LWFW_OUT_TABLE = 1  (Egress)
```

## 与 LWIP_HOOK_IP4_INPUT 的区别

| 方面 | LWIP_HOOK_IP4_INPUT | LWFW ingress_filter |
|------|---------------------|-------------------|
| 类型 | 宏定义，外部 hook | 内部函数指针调用 |
| 返回值 | 1=消费, 0=继续 | ERR_OK=允许, 其他=丢弃 |
| 特性 | 无状态 | 支持状态追踪、事件上报 |
| 位置 | IP header 解析之后 | LWIP_HOOK_IP4_INPUT 之后 |

## 执行上下文

- **Ingress**: 在 `LOCK_TCPIP_CORE()` 内执行，执行线程为 `nic_rx_thread`
- **Egress**: 在 `LOCK_TCPIP_CORE()` 内执行，执行线程为调用 `ip4_output()` 的线程

## 相关概念

- [[entities/linux/lwip/lwip-firewall]] — 完整防火墙架构
- [[entities/linux/lwip/lwip-sel4-function]] — 整体调用链
- [[entities/linux/lwip/lwip-raw-socket]] — RAW socket / cBPF

## 来源详情

- [[sources/safeos-lwip-extensions]]
