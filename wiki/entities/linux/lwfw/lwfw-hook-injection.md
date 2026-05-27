---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Hook Injection

## 定义

LWFW 通过在 lwIPv4 输入/输出路径中插入 hook 函数指针，实现防火墙过滤逻辑的注入。Hook 点位于 `ip4_input()` 和 `ip4_output_if()` 函数内。

## Ingress Hook — ip4_input()

```c
// ip4_input() 收到 IP 包后，交给上层协议之前
#ifdef NIO_LWIP_LWFW
{
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    // ===== 防火墙 Ingress 检查 =====
    if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
      pbuf_free(p);
      IP_STATS_INC(ip.drop);
      return ERR_OK;  // 包已丢弃，返回 OK
    }
  }
}
#endif /* NIO_LWIP_LWFW */

// 继续交给传输层
switch (IPH_PROTO(iphdr)) {
  case IP_PROTO_TCP:  tcp_input(p, inp);  break;
  case IP_PROTO_UDP:   udp_input(p, inp); break;
  case IP_PROTO_ICMP:  icmp_input(p, inp); break;
}
```

## Egress Hook — ip4_output_if()

```c
// ip4_output_if() 发送 IP 包前
#ifdef NIO_LWIP_LWFW
{
  if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
      IP_STATS_INC(ip.drop);
      return ERR_FW;  // 包已丢弃
    }
  }
}
#endif /* NIO_LWIP_LWFW */

return netif->linkoutput(netif, p);  // 实际发送
```

## 函数表注册

```c
// lwfw.c
static const lwfw_firewall_ops_t lwfw_ops = {
  .firewall_ioctl  = lwfw_firewall_ioctl,
  .ingress_filter   = ip4_filter_dispatch_incoming,
  .egress_filter   = ip4_filter_dispatch_outgoing,
};

// lwfw_init() 末尾
lwfw_p->ops = &lwfw_ops;
```

## 初始化调用链

```
lwip_init()                                      [lwip.c]
  └─ lwfw_init();                              [lwfw.c:2205]
        ├─ memset(g_lwfw_firewall, 0)
        ├─ sync_mutex_new(policy_lock)
        ├─ lwfw_policies_setup()
        ├─ lwfw_manifest_parse()              → 读 YAML 配置
        ├─ lwfw_init_policy()                  → 初始化规则表
        ├─ lwct_init()                        → 连接跟踪初始化
        ├─ sys_thread_new(notification_thread) → 启动事件通知
        └─ lwfw_p->ops = &lwfw_ops
```

## 延迟测试

```c
#if LWFW_TEST_LATENCY
t_start = raw_read_pcnt_el0();
ret = lwfw_p->ops->ingress_filter(p, inp);
t_end = raw_read_pcnt_el0();
delta_ns = (t_end - t_start) * 1000000000 / freq;
printf("LWFW_TEST_LATENCY INPUT: %lu ns\n", delta_ns);
#endif
```

使用 ARM 性能计数器 (`CNTPCT_EL0`) 测量过滤延迟。

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 宏定义耦合 | P1 | lwip_ds_mcu 和 liblwfw 必须使用相同的 NIO_LWIP_LWFW 宏定义 |
| 初始化失败无 return | P0 | lwfw_init 失败分支没有 return，继续执行 |
| 状态检查 | 安全 | 调用 ops 前先检查 state == ENABLE |

## 相关概念
- [[entities/linux/safeos/safeos-lwip-lwfw-plan]]
- [[entities/linux/safeos/safeos-nsv]]

- [[entities/linux/lwfw/lwfw-architecture]] — 整体架构
- [[entities/linux/lwfw/lwfw-tcpip-thread]] — 执行上下文
- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread 详解
