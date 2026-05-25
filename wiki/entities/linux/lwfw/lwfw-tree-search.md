---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Tree Search

## 定义

LWFW 树搜索过滤引擎基于 **Intel Hyperscan 单模匹配算法**风格，使用决策树将规则按维度分割，搜索时从根节点递归到叶子节点，然后在叶子节点内线性扫描。

## 整体架构

```
规则库 (ruleset)
    │
    ├─ 维度0 (src_ip) ─── 分割点段 (segments)
    │       │
    │       ├─ 节点A: [0.0.0.0-192.168.127.254]
    │       └─ 节点B: [192.168.1.0-255.255.255.255]
    │
    ├─ 维度1 (dst_ip) ─── 各节点内再分割
    ├─ 维度2 (src_port)
    ├─ 维度3 (dst_port)
    └─ 维度4 (protocol)
```

## 核心数据结构

```c
// 规则基础结构
typedef struct rule_base {
  uint32_t pri;       // 优先级
  uint32_t action;    // 动作
  uint32_t flags;
  uint32_t range[RS_MAX_DIM][2];  // 每维的范围 [low, high]
} rule_base_t;

// 搜索键值
typedef struct hs_key4 {
  uint32_t key[4];  // [src_ip, dst_ip, src_port, dst_port]
} hs_key4_t;
```

## 树搜索算法

```c
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

## 维度映射

| 维度 | lwfw 字段 | 范围 |
|------|----------|------|
| 0 | src_ip | 0x00000000 - 0xFFFFFFFF |
| 1 | dst_ip | 0x00000000 - 0xFFFFFFFF |
| 2 | src_port | 0 - 65535 |
| 3 | dst_port | 0 - 65535 |
| 4 | protocol | 0 - 255 |

## 性能分析

| 规则数 | 树深度 | 叶子节点数 | 每叶平均规则 |
|--------|--------|-----------|------------|
| 100 | ~7 | ~16 | ~6 |
| 1000 | ~10 | ~100 | ~10 |

## 已知问题

| 问题 | 说明 |
|------|------|
| bucketSize=8 阈值 | 可能导致叶子节点规则过多 |
| 维度选择 | 高维度规则分布不均匀时可能产生不平衡树 |
| 无 L2 维度 | 树搜索不支持 L2 (VLAN/MAC) 字段 |

## 相关概念

- [[entities/linux/lwfw/lwfw-list-search]] — 线性扫描引擎
- [[entities/linux/lwfw/lwfw-core-filtering]] — 引擎选择逻辑
- [[entities/linux/lwfw/lwfw-rule-matching]] — 匹配算法优化
