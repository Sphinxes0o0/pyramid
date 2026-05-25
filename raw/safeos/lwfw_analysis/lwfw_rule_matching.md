# 规则匹配算法优化空间深度分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw.c`
> 树搜索: `libs/util_libs/liblwfw/src/tree_hs.c`, `tree_entry.c`

---

## 1. 当前匹配架构

### 1.1 两套过滤引擎

| 引擎 | 模式 | 数据结构 | 适用场景 |
|------|------|---------|---------|
| `list_search_eng` | 线性扫描 | cdlist 链表 | 规则少 (<20) |
| `tree_search_eng` | 决策树 | hyperscan 树 | 规则多 (>20) |

### 1.2 匹配流程

```
数据包
  │
  ▼
ip4_filter()
  │
  ├─ lwfw_pkt_info_constructor()  ← 解析包信息
  │
  └─ filter_engine->do_filter()
        │
        ├─ list_search_do_filter()  ← 线性扫描
        │     └─ 遍历 cdlist，对每条规则调用 check_rule()
        │
        └─ tree_search_do_filter() ← 树搜索
              └─ hs_lookup_entry() → 定位叶子 → linear_search_entry()
```

---

## 2. list_search 线性扫描

### 2.1 遍历代码

```c
// lwfw.c:list_search_do_filter()
cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE)
        continue;

    matched = check_rule(curr_rule, info, info->dir);
    if (matched)
        break;  // 首次匹配即停止
}
```

### 2.2 check_rule 匹配顺序

```c
// lwfw.c:check_rule()

// 1. 连接状态匹配 (可选)
if (rule->flags & LWFW_RULE_FLAGS_CT_STATE &&
    rule->ct_state != info->ct_state)
    return false;

// 2. 接口匹配 (可选)
if (rule->flags & LWFW_RULE_FLAGS_NETIF &&
    strncmp(...) != 0)
    return false;

// 3. L2 匹配 (可选，默认关闭)
#ifdef LWFW_ADVANCED_FUNC_L2
    check_lwfw_l2_info()
#endif

// 4. L3 匹配
check_lwfw_l3_info()

// 5. L4 匹配
check_lwfw_l4_info()

return true;
```

### 2.3 性能分析

| 规则数 | 平均比较次数 | 最坏情况 |
|--------|------------|---------|
| 10 | 5 | 10 |
| 100 | 50 | 100 |
| 1000 | 500 | 1000 |

**问题**: 规则多时，线性扫描成为瓶颈。

---

## 3. tree_search 决策树

### 3.1 树结构

```
根
 ├─ dim=0 (src_ip), thresh=192.168.1.0
 │    ├─ 左子树: src_ip < 192.168.1.0
 │    └─ 右子树: src_ip >= 192.168.1.0
 ├─ dim=1 (dst_ip)
 │    └─ ...
 ├─ dim=2 (src_port)
 ├─ dim=3 (dst_port)
 └─ dim=4 (protocol)
```

### 3.2 树搜索算法

```c
// tree_hs.c:hs_lookup_entry()
rule_base_t *hs_lookup_entry(hs_tree_t *tree, hs_key_t *hs_key)
{
    hs_node_t *n = tree->root;

    // 1. 从根节点向下遍历
    while (n && n->child_idx != UINT32_MAX) {
        if (hs_key->key[n->d2s] <= n->thresh) {
            n = hs_node_vec_at(&tree->node_vec, n->child_idx);
        } else {
            n = hs_node_vec_at(&tree->node_vec, n->child_idx + 1);
        }
    }

    // 2. 到达叶子节点，线性扫描
    if (n)
        return linear_search_entry(&n->ruleset, hs_key);
    return NULL;
}
```

### 3.3 性能分析

| 规则数 | 树深度 | 叶子节点数 | 每叶平均规则 |
|--------|--------|-----------|------------|
| 100 | ~7 | ~16 | ~6 |
| 1000 | ~10 | ~100 | ~10 |

**预期**: O(log n) 查找，比线性扫描快 10-100 倍。

---

## 4. 匹配算法优化建议

### 4.1 规则排序优化

**问题**: 当前规则按插入顺序存储，没有按优先级或特异性排序。

**建议**: 按规则特异性降序排序，特异性高的规则（更多匹配字段）排在前面。

```c
// 规则排序策略
// 1. 按 flags 中设置的字段数降序
// 2. 按 flags 权重计算特异性分数
int rule_specificity(const lwfw_rule_t *rule) {
    int score = 0;
    if (rule->flags & LWFW_RULE_FLAGS_CT_STATE) score += 1;
    if (rule->flags & LWFW_RULE_FLAGS_NETIF) score += 1;
    if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK) score += 4;
    if (rule->flags & LWFW_RULE_FLAGS_DST_IP_MASK) score += 4;
    if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE) score += 3;
    if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_RANGE) score += 3;
    // ...
    return score;
}
```

### 4.2 快速路径跳过

**问题**: 对于大多数数据包，只有少数字段变化（如同一连接的包）。

**建议**: 使用缓存跳过已匹配的规则子集。

```c
// 简单缓存示例
typedef struct {
    uint32_t src_ip;
    uint32_t dst_ip;
    uint16_t src_port;
    uint16_t dst_port;
    uint8_t proto;
    uint16_t rule_id;  // 上次匹配的规则
} match_cache_t;

// 检查是否命中缓存
if (cache->src_ip == pkt->src_ip &&
    cache->dst_ip == pkt->dst_ip &&
    cache->src_port == pkt->src_port &&
    // ...
    cache->rule_id == last_rule_id) {
    return last_rule;  // 直接返回
}
```

### 4.3 位图索引

