# SafeOS lwIP 防火墙实现深度分析 — 函数级

> 文档版本: 1.0
> 更新日期: 2026/04/13
> 代码路径: `/home/shiyang/nio/nt35/safeos/`

---

## 1. 整体架构

SafeOS lwIP 防火墙由 **lwfw** (无状态包过滤) + **lwct** (连接跟踪) + **cBPF** (socket级过滤) 三层组成:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Ingress (RX) Path                            │
│  NIC → used_rx_buf_ring → rx_callback()                        │
│      → ethernet_input() → ip4_input()                          │
│          → lwfw_ops.ingress_filter()  ←── lwfw 无状态过滤       │
│              → ip4_filter_dispatch_incoming()                   │
│                  → ip4_filter()                                 │
│                      → list_search_do_filter()                  │
│                          → check_rule() → check_l2/l3/l4_info()│
│                                                                   │
│          → raw_input() → raw_afpacket_input()                   │
│              → lwip_run_socket_filter()   ←── cBPF socket过滤    │
│                  → bpf_filter_run()                             │
│                                                                   │
│          → tcp_input() / udp_input() / icmp_input()            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Egress (TX) Path                              │
│  App → sys_sendto_nb() → lwip_sendto()                        │
│      → tcp_output() / udp_output() / raw_output()             │
│      → ip4_output_if()                                          │
│          → lwfw_ops.egress_filter()  ←── lwfw 无状态过滤       │
│              → ip4_filter_dispatch_outgoing()                   │
│                  → ip4_filter() ...                            │
│                                                                   │
│          → netif->linkoutput = ethif_link_output()             │
│              → pending_tx_buf_ring → NIC                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 防火墙 Hook 精确位置

### 2.1 Ingress Hook — `ip4_input()` 入口

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:743-770`

```c
#ifdef NIO_LWIP_LWFW
  {
    if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
#if LWFW_TEST_LATENCY
      in_count++;
      freq = raw_read_cntfrq_el0();
      t_start = raw_read_pcnt_el0();
#endif
      // ===== 防火墙入口 #1 =====
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
      if (in_count % 1000 == 0) {
        printf("LWFW_TEST_LATENCY INPUT: %lu ns\n", delta_ns);
      }
#endif
    }
  }
#endif /* NIO_LWIP_LWFW */

  /* 继续上传给上层协议 */
  switch (IPH_PROTO(iphdr)) {
    case IP_PROTO_TCP: tcp_input(p, inp); break;
    case IP_PROTO_UDP: udp_input(p, inp); break;
    case IP_PROTO_ICMP: icmp_input(p, inp); break;
  }
```

### 2.2 Egress Hook — `ip4_output_if()` 出口

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:1096-1122`

```c
#ifdef NIO_LWIP_LWFW
  {
    if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
#if LWFW_TEST_LATENCY
      out_count++;
      freq = raw_read_cntfrq_el0();
      t_start = raw_read_pcnt_el0();
#endif
      // ===== 防火墙入口 #2 =====
      err_t ret = lwfw_p->ops->egress_filter(p, netif);
      if (ret != ERR_OK) {
        MIB2_STATS_INC(mib2.ipoutdiscards);
        IP_STATS_INC(ip.drop);
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.ipoutlwfwdrops);
        return ERR_FW;  // 丢弃包
      }
#if LWFW_TEST_LATENCY
      t_end = raw_read_pcnt_el0();
      delta_ns = (t_end - t_start) * 1000000000 / freq;
      if (out_count % 1000 == 0) {
        printf("LWFW_TEST_LATENCY OUTPUT: %lu ns\n", delta_ns);
      }
#endif
    }
  }
#endif /* NIO_LWIP_LWFW */

  IP_STATS_INC(ip.xmit);
  return netif->linkoutput(netif, p);  // 实际发送
```

---

## 3. 完整过滤函数调用链

### 3.1 Ingress (入方向)

```
ip4_input()                                           [ip4.c]
 └─ lwfw_p->ops->ingress_filter(p, inp)               [函数指针]
     └─ ip4_filter_dispatch_incoming()                [lwfw.c:802]
         └─ ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE)  [lwfw.c:724]
             ├─ lwfw_pkt_info_constructor()           [lwfw.c:329]
             │   ├─ 从 pbuf 解析 IP 头
             │   ├─ 从 pbuf 解析 TCP/UDP 头 (端口)
             │   ├─ 填充 lwfw_pkt_l3_info_t (src_ip, dst_ip, proto)
             │   └─ 填充 lwfw_pkt_l4_info_t (src_port, dst_port)
             └─ filter_engine->do_filter()           [lwfw.c:758]
                 └─ list_search_do_filter()           [lwfw.c:1884]
                     ├─ 遍历 cdlist 规则链表 (按插入顺序)
                     └─ check_rule()                  [lwfw.c:565]
                         ├─ 连接状态匹配 (lwct)
                         ├─ 网卡接口匹配
                         ├─ check_lwfw_l2_info()     [lwfw.c:383]
                         ├─ check_lwfw_l3_info()     [lwfw.c:447]
                         └─ check_lwfw_l4_info()      [lwfw.c:498]
```

### 3.2 Egress (出方向)

