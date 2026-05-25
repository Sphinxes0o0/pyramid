# list_search 线性扫描引擎分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw.c`

---

## 1. 引擎注册

```c
// lwfw.c:1872-1882
static int list_search_init(void *handle, void *data)
{
  // Nothing to do currently
  return ERR_OK;
}

static int list_search_deinit(void *handle, void *data)
{
  // Nothing to do currently
  return ERR_OK;
}
```

引擎定义:
```c
// lwfw.c: ???
const lwfw_backend_engine_t list_search_eng = {
    .name = "list search",
    .init = list_search_init,
    .deinit = list_search_deinit,
    .do_filter = list_search_do_filter,
    .dump = list_search_dump,
};
```

---

## 2. list_search_do_filter 实现

```c
// lwfw.c:1884-1978
static int list_search_do_filter(void *handle, void *data, void *result)
{
  int ret = LWFW_ERR_OK;
  bool matched = 0;
  uint16_t rule_id = LWFW_INVALID_RULE_INDEX;
  lwfw_policy_t *policy = (lwfw_policy_t *)handle;
  lwfw_pkt_info_t *info = (lwfw_pkt_info_t *)data;
  match_result_t *ret_rule = (match_result_t *)result;
  const struct cdlist *header;
  lwfw_rule_table_t *curr_table;
  lwfw_rule_t *curr_rule = NULL;

  // 1. 获取当前方向的规则表
  curr_table = (lwfw_rule_table_t *)&(policy->rule_tables[info->dir]);

  // 2. 规则数为0，使用默认动作
  if (curr_table->rule_cnt == 0)
    goto default_action;

  header = &curr_table->header;

  // 3. 遍历 cdlist 链表
  cdlist_iter_entry(curr_rule, header, next) {
    // 跳过禁用的规则
    if (curr_rule->state == LWFW_STATE_DISABLE)
      continue;

    // 调用 check_rule 进行匹配
    matched = check_rule(curr_rule, info, info->dir);

    if (matched) // 首次匹配即停止
      break;
  }

  // 4. 处理匹配结果
  if (matched && curr_rule != NULL) {
    ret_rule->match_rule = curr_rule;
    ret_rule->rule_id = curr_rule->index;
    ret_rule->action = curr_rule->action;
    ret_rule->hit_cnt = ++curr_rule->hit_cnt;

    // 5. 限速处理
    if (curr_rule->flags & LWFW_RULE_FLAGS_RATE_LIMIT) {
      __atomic_fetch_add(&curr_rule->rlimit.rx_pps, 1, __ATOMIC_RELAXED);

      if (curr_rule->rlimit.state != LWFW_RLIMIT_STATE_LIMIT &&
          curr_rule->rlimit.rx_pps >= curr_rule->rlimit.burst) {
        // 进入限速状态
        __atomic_store_n(&curr_rule->rlimit.state,
                        LWFW_RLIMIT_STATE_LIMIT, __ATOMIC_RELAXED);
        curr_rule->rlimit.occurs++;
      } else if (curr_rule->rlimit.state == LWFW_RLIMIT_STATE_LIMIT) {
        // 已在限速状态，检查是否需要丢包
        if (curr_rule->rlimit.rate != 0 &&
            ((curr_rule->action & LWFW_ACTION_CODE_DENY) != LWFW_ACTION_CODE_DENY) &&
            curr_rule->rlimit.rx_pps > curr_rule->rlimit.rate) {
          ret_rule->action |= LWFW_ACTION_CODE_DENY;
          curr_rule->rlimit.drops++;
        }
        // EDGE 模式只在进入限速时上报一次
        if (curr_rule->rlimit.event_mode == LWFW_RLIMIT_EVENT_RAISE_MODE_EDGE)
          ret_rule->action &= ~LWFW_ACTION_CODE_EVENT;
      } else {
        // 正常状态，不上报事件
        ret_rule->action &= ~LWFW_ACTION_CODE_EVENT;
      }
    }
  } else {
    // 无匹配，使用默认动作
default_action:
    ret_rule->action = curr_table->def_action;
    ret_rule->hit_cnt = ++curr_table->def_hit_cnt;
  }

  return ret;
}
```

---

## 3. 遍历机制 cdlist

### 3.1 cdlist 链表结构

