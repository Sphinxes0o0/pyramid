# 树搜索过滤引擎分析

> 代码路径: `libs/util_libs/liblwfw/src/tree_hs.c`
> 设计参考: Intel Hyperscan 单模匹配算法

---

## 1. 整体架构

Hyperscan 风格的决策树 (Decision Tree)，将规则按维度分割为节点:

```
规则库 (ruleset)
    │
    ├─ 维度0 (src_ip) ─── 分割点段 (segments)
    │       │
    │       ├─ 节点A: [0.0.0.0-192.168.127.254]
    │       └─ 节点B: [192.168.1.0-255.255.255.255]
    │
    ├─ 维度1 (dst_ip) ─── 各节点内再分割
    │
    ├─ 维度2 (src_port)
    │
    ├─ 维度3 (dst_port)
    │
    └─ 维度4 (protocol)
```

搜索时从根节点递归，找到最深的叶子节点，然后线性扫描该节点内的规则。

---

## 2. 核心数据结构

### 2.1 规则基础结构

```c
// 统一规则头
typedef struct rule_base {
  uint32_t pri;       // 规则优先级
  uint32_t action;   // 动作
  uint32_t flags;
  uint32_t range[RS_MAX_DIM][2];  // 每维的范围 [low, high]
} rule_base_t;

// 带接口名的完整规则 (IPv4)
typedef struct rule {
  rule_base_t base;
  uint32_t rule_id;
  char if_name[IFNAMSIZ];
  void *orig_rule;    // 指向原始 lwfw_rule
} rule_t;
```

### 2.2 树节点

```c
typedef struct hs_node {
  struct rule_set ruleset;    // 该节点包含的规则集
  struct hs_node_vec children; // 子节点向量
  uint32_t dim;               // 分割维度 (0-4)
  uint32_t seg_count;         // 段数量
  struct range1d *segs;        // 分割后的段数组
} hs_node_t;
```

### 2.3 搜索键值

```c
// IPv4 搜索键
typedef struct hs_key4 {
  uint32_t key[4];  // [src_ip, dst_ip, src_port, dst_port]
  struct {
    char if_name[IFNAMSIZ];
  } more;
} hs_key4_t;
```

---

## 3. 树的构建

### 3.1 构建入口

```c
hs_tree_build(tree, ruleset, params)
  ├─ hs_prep_build(tree, aux, ruleset)  → 分配节点向量
  ├─ remove_redund(ruleset)               → 删除被包含的规则
  ├─ hs_split_node(tree->root, ruleset, aux, 0)  → 递归分割
  └─ return tree
```

### 3.2 规则去重

```c
// 删除被其他规则包含的低优先级规则
static void remove_redund(ruleset)
{
  for (i = 1; i < num; i++) {
    for (j = 0; j < i; j++) {
      if (rule_contained(ruleset[j], ruleset[i])) {
        ruleset[i].pri = UINT32_MAX;  // 标记删除
      }
    }
  }
  compact(ruleset);  // 删除标记的规则
}
```

### 3.3 节点分割

```c
static void hs_split_node(node, ruleset, aux, dim)
{
  if (ruleset->num <= bucket_size) return;  // 足够小则停止

  dim = dim % RS_MAX_DIM;

  // 1. 生成段 (按该维度分割规则)
  segs = hs_gen_segs(node, aux, dim);
  node->dim = dim;
  node->segs = segs;

  // 2. 将规则分配到子节点
  for (每条规则 r) {
    子节点 = find_child(node, r->range[dim]);
    rule_add_to_child(子节点, r);
  }

  // 3. 递归构建子节点
  for (每个子节点 child) {
    hs_split_node(child, child->ruleset, aux, dim+1);
  }
}
```

### 3.4 段生成算法

```c
// 将所有规则在该维度的范围区间分割为不重叠的段
static int hs_gen_segs(node, aux, dim)
{
  // 1. 收集所有规则的边界
  for (i = 0; i < num; i++)
    ranges_sort[i] = {rule[i].range[dim][0], rule[i].range[dim][1]}

  // 2. 排序并合并重叠区间
  qsort(ranges, num);
  unique_ranges(ranges, num);

  // 3. 用堆生成段
  // 例: 规则A: [0,100], 规则B: [50,200]
  // 生成段: [0,49], [50,100], [101,200]
}
```

---

## 4. 搜索算法

### 4.1 入口

```c
hs_linear_search_entry(ruleset, key)  // 线性扫描 (当前只有这一种)
  └─ linear_search_entry(ruleset, key)
        ├─ 遍历所有规则
        ├─ 对每条规则检查所有维度
        └─ 全部命中则返回规则指针
```

**注意**: 当前 `tree_search` 模式实际调用的是 `hs_linear_search_entry`，即线性扫描，并未使用树的层级结构加速！

### 4.2 线性扫描