```
ip4_output_if()                                       [ip4.c]
 └─ lwfw_p->ops->egress_filter(p, netif)             [函数指针]
     └─ ip4_filter_dispatch_outgoing()              [lwfw.c:847]
         └─ ip4_filter(lwfw_p, p, netif, LWFW_OUT_TABLE)  [lwfw.c:724]
             └─ (同上，区别在 dir = LWFW_OUT_TABLE，
                 L2 字段不填充，因为出口时没有 Ethernet 头)
```

---

## 4. 核心函数详解

### 4.1 `ip4_filter_dispatch_incoming()` — 入方向调度器

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:802-834`

```c
static int ip4_filter_dispatch_incoming(const struct pbuf *p, const struct netif *inp)
{
  int ret = ERR_OK;

  // 检查 IN_TABLE 是否启用
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_DISABLE)
    return ret;

  LWIP_ASSERT_CORE_LOCKED();
  LWIP_ASSERT("invalid pbuf *p", (p != NULL));
  LWIP_ASSERT("invalid netif *inp", (inp != NULL));

  // 调用通用 ip4_filter
  ret = ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE);

  // 处理返回动作
  if ((ret & LWFW_ACTION_CODE_DENY) == LWFW_ACTION_CODE_DENY) {
    LWFW_STATICS_INC(g_lwfw_stats.total_rx_drop);
    return ERR_VAL;  // 丢弃
  } else {
    return ERR_OK;  // 允许通过
  }
}
```

### 4.2 `ip4_filter()` — 过滤执行核心

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:724-789`

```c
static int ip4_filter(lwfw_firewall_t *fw, const struct pbuf *p,
                      const struct netif *inp, lwfw_table_flag_t dir)
{
  int ret = LWFW_ERR_OK;
  lwfw_pkt_info_t pkt_info = {0};
  match_result_t ret_rule = {0};
  lwfw_policy_t *policy = fw->policy;
  const lwfw_backend_engine_t *filter_engine = policy->filter_engine;

  // 连接跟踪兜底机制
#ifdef NIO_LWIP_LWCT
  if (lwct_enable == 1 && !p->_lwct) {
    // 未建立连接的包，如果策略配置为放行则直接通过
    if (policy->params.ct_oot_action == LWFW_CT_OOT_ACTION_PASS)
      return ERR_OK;
  }
#endif

  // ===== 解析数据包为 lwfw_pkt_info_t =====
  lwfw_pkt_info_constructor(p, inp, &pkt_info, dir);

  // ===== 调用过滤引擎 (链表遍历或树搜索) =====
  ret = filter_engine->do_filter((void *)policy, (void *)&pkt_info, (void *)&ret_rule);

  // 处理事件上报
  if (ret_rule.action & LWFW_ACTION_CODE_EVENT) {
    lwfw_generate_secure_event(&ret_rule, p, &pkt_info, sizeof(pkt_info), ret_rule.hit_cnt);
  }

  return ret_rule.action;
}
```

### 4.3 `lwfw_pkt_info_constructor()` — 数据包解析

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:329-373`

```c
inline static void lwfw_pkt_info_constructor(const struct pbuf *p,
                                              const struct netif *inp,
                                              lwfw_pkt_info_t *pkt_info,
                                              lwfw_table_flag_t dir)
{
  const struct ip_hdr *ip_hdr;
  const void *trans_hdr;

  ip_hdr = (struct ip_hdr *)p->payload;
  trans_hdr = ((void *)p->payload) + IPH_HL_BYTES((struct ip_hdr *)p->payload);

#ifdef NIO_LWIP_LWCT
  pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);
#endif

  // 填充接口名
  if (inp) {
    if (inp->fullname) {
      memcpy(pkt_info->interface.if_name, inp->fullname, ...);
    } else {
      memcpy(pkt_info->interface.if_name, inp->name, sizeof(inp->name));
    }
  }

  if (dir == LWFW_IN_TABLE) {
    pkt_info->dir = LWFW_DIR_RX;
    // 入方向有 Ethernet 头
#ifdef LWFW_ADVANCED_FUNC_L2
    eth_hdr = (struct eth_hdr *)((uint8_t *)p + SIZEOF_STRUCT_PBUF);
    lwfw_pkt_l2_info_constructor(eth_hdr, dir, l2);
#endif
  } else {
    pkt_info->dir = LWFW_DIR_TX;
    // 出方向无 L2 (Ethernet 头尚未添加)
  }

  // 填充 L3 信息
  lwfw_pkt_l3_info_constructor(ip_hdr, l3);

  // 填充 L4 信息 (TCP/UDP 端口)
  lwfw_pkt_l4_info_constructor(trans_hdr, l3->proto, l4);
}
```

### 4.4 `list_search_do_filter()` — 链表遍历过滤引擎

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:1884-1978`