```c
// 链表头
struct cdlist {
  struct cdlist *next;
  struct cdlist *prev;
};

// 规则中的链表节点
struct cdlist next;
```

### 3.2 遍历宏

```c
// 遍历宏
cdlist_iter_entry(curr, head, member) {
  for (curr = (head)->next; curr != (head); curr = curr->next)
}

// 安全遍历 (用于删除)
cdlist_iter_entry_safe(curr, tmp, head, member) {
  for (curr = (head)->next, tmp = curr->next;
       curr != (head);
       curr = tmp, tmp = curr->next)
}
```

---

## 4. 预取优化 (LWFW_PREFETCH)

```c
#if LWFW_PREFETCH
  lwfw_rule_t *next_rule = NULL;
// ...
  cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE)
      continue;

    // 预取下一条规则
    if (curr_rule->next != header) {
      prefetch_addr_L1((u64_t)((uint8_t *)curr_rule + RULE_ELE_SIZE));
    }

    matched = check_rule(curr_rule, info, info->dir);
    if (matched)
      break;
  }
#endif
```

---

## 5. 限速状态机

```
                              ┌─────────────────┐
                              │     NORMAL      │
                              │ (初始状态)       │
                              └────────┬────────┘
                                       │
                    rx_pps >= burst    │ (首次超过桶容量)
                                       ▼
                              ┌─────────────────┐
                              │     LIMIT       │
         rx_pps > rate        │ (限速中)         │
    ┌─────────────────────────►│ (可配置 expire)  │
    │                          └────────┬────────┘
    │                                   │
    │           rate=0 或               │
    │    (event_mode == EDGE)           │ expire 时间到达
    │                                   │
    └───────────────────────────────────┘
                              (回到 NORMAL)
```

### 5.1 限速参数

| 参数 | 说明 |
|------|------|
| `burst` | 桶容量 (pps) |
| `rate` | 限速阈值 (pps) |
| `expire` | 限速持续时间 (秒) |
| `event_mode` | EDGE (仅首次) / LEVEL (持续) |

---

## 6. 性能分析

### 6.1 时间复杂度

| 规则数 | 平均比较次数 | 最坏情况 |
|--------|------------|---------|
| 10 | 5 | 10 |
| 20 | 10 | 20 |
| 100 | 50 | 100 |

### 6.2 空间复杂度

```
O(n)  - 每条规则一个链表节点
```

### 6.3 缓存友好性

- 规则按 `CACHE_ALIGNMENT` (64字节) 对齐
- 预取优化减少缓存未命中
- 链表顺序遍历，连续内存访问

---

## 7. 引擎选择逻辑

```c
// lwfw.c:lwfw_engine_init()
if (rule_cnt >= 20 && LWFW_TREE_SEARCH_EN) {
    policy->filter_engine = &tree_search_eng;
} else {
    policy->filter_engine = &list_search_eng;
}
```

- 规则数 < 20: 使用 list_search
- 规则数 ≥ 20: 使用 tree_search (需启用 LWFW_TREE_SEARCH_EN)

---

## 8. 与 tree_search 对比

| 特性 | list_search | tree_search |
|------|------------|-------------|
| 数据结构 | cdlist 链表 | hyperscan 决策树 |
| 搜索复杂度 | O(n) | O(log n) 最好，O(n) 最坏 |
| 实现复杂度 | 简单 | 复杂 |
| 规则维度 | 全部字段 | 仅 L3/L4 (5维) |
| L2 支持 | 完整 (需编译选项) | 不支持 |
| 内存开销 | 低 | 高 (树节点) |

---

## 9. 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `list_search_do_filter` | lwfw.c | 1884 |
| `list_search_init` | lwfw.c | 1872 |
| `list_search_deinit` | lwfw.c | 1878 |
| `check_rule` | lwfw.c | 565 |

---

## 10. 已知问题

### 10.1 无优先级排序

规则按插入顺序存储，没有按优先级排序。如果高特异性规则插入在链表尾部，会遍历大量不相关规则。

### 10.2 禁用规则仍占用链表位置

```c
if (curr_rule->state == LWFW_STATE_DISABLE)
  continue;
```

禁用规则仍在链表中，遍历时需要跳过，增加比较次数。

### 10.3 全局状态访问

```c
curr_table = (lwfw_rule_table_t *)&(policy->rule_tables[info->dir]);
```

需要根据方向索引获取规则表，有一次内存访问。
