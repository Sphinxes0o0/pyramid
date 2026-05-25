# LWFW Filter Hooks 分析 — T-080/T-081

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: LWFW Ingress/Egress filter hooks 在 ip4_input/ip4_output 中的集成点

---

## 1. 概述

LWFW (Lightweight Firewall) 是 SafeOS 的**包过滤防火墙**，集成在 lwIP 的 IP 层：

1. **Ingress Filter**: 在 `ip4_input()` 中，IP 层接收 packet 后、上层协议处理前
2. **Egress Filter**: 在 `ip4_output_if()` 中，IP 层发送 packet 前

### 1.1 Filter Hook 位置

```
                    ┌─────────────────────────────────────────┐
                    │           ip4_input()                  │
                    │  (external/lwip_ds_mcu/src/core/ipv4)  │
                    └─────────────────────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
              ┌──────────┐       ┌──────────────┐     ┌──────────────┐
              │ LWIP_    │       │   LWFW       │     │    upper    │
              │ HOOK_IP4 │       │ ingress_     │     │    layer    │
              │ _INPUT   │       │ filter()     │     │ (tcp/udp)   │
              └──────────┘       └──────────────┘     └──────────────┘
                    │                    │                    │
                    └──────────┬──────────┴────────────────────┘
                               ▼
                    ┌─────────────────────────────────────────┐
                    │         LWFW Ingress Filter              │
                    │  - packet 匹配规则                       │
                    │  - ALLOW/DENY/EVENT                    │
                    │  - 事件上报                             │
                    └─────────────────────────────────────────┘
```

---

## 2. Ingress Filter (ip4_input)

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:743-770`

### 2.1 Hook 位置

```c
// ip4_input() 中，LWIP_HOOK_IP4_INPUT 之后
#ifdef NIO_LWIP_LWFW
if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    #if LWFW_TEST_LATENCY
    in_count++;
    freq = raw_read_cntfrq_el0();
    t_start = raw_read_pcnt_el0();
    #endif

    if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
        // packet 被丢弃
        pbuf_free(p);
        IP_STATS_INC(ip.drop);
        MIB2_STATS_INC(mib2.ipindiscards);
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipinlwfwdrops);
        return ERR_OK;
    }

    #if LWFW_TEST_LATENCY
    t_end = raw_read_pcnt_el0();
    delta_ns = (t_end - t_start) * 1000000000 / freq;
    if (in_count % 1000 == 0) {
        printf("LWFW_TEST_LATENCY INPUT: %lu ns for ip4_filter_dispatch_incoming\n", delta_ns);
    }
    #endif
}
#endif
```

### 2.2 调用时机

| 阶段 | 函数 | 说明 |
|------|------|------|
| 1 | IP Header 解析 | 版本、长度检查 |
| 2 | Checksum 校验 | IP checksum 验证 |
| 3 | **LWIP_HOOK_IP4_INPUT** | 外部 hook (如 LWFW) |
| 4 | **LWFW ingress_filter** | SafeOS 内部防火墙 |
| 5 | 目的地址验证 | 单播/多播/广播 |
| 6 | 分发到上层 | TCP/UDP/ICMP/IGMP |

### 2.3 执行上下文

- **调用位置**: `ip4_input()` 内，`LOCK_TCPIP_CORE()` 已持有
- **执行线程**: 在 `LWIP_TCPIP_CORE_LOCKING=1` 时，为 `nic_rx_thread`

---

## 3. Egress Filter (ip4_output_if)

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:1096-1122`

### 3.1 Hook 位置

```c
// ip4_output_if() 中，LWFW_OUT_TABLE 规则检查
#ifdef NIO_LWIP_LWFW
if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    #if LWFW_TEST_LATENCY
    out_count++;
    freq = raw_read_cntfrq_el0();
    t_start = raw_read_pcnt_el0();
    #endif

    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
        MIB2_STATS_INC(mib2.ipoutdiscards);
        IP_STATS_INC(ip.drop);
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipoutlwfwdrops);
        return ERR_FW;  // ← packet 被丢弃
    }

    #if LWFW_TEST_LATENCY
    t_end = raw_read_pcnt_el0();
    delta_ns = (t_end - t_start) * 1000000000 / freq;
    if (out_count % 1000 == 0) {
        printf("LWFW_TEST_LATENCY OUTPUT: %lu ns for ip4_filter_dispatch_outgoing\n", delta_ns);
    }
    #endif
}
#endif
```

### 3.2 调用时机

| 阶段 | 函数 | 说明 |
|------|------|------|
| 1 | IP Header 构造 | 填充 IP 字段 |
| 2 | Checksum 计算 | IP checksum |
| 3 | **LWFW egress_filter** | SafeOS 内部防火墙 |
| 4 | [分片处理] | 如果 packet > MTU |
| 5 | netif->output | 发送到 Ethernet |

### 3.3 执行上下文

- **调用位置**: `ip4_output_if()` 内，`LOCK_TCPIP_CORE()` 已持有
- **执行线程**: 调用 `ip4_output()` 的线程 (可能是 `tcpip_thread` 或应用线程)