```c
static int list_search_do_filter(void *handle, void *data, void *result)
{
  lwfw_policy_t *policy = (lwfw_policy_t *)handle;
  lwfw_pkt_info_t *info = (lwfw_pkt_info_t *)data;
  match_result_t *ret_rule = (match_result_t *)result;
  lwfw_rule_table_t *curr_table;
  lwfw_rule_t *curr_rule = NULL;

  // 获取对应方向的规则表 (IN 或 OUT)
  curr_table = (lwfw_rule_table_t *)&(policy->rule_tables[info->dir]);

  // 无规则时使用默认动作
  if (curr_table->rule_cnt == 0)
    goto default_action;

  // 遍历规则链表
  cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE)
      continue;

    matched = check_rule(curr_rule, info, info->dir);

    if (matched)
      break;  // 首次匹配即停止 (First-Match)
  }

  if (matched && curr_rule != NULL) {
    // 规则匹配
    ret_rule->match_rule = curr_rule;
    ret_rule->rule_id = curr_rule->index;
    ret_rule->action = curr_rule->action;
    ret_rule->hit_cnt = ++curr_rule->hit_cnt;

    // 速率限制检查
    if (curr_rule->flags & LWFW_RULE_FLAGS_RATE_LIMIT) {
      __atomic_fetch_add(&curr_rule->rlimit.rx_pps, 1, __ATOMIC_RELAXED);

      if (curr_rule->rlimit.state == LWFW_RLIMIT_STATE_LIMIT) {
        // 已处于限速状态
        if (curr_rule->rlimit.rate != 0 &&
            curr_rule->rlimit.rx_pps > curr_rule->rlimit.rate &&
            !(curr_rule->action & LWFW_ACTION_CODE_DENY)) {
          // 超过速率限制则拒绝
          ret_rule->action |= LWFW_ACTION_CODE_DENY;
          curr_rule->rlimit.drops++;
        }
      } else if (curr_rule->rlimit.state != LWFW_RLIMIT_STATE_LIMIT &&
                 curr_rule->rlimit.rx_pps >= curr_rule->rlimit.burst) {
        // 从正常状态进入限速状态
        __atomic_store_n(&curr_rule->rlimit.state, LWFW_RLIMIT_STATE_LIMIT, __ATOMIC_RELAXED);
        curr_rule->rlimit.occurs++;
      }
    }
  } else {
default_action:
    // 无匹配，使用默认动作
    ret_rule->action = curr_table->def_action;
    ret_rule->hit_cnt = ++curr_table->def_hit_cnt;
  }

  return LWFW_ERR_OK;
}
```

### 4.5 `check_rule()` — 单条规则匹配

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:565-613`

```c
static bool check_rule(const lwfw_rule_t *rule, const lwfw_pkt_info_t *info,
                      lwfw_table_flag_t dir)
{
  bool matched = 1;

  // ===== 1. 连接状态匹配 =====
#ifdef NIO_LWIP_LWCT
  if ((rule->flags & LWFW_RULE_FLAGS_CT_STATE) &&
      rule->ct_state != info->ct_state) {
    return 0;
  }
#endif

  // ===== 2. 网卡接口匹配 =====
  if ((rule->flags & LWFW_RULE_FLAGS_NETIF) &&
      strncmp(rule->interface.if_name, info->interface.if_name,
              sizeof(info->interface.if_name) - 1) != 0) {
    return 0;
  }

  // ===== 3. L2 层匹配 =====
#ifdef LWFW_ADVANCED_FUNC_L2
  matched = check_lwfw_l2_info(rule, &info->l2);
  if (!matched) return 0;
#endif

  // ===== 4. L3 层匹配 =====
  matched = check_lwfw_l3_info(rule, &info->l3);
  if (!matched) return 0;

  // ===== 5. L4 层匹配 =====
  matched = check_lwfw_l4_info(rule, &info->l4);
  if (!matched) return 0;

  return matched;
}
```

### 4.6 `check_lwfw_l2_info()` — L2 字段匹配

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:383-437`

```c
inline static bool check_lwfw_l2_info(const lwfw_rule_t *rule,
                                      const lwfw_pkt_l2_info_t *packet_info)
{
  const lwfw_rule_l2_info_t *rule_l2_info = &rule->l2;
  bool matched = 0;

  // EtherType 匹配 (ETH_P_IP / ETH_P_ARP / ETH_P_ALL 等)
  if ((rule->flags & LWFW_RULE_FLAGS_ETHER_TYPE) &&
      rule_l2_info->ether_type != packet_info->ether_type) {
    goto out;
  }

  // VLAN ID 匹配
  if ((rule->flags & LWFW_RULE_FLAGS_VLAN) &&
      rule_l2_info->vlan != packet_info->vlan) {
    goto out;
  }

  // 源 MAC + 掩码匹配
  if (rule->flags & LWFW_RULE_FLAGS_SRC_MAC) {
    for (int i = 0; i < LWFW_ETH_HWADDR_LEN; i++) {
      if ((rule_l2_info->src_mac.mask[i] & packet_info->src_mac[i]) !=
          (rule_l2_info->src_mac.addr[i] & rule_l2_info->src_mac.mask[i]))
        goto out;
    }
  }

  // 目标 MAC + 掩码匹配
  if (rule->flags & LWFW_RULE_FLAGS_DST_MAC) {
    for (int i = 0; i < LWFW_ETH_HWADDR_LEN; i++) {
      if ((rule_l2_info->dst_mac.mask[i] & packet_info->dst_mac[i]) !=
          (rule_l2_info->dst_mac.addr[i] & rule_l2_info->dst_mac.mask[i]))
        goto out;
    }
  }

  matched = 1;
out:
  return matched;
}
```

