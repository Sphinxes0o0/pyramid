---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Data Structures

## 定义

LWFW 防火墙策略数据结构，包含规则 (rule)、策略 (policy)、包信息 (pkt_info) 和事件 FIFO 等核心结构，设计上支持缓存行对齐优化和双缓冲热切换。

## 内存布局与缓存对齐

```c
struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_rule {
  struct cdlist next;         // 链表节点
  uint16_t index;            // 规则索引
  uint16_t priority;         // 优先级
  uint16_t state;            // 启用/禁用
  uint16_t ct_state;         // 连接跟踪状态
  uint32_t flags;            // 匹配标志位图
  char rule_name[32];        // 规则名称
  lwfw_netif_t interface;    // 接口名 (32 字节)
  lwfw_rule_l2_info_t l2;   // L2 匹配字段
  lwfw_rule_l3_info_t l3;   // L3 匹配字段
  lwfw_rule_l4_info_t l4;   // L4 匹配字段
  lwfw_action_t action;      // 动作
  rate_limit_t rlimit;       // 速率限制
  uint32_t hit_cnt;         // 命中计数
};
```

`CACHE_ALIGNMENT` 通常为 64 字节 (L1 cache line)，确保规则遍历时减少跨缓存行访问。

## 热切换机制

```c
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };

// 切换流程
lwfw_config_reset_state()
  ├─ sync_mutex_lock(&policy_lock)
  ├─ lwfw_policy_clean(inactive_policy);
  ├─ lwfw_copy_policy(inactive_policy, policy);  // 深拷贝
  ├─ inactive_policy->filter_engine->init();
  ├─ policy <-> inactive_policy 交换指针
  └─ sync_mutex_unlock(&policy_lock)
```

## 匹配标志位图

```c
// L2
BIT(0)  LWFW_RULE_FLAGS_NETIF
BIT(1)  LWFW_RULE_FLAGS_SRC_MAC
BIT(2)  LWFW_RULE_FLAGS_DST_MAC
BIT(3)  LWFW_RULE_FLAGS_VLAN
BIT(4)  LWFW_RULE_FLAGS_ETHER_TYPE

// L3
BIT(5)  LWFW_RULE_FLAGS_PROTOCOL
BIT(6)  LWFW_RULE_FLAGS_SRC_IP_MASK
BIT(7)  LWFW_RULE_FLAGS_SRC_IP_MASK_LEN  // ← 与 SRC_L4_PORT_RANGE 冲突！
BIT(8)  LWFW_RULE_FLAGS_DST_IP_MASK
BIT(9)  LWFW_RULE_FLAGS_DST_IP_MASK_LEN

// L4
BIT(10) LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE
BIT(11) LWFW_RULE_FLAGS_SRC_L4_PORT_LIST
BIT(12) LWFW_RULE_FLAGS_DST_L4_PORT_RANGE
BIT(13) LWFW_RULE_FLAGS_DST_L4_PORT_LIST

// 其他
BIT(14) LWFW_RULE_FLAGS_CT_STATE
BIT(15) LWFW_RULE_FLAGS_RATE_LIMIT
```

## 事件 FIFO

```c
struct lwfw_event_fifo {
  sync_mutex_t prod_lock;         // 多生产者锁
  lwfw_agent_parameters_t params; // 配置参数
  volatile uint32_t get_idx;     // 消费者索引
  volatile uint32_t put_idx;     // 生产者索引
  uint32_t queue_size;          // 事件槽数量 (512)
  lwfw_event events[1];          // 变长数组
};
```

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| 位标志冲突 | P0 | `SRC_IP_MASK_LEN` (BIT 7) 与 `SRC_L4_PORT_RANGE` 冲突 |
| copy_offset 技巧 | P1 | 使用空结构体计算偏移量，缺文档 |
| 双内存池占用 | P2 | 两套 MEMP_LWFW_RULE* 规则数量多时内存翻倍 |

## 相关概念

- [[entities/linux/lwfw/lwfw-architecture]] — 整体架构
- [[entities/linux/lwfw/lwfw-core-filtering]] — 过滤逻辑
- [[entities/linux/lwfw/lwfw-hotswap-analysis]] — 热切换深度分析
