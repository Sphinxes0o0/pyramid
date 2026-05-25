# lwfw 网络栈 Hook 注入分析

> Hook 代码路径: `external/lwip_ds_mcu/src/core/ipv4/ip4.c` (lwip_ds_mcu 仓库)
> lwip_ds_mcu 仓库: `ssh://git@git.nevint.com/ds/foundation/lwip_ds_mcu`
> POC branch: `feature/lwfw`

---

## 1. Hook 注入位置

### 1.1 Ingress Hook — `ip4_input()`

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c`

```c
// ip4_input() 收到 IP 包后，交给上层协议之前
#ifdef NIO_LWIP_LWFW
{
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    // ===== 防火墙 Ingress 检查 =====
    if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
      pbuf_free(p);
      IP_STATS_INC(ip.drop);
      MIB2_STATS_INC(mib2.ipindiscards);
      LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipinlwfwdrops);
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

### 1.2 Egress Hook — `ip4_output_if()`

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c`

```c
// ip4_output_if() 发送 IP 包前
#ifdef NIO_LWIP_LWFW
{
  if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    // ===== 防火墙 Egress 检查 =====
    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
      MIB2_STATS_INC(mib2.ipoutdiscards);
      IP_STATS_INC(ip.drop);
      LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipoutlwfwdrops);
      return ERR_FW;  // 包已丢弃
    }
  }
}
#endif /* NIO_LWIP_LWFW */

IP_STATS_INC(ip.xmit);
return netif->linkoutput(netif, p);  // 实际发送
```

---

## 2. 全局变量初始化

### 2.1 防火墙控制块

```c
// lwfw.c:36
lwfw_firewall_t g_lwfw_firewall, *lwfw_p;
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };
```

### 2.2 lwfw_ops 函数表

```c
// lwfw.c:1996-2000
static const lwfw_firewall_ops_t lwfw_ops = {
  .firewall_ioctl  = lwfw_firewall_ioctl,
  .ingress_filter   = ip4_filter_dispatch_incoming,
  .egress_filter   = ip4_filter_dispatch_outgoing,
};

// lwfw_init() 末尾
lwfw_p->ops = &lwfw_ops;
```

### 2.3 初始化调用链

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

---

## 3. 编译选项

### 3.1 条件编译宏

| 宏 | 定义位置 | 作用 |
|----|----------|------|
| `NIO_LWIP_LWFW` | lwip_ds_mcu 配置 | 启用 lwfw 功能 |
| `NIO_LWIP_LWCT` | lwip_ds_mcu 配置 | 启用连接跟踪 |
| `LWFW_ADVANCED_FUNC_L2` | liblwfw | 启用 L2 字段过滤 |
| `LWFW_TREE_SEARCH_EN` | liblwfw | 启用树搜索模式 |
| `LWFW_UNIT_TEST` | liblwfw | 单元测试模式 |

### 3.2 编译配置检查

```c
// ip4.c 中
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE)
    lwfw_p->ops->ingress_filter(p, inp);
#endif
```

**问题**: 如果 `lwip_ds_mcu` 和 `liblwfw` 的 `NIO_LWIP_LWFW` 宏定义不一致，会导致链接错误或运行时崩溃。

---

## 4. 与 lwIP 初始化顺序

```c
// lwip.c
void lwip_init()
{
  // 1. 网络接口初始化
  netif_init();

  // 2. lwIP 协议栈初始化
  tcpip_init(NULL, NULL);

  // 3. lwfw 防火墙初始化 (在 tcpip_init 之后)
  lwfw_init();

  // 4. lwfw_agent 可能先于 lwfw 就绪
}
```

**潜在问题**: lwfw 初始化依赖于 lwIP 协议栈已就绪，但如果 lwfw 初始化失败，`lwfw_p` 为 NULL，所有防火墙 hook 会解引用空指针。

---

## 5. 宏开关安全性

### 5.1 状态检查

```c
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    ret = lwfw_p->ops->ingress_filter(p, inp);
```

**防护**: 在调用 `ops` 之前先检查 `state == ENABLE`，避免无效调用。

### 5.2 NULL 检查

```c
lwfw_init()
{
  memset((uint8_t *)lwfw_p, 0x0, sizeof(lwfw_firewall_t));
  // ...
  if (err) {
    lwfw_p->ctrl.fw_cur_status = LWFW_STATUS_ERROR;
    // 不设置 ops = &lwfw_ops，保持为 NULL
  }
}
```

**风险**: 如果初始化失败但 hook 仍然被调用（因为 `state` 初始为 0 == DISABLE），`ops` 不会被使用。但一旦 `state` 被错误配置为 ENABLE 而 `ops` 仍为 NULL，则会崩溃。

---

## 6. 延迟测试

```c
#ifdef NIO_LWIP_LWFW
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
#if LWFW_TEST_LATENCY
    t_start = raw_read_pcnt_el0();
#endif
    ret = lwfw_p->ops->ingress_filter(p, inp);
#if LWFW_TEST_LATENCY
    t_end = raw_read_pcnt_el0();
    delta_ns = (t_end - t_start) * 1000000000 / freq;
    if (in_count % 1000 == 0)
      printf("LWFW_TEST_LATENCY INPUT: %lu ns\n", delta_ns);
#endif
  }
#endif
```

使用 ARM 性能计数器 (`CNTPCT_EL0`) 测量过滤延迟，精度约数十纳秒。

---

## 7. 已知问题

### 7.1 宏定义耦合

lwip_ds_mcu 和 util_libs 必须使用相同的编译配置（相同的 `NIO_LWIP_LWFW` 等宏），否则可能导致：
- 函数指针未初始化时被调用
- 数据结构布局不一致

建议在公共头文件中统一版本号和宏定义检查。

### 7.2 初始化失败处理

```c
if (LWFW_ERR_OK != err) {
  lwfw_p->ctrl.fw_cur_status = LWFW_STATUS_ERROR;
  lwfw_resource_clean();
  sync_mutex_destroy(&lwfw_p->policy_lock);
}
// 注意: 这里没有 return！
```

初始化失败后，`g_lwfw_firewall` 已清零，`state == 0 == DISABLED`，所以 hook 会跳过。但语义上这是一个 BUG —— 初始化失败后没有 `return`。

### 7.3 错误统计扩展

```c
LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipinlwfwdrops);
```

`stats_diag_ext.ipinlwfwdrops` 是在标准 lwIP 统计结构上的扩展，需要确认 `stats_diag_ext` 在 lwip_ds_mcu 中已正确定义。