### 4.7 `check_lwfw_l3_info()` — L3 字段匹配

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:447-488`

```c
inline static bool check_lwfw_l3_info(const lwfw_rule_t *rule,
                                      const lwfw_pkt_l3_info_t *packet_info)
{
  const lwfw_rule_l3_info_t *rule_l3_info = &rule->l3;
  uint32_t rule_mask = 0;
  bool matched = 0;

  // 协议匹配 (IP(0)/ICMP(1)/TCP(6)/UDP(17))
  if ((rule->flags & LWFW_RULE_FLAGS_PROTOCOL) &&
      rule_l3_info->proto != LWFW_RULE_PROTO_IP &&
      rule_l3_info->proto != packet_info->proto) {
    goto out;
  }

  // 源 IP + CIDR 掩码匹配
  if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK_LEN) {
    // 计算掩码: ~((1 << (32 - masklen)) - 1)
    rule_mask = (uint32_t)(~((1UL << (LWFW_IPV4_ADDR_BIT_SIZE - rule_l3_info->src_ip.masklen)) - 1));
    if ((packet_info->src_ip & rule_mask) != (rule_l3_info->src_ip.addr & rule_mask))
      goto out;
  }

  // 目标 IP + CIDR 掩码匹配
  if (rule->flags & LWFW_RULE_FLAGS_DST_IP_MASK_LEN) {
    rule_mask = (uint32_t)(~((1UL << (LWFW_IPV4_ADDR_BIT_SIZE - rule_l3_info->dst_ip.masklen)) - 1));
    if ((packet_info->dst_ip & rule_mask) != (rule_l3_info->dst_ip.addr & rule_mask))
      goto out;
  }

  matched = 1;
out:
  return matched;
}
```

### 4.8 `check_lwfw_l4_info()` — L4 字段匹配

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:498-543`

```c
inline static bool check_lwfw_l4_info(const lwfw_rule_t *rule,
                                      const lwfw_pkt_l4_info_t *packet_info)
{
  const lwfw_rule_l4_info_t *rule_l4_info = &rule->l4;
  bool src_matched = 0, dst_matched = 0;

  // 源端口范围 [begin, end]
  if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE) {
    src_matched = (packet_info->src_port >= rule_l4_info->src_ports.port_range[0] &&
                   packet_info->src_port <= rule_l4_info->src_ports.port_range[1]);
  }
  // 源端口列表 (最多 LWFW_MAX_PORT_COUNT=4 个端口)
  else if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_LIST) {
    for (int i = 0; i < LWFW_MAX_PORT_COUNT; i++) {
      if (rule_l4_info->src_ports.port_list[i] == 0) break;
      if (rule_l4_info->src_ports.port_list[i] == packet_info->src_port) {
        src_matched = 1; break;
      }
    }
  } else {
    src_matched = 1;  // 无源端口限制
  }

  // 目标端口范围 [begin, end]
  if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_RANGE) {
    dst_matched = (packet_info->dst_port >= rule_l4_info->dst_ports.port_range[0] &&
                   packet_info->dst_port <= rule_l4_info->dst_ports.port_range[1]);
  }
  // 目标端口列表
  else if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_LIST) {
    for (int i = 0; i < LWFW_MAX_PORT_COUNT; i++) {
      if (rule_l4_info->dst_ports.port_list[i] == 0) break;
      if (rule_l4_info->dst_ports.port_list[i] == packet_info->dst_port) {
        dst_matched = 1; break;
      }
    }
  } else {
    dst_matched = 1;  // 无目标端口限制
  }

  return src_matched && dst_matched;
}
```

### 4.9 `lwfw_generate_secure_event()` — 安全事件生成

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:641-708`

```c
static int lwfw_generate_secure_event(match_result_t *ret_rule,
                                       const struct pbuf *p,
                                       void *pkt_info,
                                       uint16_t len,
                                       uint32_t hit_cnt)
{
  // 事件限速检查
  if (lwfw_event_need_throttling())
    return ret;

  // FIFO 满检查
  if (lwfw_event_fifo_is_full()) {
    LWFW_STATICS_INC(g_lwfw_stats.drop_events);
    return LWFW_ERR_FIFO_FULL;
  }

  __atomic_fetch_add(&fw_ctrl_p->event_pps, 1, __ATOMIC_RELAXED);
  LWFW_STATICS_INC(g_lwfw_stats.total_event_cnt);

  lwfw_event_t event = {0};
  event.hdr.event_type = LOG_KIND_EVENT;

  // 填充事件数据
  event.data.rule_id = ret_rule->rule_id;
  event.data.action = ret_rule->action;
  strncpy(event.data.if_name, pkt->interface.if_name, ...);

  // 推送到事件 FIFO
  ret = lwfw_event_push(&event);

  return ret;
}
```

---

## 5. cBPF Socket 级过滤

### 5.1 cBPF 过滤函数链

```
raw_afpacket_input()                        [raw.c:282]
 └─ 遍历 raw_afpacket_pcbs 链表
     └─ lwip_run_socket_filter()            [sockets.c:580]
         └─ bpf_filter_run()               [bpf_filter.c:548]
             └─ cbpf_execute_interpreter() [bpf_filter.c:364]
                 └─ bpf_filter_with_aux_data()  [bpf_filter.c:68]
