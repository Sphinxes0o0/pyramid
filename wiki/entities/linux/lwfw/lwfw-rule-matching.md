---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Rule Matching Optimization

## 定义

LWFW 规则匹配算法优化空间分析，覆盖 list_search 线性扫描和 tree_search 决策树两种引擎的性能特征与优化方向。

## 当前匹配架构

| 引擎 | 模式 | 数据结构 | 适用场景 |
|------|------|---------|---------|
| `list_search_eng` | 线性扫描 | cdlist 链表 | 规则少 (<20) |
| `tree_search_eng` | 决策树 | hyperscan 树 | 规则多 (>20) |

## list_search 线性扫描

```c
// 遍历 cdlist 链表
cdlist_iter_entry(curr_rule, header, next) {
  if (curr_rule->state == LWFW_STATE_DISABLE)
    continue;
  matched = check_rule(curr_rule, info, info->dir);
  if (matched)
    break;  // 首次匹配即停止
}
```

**性能分析**:

| 规则数 | 平均比较次数 | 最坏情况 |
|--------|------------|---------|
| 10 | 5 | 10 |
| 100 | 50 | 100 |
| 1000 | 500 | 1000 |

## tree_search 决策树

```
根
 ├─ dim=0 (src_ip), thresh=192.168.1.0
 │    ├─ 左: src_ip < 192.168.1.0
 │    └─ 右: src_ip >= 192.168.1.0
 ├─ dim=1 (dst_ip)
 ├─ dim=2 (src_port)
 ├─ dim=3 (dst_port)
 └─ dim=4 (protocol)
```

## check_rule 匹配顺序

```c
// 1. CT_STATE 匹配 (可选)
if (rule->flags & CT_STATE && rule->ct_state != info->ct_state)
  return false;

// 2. NETIF 接口匹配 (可选)
if (rule->flags & NETIF && strncmp(...) != 0)
  return false;

// 3. L2 匹配 (需 LWFW_ADVANCED_FUNC_L2)
if (!check_lwfw_l2_info(rule, &info->l2))
  return false;

// 4. L3 匹配
if (!check_lwfw_l3_info(rule, &info->l3))
  return false;

// 5. L4 匹配
if (!check_lwfw_l4_info(rule, &info->l4))
  return false;
```

## 优化建议汇总

| 优先级 | 优化 | 预期收益 | 复杂度 |
|--------|------|----------|--------|
| **P0** | 规则按特异性排序 | 减少平均比较次数 30-50% | 低 |
| **P1** | 匹配缓存 | 同连接包跳过规则查找 | 中 |
| **P1** | 快速路径跳过 | 根据 L3 信息快速过滤候选集 | 中 |
| **P2** | 位图索引 | O(1) 过滤不符合字段的规则 | 高 |
| **P2** | 规则预编译 | 消除 YAML 解析开销 | 高 |
| **P3** | SIMD 加速 | 批量处理时 4-8x 加速 | 高 |

## 相关概念

- [[entities/linux/lwfw/lwfw-list-search]] — 线性扫描引擎
- [[entities/linux/lwfw/lwfw-tree-search]] — 树搜索引擎
- [[entities/linux/lwfw/lwfw-core-filtering]] — check_rule 实现