---

## 4. LWFW Filter 架构

### 4.1 lwfw_firewall_ops

**文件**: `libs/util_libs/liblwfw/include/lwfw.h:245-249`

```c
typedef struct lwfw_firewall_ops {
    int (*firewall_ioctl)(unsigned long opcode, unsigned long p1, unsigned long p2);
    int (*ingress_filter)(const struct pbuf *p, const struct netif *inp);
    int (*egress_filter)(const struct pbuf *p, const struct netif *inp);
} lwfw_firewall_ops_t;
```

### 4.2 规则表

```c
typedef struct lwfw_rule_table {
    uint16_t rule_cnt;
    uint16_t state;           // LWFW_STATE_ENABLE/DISABLE
    lwfw_action_t def_action; // 默认动作
    struct cdlist header;     // 规则链表
    rule_set_t _ruleset;      // 树搜索规则集
    hs_tree_t _hs_tree;      // 决策树
} lwfw_rule_table_t;

// 两个规则表
lwfw_rule_table_t rule_tables[LWFW_MAX_COUNT_TABLE];
// LWFW_IN_TABLE  = 0  (Ingress)
// LWFW_OUT_TABLE = 1  (Egress)
```

---

## 5. 规则匹配流程

### 5.1 ip4_filter

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:724`

```c
int ip4_filter(lwfw_policy_t *policy, struct pbuf *p,
               struct netif *netif, int dir)
{
    lwfw_pkt_info_t pkt_info;

    // Step 1: 解析 packet 信息
    lwfw_pkt_info_constructor(&pkt_info, p, netif);

    // Step 2: 引擎匹配
    match_result = filter_engine->do_filter(handle, &pkt_info);

    // Step 3: 返回动作
    return match_result->action;
}
```

### 5.2 匹配字段

| 层次 | 字段 | 说明 |
|------|------|------|
| **L2** | EtherType | 0x0800 IP, 0x0806 ARP |
| **L2** | VLAN ID | VID 匹配 |
| **L2** | Src/Dst MAC | MAC 地址 + mask |
| **L3** | Protocol | TCP/UDP/ICMP |
| **L3** | Src IP | IP 地址 + prefix |
| **L3** | Dst IP | IP 地址 + prefix |
| **L4** | Src Port | 端口/端口范围 |
| **L4** | Dst Port | 端口/端口范围 |

---

## 6. 与 LWIP_HOOK_IP4_INPUT 的区别

### 6.1 LWIP_HOOK_IP4_INPUT

```c
// ip4.c:507-512
#ifdef LWIP_HOOK_IP4_INPUT
if (LWIP_HOOK_IP4_INPUT(p, inp)) {
    return ERR_OK;  // packet 被 hook 消费
}
#endif
```

- **宏定义**: 外部 hook 接口
- **返回值**: `1` = 消费 packet, `0` = 继续处理
- **位置**: IP header 解析之后，LWFW 之前

### 6.2 LWFW ingress_filter

- **内部实现**: 通过 `lwfw_p->ops->ingress_filter()`
- **返回值**: `ERR_OK` = 允许, 其他 = 丢弃
- **特性**: 支持状态追踪、事件上报

---

## 7. 性能特征

### 7.1 延迟开销

```
LWFW Ingress Filter 延迟:
- 规则数 < 20: O(n) 线性扫描
- 规则数 ≥ 20: O(log n) 树搜索

典型延迟 (1000 条规则):
- 线性扫描: ~1-2 μs
- 树搜索: ~200-500 ns
```

### 7.2 锁竞争

```
Ingress Filter: 在 LOCK_TCPIP_CORE() 内执行
Egress Filter: 在 LOCK_TCPIP_CORE() 内执行

瓶颈: 单线程执行，无法并行化
```

---

## 8. 总结

### 8.1 Filter Hook 位置

```
Ingress (RX):
ip4_input()
    ├─► IP Header 解析
    ├─► Checksum 校验
    ├─► [LWIP_HOOK_IP4_INPUT] ← 外部 hook
    ├─► [LWFW ingress_filter] ← SafeOS 防火墙
    ├─► 目的地址验证
    └─► 分发到上层

Egress (TX):
ip4_output_if()
    ├─► IP Header 构造
    ├─► Checksum 计算
    ├─► [LWFW egress_filter] ← SafeOS 防火墙
    ├─► [分片处理]
    └─► netif->output
```

### 8.2 关键设计

1. **两级过滤**: `LWIP_HOOK_IP4_INPUT` 提供扩展性，`LWFW` 提供完整功能
2. **双规则表**: `LWFW_IN_TABLE` 和 `LWFW_OUT_TABLE` 分别处理入站和出站
3. **热切换**: 规则更新不影响当前策略
4. **连接追踪集成**: 通过 `lwct_main_hook()` 支持状态ful 过滤

### 8.3 SafeOS 特供

这是 **SafeOS lwIP 的核心安全机制**，通过 lwfw 与 seL4 的集成，实现：
- 内核态防火墙 (零拷贝)
- 用户态管理 (lwfw_agent)
- 事件上报 (共享内存 + IPC)