**问题**: 检查 `rule->flags` 判断字段是否存在，每次都要读取内存。

**建议**: 使用预计算的位图索引。

```c
// 位图索引设计
#define RULE_FIELD_BITMAP_SIZE 16
typedef struct {
    uint16_t rules_with_ct_state[RULE_FIELD_BITMAP_SIZE];
    uint16_t rules_with_netif[RULE_FIELD_BITMAP_SIZE];
    uint16_t rules_with_src_ip[RULE_FIELD_BITMAP_SIZE];
    // ...
} rule_bitmap_index_t;

// 使用位图快速过滤
uint16_t candidates = all_rules;
candidates &= rule_bitmap_index.rules_with_src_ip[pkt->src_ip >> 10];
candidates &= rule_bitmap_index.rules_with_dst_ip[pkt->dst_ip >> 10];
// 候选规则大大减少
```

### 4.4 SIMD 加速

**建议**: 对于批量数据包，使用 SIMD 并行比较。

```c
// 伪代码 - 使用 AVX2 并行比较
__m256i pkt_src_ip = _mm256_set1_epi32(pkt_info->l3.src_ip);
__m256i pkt_dst_ip = _mm256_set1_epi32(pkt_info->l3.dst_ip);

// 批量加载 8 条规则的 src_ip
__m256i rule_src_ips = _mm256_loadu_si256(&rule_table->src_ips[i]);

// 并行比较
__m256i cmp = _mm256_cmpeq_epi32(pkt_src_ip, rule_src_ips);
int mask = _mm256_movemask_epi32(cmp);
// mask 中 1 的位置表示匹配
```

---

## 5. check_rule 内部优化

### 5.1 L3 检查优化

```c
// 当前实现
inline static bool check_lwfw_l3_info(const lwfw_rule_t *rule,
                                       const lwfw_pkt_l3_info_t *pkt)
{
    // 协议检查
    if (rule->l3.proto != pkt->proto)
        return false;

    // IP 范围检查
    uint32_t src_ip = pkt->src_ip;
    if (rule->l3.src_ip.addr <= src_ip && src_ip <= rule->l3.src_ip.mask)
        return true;

    return false;
}
```

**问题**: 范围检查使用两次比较，可以优化。

### 5.2 建议优化

```c
// 优化后的范围检查
// src_ip >= rule->l3.src_addr && src_ip <= rule->l3.src_mask
// 改为: (src_ip - rule->l3.src_addr) <= (rule->l3.src_mask - rule->l3.src_addr)
//
// 或者使用无符号比较技巧
// if (src_ip >= min && src_ip <= max) 改为:
// if ((src_ip - min) <= (max - min))
```

### 5.3 短路优化

```c
// 当前: 逐字段检查，短路过慢
if (rule->flags & FLAG_CT_STATE && ct_state != rule->ct_state) return false;
if (rule->flags & FLAG_NETIF && ...) return false;
if (rule->flags & FLAG_L2 && ...) return false;
if (rule->flags & FLAG_L3 && ...) return false;
if (rule->flags & FLAG_L4 && ...) return false;

// 建议: 按字段特异性排序，优先检查高特异性字段
// 1. L3 IP 地址 (32bit, 特异性高)
if (rule->flags & FLAG_L3) {
    if (!check_l3_match()) return false;
}
// 2. L4 端口 (16bit)
if (rule->flags & FLAG_L4) {
    if (!check_l4_match()) return false;
}
// 3. L2 (Ethernet, 特异性较低)
if (rule->flags & FLAG_L2) {
    if (!check_l2_match()) return false;
}
```

---

## 6. 规则预编译

### 6.1 Idea

将 YAML 规则预编译为二进制格式，避免运行时解析开销。

```c
// 二进制规则格式
struct binary_rule {
    uint32_t flags;              // 匹配标志
    uint8_t proto;                // 协议
    uint8_t action;               // 动作
    uint16_t ct_state;            // 连接状态

    uint32_t src_ip_min;
    uint32_t src_ip_max;
    uint32_t dst_ip_min;
    uint32_t dst_ip_max;

    uint16_t src_port_min;
    uint16_t src_port_max;
    uint16_t dst_port_min;
    uint16_t dst_port_max;
};
```

### 6.2 预编译优势

1. **无需运行时解析**: YAML 解析非常耗时
2. **内存布局优化**: 字段按缓存行对齐
3. **SIMD 友好**: 连续内存便于批量加载

---

## 7. 优化建议汇总

| 优先级 | 优化 | 预期收益 | 复杂度 |
|--------|------|----------|--------|
| **P0** | 规则按特异性排序 | 减少平均比较次数 30-50% | 低 |
| **P1** | 匹配缓存 | 同连接包跳过规则查找 | 中 |
| **P1** | 快速路径跳过 | 根据 L3 信息快速过滤候选集 | 中 |
| **P2** | 位图索引 | O(1) 过滤不符合字段的规则 | 高 |
| **P2** | 规则预编译 | 消除 YAML 解析开销 | 高 |
| **P3** | SIMD 加速 | 批量处理时 4-8x 加速 | 高 |

---

## 8. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `lwfw.c` | 565 | `check_rule` 匹配入口 |
| `lwfw.c` | 383 | `check_lwfw_l2_info` L2 匹配 |
| `lwfw.c` | 447 | `check_lwfw_l3_info` L3 匹配 |
| `lwfw.c` | 498 | `check_lwfw_l4_info` L4 匹配 |
| `lwfw.c` | 1884 | `list_search_do_filter` 线性扫描 |
| `tree_entry.c` | 267 | `tree_search_do_filter` 树搜索 |
| `tree_hs.c` | 785 | `hs_lookup_entry` 树遍历 |