```

### 5.2 `raw_afpacket_input()` — AF_PACKET 分发

**文件**: `external/lwip_ds_mcu/src/core/raw.c:282-345`

```c
raw_input_state_t raw_afpacket_input(struct pbuf *p, struct netif *inp, u16_t type)
{
  struct raw_pcb *pcb, *prev;
  prev = NULL;
  pcb = raw_afpacket_pcbs;

  while (pcb != NULL) {
    // ===== 1. 协议匹配 =====
    switch (PP_NTOHS(pcb->protocol)) {
      case ETH_P_ALL:   skip_this_frame = 0; break;
      case ETH_P_IP:    skip_this_frame = (PP_HTONS(ETHTYPE_IP) != type); break;
      default:          skip_this_frame = 1; break;
    }

    // ===== 2. PCB 状态检查 =====
    if (pcb->state == AF_PACKET_STATE_INIT)
      skip_this_frame = 1;

    // ===== 3. 网卡绑定检查 =====
    if ((pcb->netif_idx != AF_PACKET_NOBIND) && (pcb->netif_idx != netif_get_index(inp)))
      skip_this_frame = 1;

    // ===== 4. cBPF 过滤器匹配 =====
    if (!skip_this_frame && pcb->recv != NULL &&
        lwip_run_socket_filter(pcb->conn, p, inp) != 0)  // 返回 1=匹配
    {
      // 匹配成功，调用 recv 回调 (tpacket_recv)
      ret = RAW_INPUT_DELIVERED;
      pcb->recv(pcb->recv_arg, pcb, p, NULL);
    }

next_pcb_output:
    prev = pcb;
    pcb = pcb->next;
  }
  return ret;
}
```

### 5.3 `lwip_run_socket_filter()` — Socket 过滤入口

**文件**: `external/lwip_ds_mcu/src/api/sockets.c:580-625`

```c
int lwip_run_socket_filter(void *conn, struct pbuf *p, struct netif* inp)
{
  if (conn == NULL)
    return 0;  // 不捕获

  struct lwip_sock *sock = tryget_socket_unconn_nouse(((struct netconn *)conn)->socket);

  if (sock != NULL) {
    // 检查接口过滤位图
    if (GET_INF_FILTER(sock->inf_filter, netif_get_index(inp))) {
      ret = bpf_filter_run(sock->bpf_prog, p);
    }
  }

  return ret;  // 1=匹配(捕获), 0=不匹配(跳过)
}
```

### 5.4 `bpf_filter_run()` — cBPF 执行

**文件**: `external/lwip_ds_mcu/src/core/bpf/bpf_filter.c:548-562`

```c
int bpf_filter_run(struct bpf_program *fprog, struct pbuf *pb)
{
  unsigned int pkt_len;

  if (fprog == NULL)
    return 1;  // 无过滤器，接受所有
  if (pb == NULL)
    return 0;

  pkt_len = cbpf_execute_interpreter(fprog, pb);

  return pkt_len ? 1 : 0;  // 1=匹配(捕获), 0=不匹配(跳过)
}
```

### 5.5 `bpf_filter_with_aux_data()` — cBPF 解释器核心

**文件**: `external/lwip_ds_mcu/src/core/bpf/bpf_filter.c:68-362`

完整 cBPF 解释器，支持所有标准指令:

| 指令类别 | 支持的操作 |
|---------|-----------|
| `BPF_LD` / `BPF_LDX` | `W`, `H`, `B`, `MEM` (加载数据包字段到 A/X 寄存器) |
| `BPF_ST` / `BPF_STX` | 存储到 memory word |
| `BPF_ALU` | `ADD`, `SUB`, `MUL`, `DIV`, `MOD`, `AND`, `OR`, `XOR`, `LSH`, `RSH` |
| `BPF_JMP` | `JEQ`, `JNE`, `JGT`, `JGE`, `JLT`, `JLE`, `JSET`, `JA` |
| `BPF_RET` | 返回接受/拒绝 |
| `BPF_MISC` | 寄存器移动 (TAX, TXA) |

### 5.6 `bpf_filter_verify()` — cBPF 程序校验

**文件**: `external/lwip_ds_mcu/src/core/bpf/bpf_filter_linux.c:571`

确保 cBPF 程序:
- 跳转目标有效 (前向跳转且在程序范围内)
- 内存访问在包范围内 (静态检查)
- 以 `RET` 指令终止
- 不包含无效指令编码

### 5.7 `SO_ATTACH_FILTER` setsockopt

**文件**: `external/lwip_ds_mcu/src/api/sockets.c:4434-4460`

```c
case SO_ATTACH_FILTER: {
  LWIP_SOCKOPT_CHECK_OPTLEN_CONN(sock, optlen, int);

  // 仅支持 RAW 和 PACKET 类型 socket
  if (NETCONNTYPE_GROUP(sock->conn->type) != NETCONN_RAW &&
      NETCONNTYPE_GROUP(sock->conn->type) != NETCONN_PACKET)
    return ENOPROTOOPT;

  struct bpf_insn *prog_insns = (struct bpf_insn *)optval;
  int prog_size = optlen / sizeof(struct bpf_insn);

  // 校验 cBPF 程序
  if (bpf_filter_verify(prog_insns, prog_size) != 1)
    return EINVAL;

  // 释放旧过滤器
  if (sock->bpf_prog != NULL)
    socket_bpf_release_filter(sock);

  // 创建并附加新过滤器
  err = socket_bpf_create_filter(sock, prog_size, prog_insns);
  return err;
}
```

---

## 6. 数据结构

### 6.1 `lwfw_rule` — 防火墙规则

**文件**: `libs/util_libs/liblwfw/include/lwfw.h:150`

```c
struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_rule {
  struct cdlist next;                  // 链表节点
  uint16_t index;                      // 规则唯一索引
  uint16_t priority;                   // 优先级 (未使用)
  uint16_t state;                     // LWFW_STATE_ENABLE / DISABLE
  uint16_t ct_state;                  // 连接跟踪状态
  uint32_t flags;                      // 匹配标志位图
  char rule_name[MAX_RULE_NAME_LEN];  // 规则名称

  lwfw_netif_t interface;             // 网卡接口名
  lwfw_rule_l2_info_t l2;            // L2: EtherType, VLAN, MAC
  lwfw_rule_l3_info_t l3;            // L3: src/dst IP, Protocol
  lwfw_rule_l4_info_t l4;            // L4: src/dst Port

  lwfw_action_t action;                // 动作: DENY / EVENT / LOGGING
  rate_limit_t rlimit;               // 速率限制
  uint32_t hit_cnt;                   // 命中计数
};
```

### 6.2 `lwfw_pkt_info_t` — 数据包信息

**文件**: `libs/util_libs/liblwfw/include/lwfw_common.h:351`

```c
typedef struct lwfw_pkt_info {
  lwfw_netif_t interface;              // 接口名
  lwfw_pkt_l2_info_t l2;            // L2: EtherType, VLAN, MAC
  lwfw_pkt_l3_info_t l3;             // L3: src_ip, dst_ip, proto
  lwfw_pkt_l4_info_t l4;            // L4: src_port, dst_port
  lwfw_direction_t dir;              // LWFW_DIR_RX / LWFW_DIR_TX
#ifdef NIO_LWIP_LWCT
  uint16_t ct_state;                 // 连接跟踪状态
#endif
} lwfw_pkt_info_t;
```

### 6.3 `lwfw_rule_table_t` — 规则表

**文件**: `libs/util_libs/liblwfw/include/lwfw.h:174`

```c
typedef struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_rule_table {
  uint16_t rule_cnt;                  // 规则数量
  uint16_t state;                    // LWFW_STATE_ENABLE / DISABLE
  lwfw_action_t def_action;           // 默认动作 (无匹配时)
  uint32_t def_hit_cnt;              // 默认动作命中计数
  struct cdlist header;              // 规则链表头
  rule_set_t _ruleset;               // 预留 (树搜索)
  hs_tree_t _hs_tree;               // 预留 (Hyperscan)
} lwfw_rule_table_t;
```

### 6.4 `lwfw_policy_t` — 防火墙策略

**文件**: `libs/util_libs/liblwfw/include/lwfw.h`

```c
typedef struct lwfw_policy {
  lwfw_rule_table_t rule_tables[LWFW_MAX_COUNT_TABLE];  // IN + OUT 两张表
  const lwfw_backend_engine_t *filter_engine;            // 过滤引擎
  lwfw_policy_params_t params;                           // 策略参数
} lwfw_policy_t;
```

### 6.5 `lwip_sock` — Socket + BPF

**文件**: `external/lwip_ds_mcu/src/include/lwip/priv/sockets_priv.h:69`

```c
struct lwip_sock {
  struct netconn *conn;
  union lwip_sock_lastdata lastdata;
#if LWIP_SOCKET_SELECT || LWIP_SOCKET_POLL
  s16_t rcvevent;
  u16_t sendevent;
  u16_t errevent;
  SELWAIT_T select_waiting;
#endif
  /* cBPF 过滤器程序 */
  struct bpf_program *bpf_prog;
  /* 接口过滤位图 (bit N = 1 表示监听接口 N) */
  u16_t inf_filter;
  /* PACKET_MMAP 元数据 */
  void *packet_info;
};
```

### 6.6 匹配标志位定义

**文件**: `libs/util_libs/liblwfw/include/lwfw_common.h` + `lwfw.h`

```c
// L2
#define LWFW_RULE_FLAGS_ETHER_TYPE       BIT(0)
#define LWFW_RULE_FLAGS_VLAN             BIT(1)
#define LWFW_RULE_FLAGS_SRC_MAC          BIT(2)
#define LWFW_RULE_FLAGS_DST_MAC          BIT(3)