```c
static inline rule_base_t * linear_search_entry(ruleset, key)
{
  for (i = 0; i < num; i++) {
    r = rule_base_from_rs(ruleset, i);
    for (j = 0; j < dim; j++) {
      if (key->key[j] < r->range[j][0] || key->key[j] > r->range[j][1])
        goto next_rule;
    }
    if (NETIF && if_name not match)
      goto next_rule;
    return r;
next_rule:
    continue;
  }
  return NULL;  // 无匹配
}
```

---

## 5. lwfw 集成

### 5.1 引擎注册

```c
// lwfw.c:2073-2075
if (policy->params.filter_mode == LWFW_FILTER_TREE) {
  policy->filter_engine = &tree_search_eng;
  policy->filter_engine->init(policy, 0);
}
```

### 5.2 规则转换

规则从 lwfw 链表格式转换为树格式:

```c
// lwfw_build_hs(ruleset, dir)
├─ lwfw_copy_rule_table_to_ruleset()
│     ├─ 遍历 lwfw_rule 链表
│     ├─ 转换为 rule_t 结构
│     └─ 填充 range[dim][2]
└─ hs_tree_build(tree, ruleset)
```

### 5.3 维度映射

| 维度 | lwfw 字段 | 范围 |
|------|----------|------|
| 0 | src_ip | 0x00000000 - 0xFFFFFFFF |
| 1 | dst_ip | 0x00000000 - 0xFFFFFFFF |
| 2 | src_port | 0 - 65535 |
| 3 | dst_port | 0 - 65535 |
| 4 | protocol | 0 - 255 |

---

## 6. 关键问题

### 6.1 关于"树搜索退化"的分析结论

**经过源码验证，树搜索实际上是正确实现的。**

优化文档 (`lwfw_optimization.md`) 声称 tree 模式调用 `hs_linear_search_entry` 对整个 ruleset 线性扫描，但实际代码分析显示：

**实际调用链**:
```
tree_search_do_filter (tree_entry.c:315)
  └─ hs_lookup_entry(tree_hs.c:785)
        ├─ 树遍历: while(n && n->child_idx != UINT32_MAX) { ... }
        └─ leaf节点才调用: linear_search_entry(&n->ruleset, key)
```

**hs_lookup_entry 正确实现了树遍历** (tree_hs.c:785-804):
```c
rule_base_t *hs_lookup_entry(hs_tree_t *tree, hs_key_t *hs_key)
{
    hs_node_t *n = tree->root;
    while(n && n->child_idx != UINT32_MAX) {
        // 按维度分割点导航: d2s=维度, thresh=阈值
        if(hs_key->key[n->d2s] <= n->thresh) {
            n = hs_node_vec_at(&tree->node_vec, n->child_idx);  // 左子树
        } else {
            n = hs_node_vec_at(&tree->node_vec, n->child_idx +1); // 右子树
        }
    }
    // 只在叶子节点线性扫描
    if (n)
        return linear_search_entry(&n->ruleset, hs_key);
    return NULL;
}
```

**注意**: `hs_linear_search` (tree_hs.c:110) 虽然也是线性扫描，但**从未被调用**。它只是测试代码中使用的函数。

**可能的问题场景**:
- 如果树构建算法质量差，所有规则可能集中在少数叶子节点
- `bucketSize = 8` 的阈值可能过大，导致叶子节点规则过多
- 维度选择算法可能不适合特定规则分布

### 6.2 维度固定

```c
#define RS_MAX_DIM  5
```

维度硬编码为 5，无法扩展支持 L2 字段 (MAC, VLAN)。

### 6.3 段生成内存分配

```c
// hs_gen_segs 使用堆来生成段
heap_offer(&aux->heap, &r->high)  // 堆操作
```

每次分割都重新分配/释放内存，效率低。

### 6.4 无优先级排序

规则按插入顺序存储，搜索时从头遍历。没有按优先级排序，即使高优先级规则排在链表尾部也会遍历到最后。

---

## 7. 优化方向

### 7.1 实现真正的树搜索

```c
// 正确的树搜索算法
rule_base_t *tree_search(hs_node_t *node, hs_key_t *key)
{
  if (node->children.len == 0) {
    // 叶子节点，线性扫描
    return linear_search_entry(&node->ruleset, key);
  }

  // 找到 key 落在哪个段
  for (i = 0; i < node->seg_count; i++) {
    if (key->key[node->dim] >= node->segs[i].low &&
        key->key[node->dim] <= node->segs[i].high) {
      // 递归搜索子节点
      child = node->children.hs_nodes[i];
      return tree_search(child, key);
    }
  }
  return NULL;
}
```

### 7.2 支持 L2 维度扩展

将 RS_MAX_DIM 改为可配置，并添加 L2 字段支持。

### 7.3 规则优先级排序

在构建规则集时按优先级降序排序，搜索时首次匹配即最优匹配。
