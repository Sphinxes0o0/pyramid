---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW List Search

## 定义

`list_search` 是 LWFW 的**默认过滤引擎**，使用 cdlist 链表存储规则，按插入顺序线性遍历，首匹配即停止。适用于规则数量少 (<20) 的场景。

## 引擎注册

```c
const lwfw_backend_engine_t list_search_eng = {
    .name = "list search",
    .init = list_search_init,
    .deinit = list_search_deinit,
    .do_filter = list_search_do_filter,
    .dump = list_search_dump,
};
```

## do_filter 实现

```c
static int list_search_do_filter(void *handle, void *data, void *result)
{
  curr_table = &policy->rule_tables[info->dir];

  if (curr_table->rule_cnt == 0)
    goto default_action;

  // 遍历 cdlist 链表
  cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE)
      continue;

    matched = check_rule(curr_rule, info, info->dir);
    if (matched)
      break;  // 首次匹配即停止
  }

  if (matched && curr_rule != NULL) {
    ret_rule->action = curr_rule->action;
    ret_rule->hit_cnt = ++curr_rule->hit_cnt;

    // 速率限制检查
    if (flags & RATE_LIMIT) {
      __atomic_fetch_add(&rlimit.rx_pps, 1, __ATOMIC_RELAXED);
      if (rlimit.state == LIMIT && rx_pps > rate)
        action |= DENY;
    }
  } else {
default_action:
    action = curr_table->def_action;
  }
}
```

## 限速状态机

```
NORMAL ──(rx_pps >= burst)──► LIMIT ──(time >= expire)──► NORMAL

LIMIT 状态下:
- 如果 action 不是 DENY 且 rx_pps > rate → 拒绝
- EDGE 模式只在进入限速时上报一次 event
```

## 预取优化 (LWFW_PREFETCH)

```c
#if LWFW_PREFETCH
  cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE)
      continue;

    // 预取下一条规则到 L1 cache
    if (curr_rule->next != header) {
      prefetch_addr_L1((u64_t)((uint8_t *)curr_rule + RULE_ELE_SIZE));
    }

    matched = check_rule(curr_rule, info, info->dir);
    if (matched) break;
  }
#endif
```

## 性能分析

| 规则数 | 平均比较次数 | 最坏情况 |
|--------|------------|---------|
| 10 | 5 | 10 |
| 20 | 10 | 20 |
| 100 | 50 | 100 |

## 与 tree_search 对比

| 特性 | list_search | tree_search |
|------|------------|-------------|
| 数据结构 | cdlist 链表 | hyperscan 决策树 |
| 搜索复杂度 | O(n) | O(log n) 最好，O(n) 最坏 |
| L2 支持 | 完整 | 不支持 |
| 内存开销 | 低 | 高 |

## 已知问题

| 问题 | 说明 |
|------|------|
| 无优先级排序 | 规则按插入顺序存储，高特异性规则可能在链表尾部 |
| 禁用规则占位置 | `state == DISABLE` 的规则仍需遍历跳过 |

## 相关概念

- [[entities/linux/lwfw/lwfw-core-filtering]] — 核心过滤逻辑
- [[entities/linux/lwfw/lwfw-tree-search]] — 树搜索引擎
- [[entities/linux/lwfw/lwfw-rule-matching]] — 规则匹配算法优化