// L3
#define LWFW_RULE_FLAGS_PROTOCOL         BIT(4)
#define LWFW_RULE_FLAGS_SRC_IP_MASK_LEN  BIT(5)  // CIDR 格式
#define LWFW_RULE_FLAGS_DST_IP_MASK_LEN  BIT(6)
#define LWFW_RULE_FLAGS_SRC_IP_MASK      BIT(5)  // 旧格式
#define LWFW_RULE_FLAGS_DST_IP_MASK      BIT(6)

// L4
#define LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE BIT(7)
#define LWFW_RULE_FLAGS_SRC_L4_PORT_LIST  BIT(8)
#define LWFW_RULE_FLAGS_DST_L4_PORT_RANGE BIT(9)
#define LWFW_RULE_FLAGS_DST_L4_PORT_LIST  BIT(10)

// 其他
#define LWFW_RULE_FLAGS_NETIF             BIT(11)
#define LWFW_RULE_FLAGS_CT_STATE         BIT(12)
#define LWFW_RULE_FLAGS_RATE_LIMIT       BIT(13)
```

### 6.7 动作定义

**文件**: `libs/util_libs/liblwfw/include/lwfw_external.h:38`

```c
typedef enum lwfw_action {
  LWFW_ACTION_CODE_DENY     = BIT(0),  // 拒绝
  LWFW_ACTION_CODE_EVENT    = BIT(1),  // 生成安全事件
  LWFW_ACTION_CODE_LOGGING  = BIT(2),  // 记录日志
} lwfw_action_t;

// 支持组合: LWFW_ACTION_CODE_DENY | LWFW_ACTION_CODE_EVENT
```

### 6.8 规则动作优先级

1. **DENY** 优先 — 即使规则同时设置了 EVENT，DENY 也会先执行丢弃
2. **EVENT** 次之 — 匹配规则且未拒绝时，生成安全事件
3. **LOGGING** 最低 — 仅记录日志，不影响数据包

---

## 7. lwfw_ops 函数指针表

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:1996-2000`

```c
static const lwfw_firewall_ops_t lwfw_ops = {
  .firewall_ioctl = lwfw_firewall_ioctl,                    // IOCTL 控制
  .ingress_filter = ip4_filter_dispatch_incoming,           // RX 过滤
  .egress_filter = ip4_filter_dispatch_outgoing,           // TX 过滤
};
```

全局变量:

```c
lwfw_firewall_t g_lwfw_firewall, *lwfw_p;  // [lwfw.c:36]
lwfw_policy_t lwfw_policy;                  // 当前活跃策略
lwfw_policy_t lwfw_policy_swap;             // 热切换备份策略
```

---

## 8. 两种过滤引擎

### 8.1 链表搜索引擎 (默认)

**文件**: `libs/util_libs/liblwfw/src/lwfw.c:1884-1994`

```c
static const lwfw_backend_engine_t list_search_eng = {
  .name = "list search",
  .init = list_search_init,
  .deinit = list_search_deinit,
  .do_filter = list_search_do_filter,  // 逐条遍历
  .dump = list_search_dump,
};
```

特点:
- 无索引，按顺序遍历所有规则
- 时间复杂度 O(n)
- 首次匹配即停止

### 8.2 树搜索引擎 (Hyperscan-like)

**文件**: `libs/util_libs/liblwfw/src/tree_hs.c`

当 `LWFW_FILTER_TREE` 启用时使用，支持:
- 多维匹配 (L2+L3+L4 同时索引)
- 批量规则预编译
- 热切换时原子替换策略

---

## 9. 过滤决策流程图

```
数据包
  │
  ├─► [lwfw IN/OUT_TABLE 启用?] ─否─► 直接放行
  │                              │
  │                              ▼
  │                   lwfw_pkt_info_constructor()
  │                   (解析 pbuf → lwfw_pkt_info_t)
  │                              │
  │                              ▼
  │                filter_engine->do_filter()
  │                ├─ 遍历规则链表 (list_search_do_filter)
  │                └─ check_rule() 逐字段匹配
  │                              │
  │         ┌────────────────────┴────────────────────┐
  │         ▼                                         ▼
  │   [有规则匹配]                              [无匹配]
  │   ├─ action = rule.action                     def_action
  │   ├─ hit_cnt++                                (默认动作)
  │   └─ 速率限制检查                                │
  │         │                                        │
  │         ▼                                        ▼
  │   [action & DENY?]                        放行 / 拒绝
  │   ├─ 是 → 丢弃 + total_rx_drop++            (取决于默认动作)
  │   └─ 否 → 放行
  │         │
  │         ├─► [action & EVENT?] → 推送到 lwfw_event_fifo
  │         └─► [action & LOGGING?] → 记录日志
  │
  ├─► [cBPF attached?] ─否─► 不执行 cBPF
  │         │
  │         ▼
  │  bpf_filter_run()
  │  ├─ cBPF 返回 0 → 不捕获
  │  └─ cBPF 返回 >0 → 捕获
  │
  └─► raw_afpacket_input() → 分发到 AF_PACKET socket 回调
```

---

## 10. lwfw_agent 事件处理

**文件**: `os-framework/servers/daemons/lwfw_agent/src/main.c`

lwfw 生成的安全事件通过共享内存 FIFO 传递给 lwfw_agent 守护进程处理:

```
lwfw_event_fifo (共享内存)
  ↓ (LWFW_EVENT_NUM = 512 个事件槽)
lwfw_agent
  ├─ 读取事件
  ├─ JSON 格式化
  ├─ 写入日志文件 (/var/log/lwfw/events_*.log)
  └─ 可选: 上报到云端
```

---

## 11. 全局统计

**文件**: `libs/util_libs/liblwfw/include/lwfw_stats.h`

```c
struct lwfw_stats {
  uint64_t total_rx_drop;           // RX 丢弃总数
  uint64_t total_tx_drop;           // TX 丢弃总数
  uint64_t total_event_cnt;         // 安全事件总数
  uint64_t drop_events;             // 因 FIFO 满丢弃的事件
  uint64_t throttled_events;       // 限速丢弃的事件
  uint64_t throttled_logs;         // 限速丢弃的日志
  uint64_t err_log_cnt;
  uint64_t warn_log_cnt;
  uint64_t ct_notrack;             // 未跟踪的包数 (兜底)
  // ...
};
```

---

## 12. 关键文件清单

| 层级 | 文件路径 | 职责 |
|------|----------|------|
| **Hook 注入** | `external/lwip_ds_mcu/src/core/ipv4/ip4.c:743,1096` | 防火墙 hook 调用点 |
| **防火墙核心** | `libs/util_libs/liblwfw/src/lwfw.c` | ip4_filter, check_rule, list_search_do_filter |
| **规则头文件** | `libs/util_libs/liblwfw/include/lwfw.h` | lwfw_rule, lwfw_policy, lwfw_rule_table 结构体 |
| **通用定义** | `libs/util_libs/liblwfw/include/lwfw_common.h` | lwfw_pkt_info_t, 动作/标志枚举 |
| **外部 API** | `libs/util_libs/liblwfw/include/lwfw_external.h` | 动作枚举, 日志级别 |
| **规则解析** | `libs/util_libs/liblwfw/src/lwfw_parser.c` | YAML 规则解析 |
| **事件通知** | `libs/util_libs/liblwfw/src/lwfw_notif.c` | lwfw_event_fifo 实现 |
| **树搜索** | `libs/util_libs/liblwfw/src/tree_hs.c` | Hyperscan 风格多维匹配 |
| **统计** | `libs/util_libs/liblwfw/include/lwfw_stats.h` | 防火墙统计结构体 |
| **cBPF 解释器** | `external/lwip_ds_mcu/src/core/bpf/bpf_filter.c` | bpf_filter_run, bpf_filter_with_aux_data |
| **cBPF 校验器** | `external/lwip_ds_mcu/src/core/bpf/bpf_filter_linux.c` | bpf_filter_verify |
| **Socket Filter** | `external/lwip_ds_mcu/src/core/bpf/socket_filter.c` | socket_bpf_create_filter |
| **AF_PACKET** | `external/lwip_ds_mcu/src/core/raw.c:282` | raw_afpacket_input, cBPF 调用点 |
| **Socket API** | `external/lwip_ds_mcu/src/api/sockets.c:580,4434` | lwip_run_socket_filter, SO_ATTACH_FILTER |
| **NSv 配置** | `os-framework/servers/net/src/lwfwcfg.c` | 防火墙 CLI 配置接口 |
| **lwfw_agent** | `os-framework/servers/daemons/lwfw_agent/` | 安全事件处理守护进程 |

---

## 13. 性能特性

### 13.1 延迟测试

当 `LWFW_TEST_LATENCY` 启用时，每个数据包过滤路径会测量:

```c
// ip4_input() 中
t_start = raw_read_pcnt_el0();  // ARM 性能计数器
lwfw_p->ops->ingress_filter(p, inp);
t_end = raw_read_pcnt_el0();
delta_ns = (t_end - t_start) * 1000000000 / freq;
```

典型测量结果:
- `ip4_filter_dispatch_incoming`: ~数微秒 (取决于规则数量)
- `check_rule` 单次匹配: ~数十纳秒

### 13.2 速率限制

基于 token bucket 算法:
- `rlimit.burst` — 桶容量
- `rlimit.rate` — 速率 (packets per second)
- `rlimit.state` — NORMAL / LIMIT 状态转换

### 13.3 事件限速

```c
if (policy_p->params.event_rlimit_rate != 0 &&
    fw_ctrl_p->event_pps >= policy_p->params.event_rlimit_rate) {
  // 丢弃事件
}
```

---

## 14. 与 Linux netfilter 的关键差异

| 方面 | Linux netfilter (iptables) | SafeOS lwfw |
|------|---------------------------|-------------|
| 架构 | 内核态 hook (NF_HOOK 宏) | 用户态 lwIP 函数调用 |
| 规则组织 | 链表 / ipset / nftable | 单向链表 (list) 或树 (tree) |
| 连接跟踪 | 内核 conntrack | 用户态 lwct (可选) |
| 匹配字段 | 完整 L1-L4 + mark/connmark | L2(Lwfw ADVANCED) + L3 + L4 |
| 动作 | ACCEPT / DROP / REJECT / LOG / ... | DENY / EVENT / LOGGING |
| 性能计数器 | 内核原子操作 | __atomic_fetch_add |
| 热更新 | atomic replacement | 切换 lwfw_policy / lwfw_policy_swap |
