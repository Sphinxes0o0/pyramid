---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Hot-Swap Analysis

## 定义

LWFW 热切换机制使用**双缓冲架构**实现策略的原子更新：两套策略结构 (`policy` / `inactive_policy`) 通过指针交换实现零 downtime 配置重载。

## 双缓冲架构

```c
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };

// 运行时指针
lwfw_policy_t *policy;           // 当前活跃策略
lwfw_policy_t *inactive_policy; // 备份/目标策略
```

## 热切换流程 (TREE 模式)

```c
// lwfw_config_reset_state
sync_mutex_lock(&lwfw_p->policy_lock);
lwfw_policy_clean(lwfw_p->inactive_policy);
ret = lwfw_copy_policy(lwfw_p->inactive_policy, lwfw_p->policy);
inactive_policy->filter_engine->init(lwfw_p->inactive_policy, 0);
lwfw_policy_t *tmp = lwfw_p->policy;
lwfw_p->policy = lwfw_p->inactive_policy;
lwfw_p->inactive_policy = tmp;
sync_mutex_unlock(&lwfw_p->policy_lock);
```

## 严重问题：热重载窗口期

```c
// lwfw_notif.c 中的错误实现
if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
   lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state = LWFW_STATE_DISABLE;  // ← 防火墙失效！
   lwfw_config_reload_manifest(fw_ctrl_p->cfg_path);  // ← 这期间无防护！
   lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state = LWFW_STATE_ENABLE;
}
```

**P0 严重问题**: 热重载期间 Ingress 防火墙完全失效，所有入口数据包不被过滤。

## 深拷贝阻塞时间

| 参数 | 值 |
|------|-----|
| `sizeof(lwfw_rule_t)` | ~256 bytes |
| 规则数量 | 100 条 |
| MEMP 分配 + 拷贝延迟 | ~1-5 μs/rule |
| **100条规则总时间** | **~100-500 μs** |

## 深拷贝错误恢复问题

`lwfw_copy_policy` 在拷贝过程中如果失败，`dst_policy` 已是部分修改状态，无法回滚。

## RCU 无锁切换可行性

```c
// 可行方案: 读写锁分离
writer:
    mutex_lock(&policy_lock);
    // 深拷贝
    mutex_unlock(&policy_lock);

reader:
    // 无锁读取当前 policy
    policy = rcu_dereference(lwfw_p->policy);
```

## 相关概念

- [[entities/linux/lwfw/lwfw-architecture]] — 整体架构
- [[entities/linux/lwfw/lwfw-data-structure]] — 双策略结构
- [[entities/linux/lwfw/lwfw-core-filtering]] — 过滤逻辑
- [[entities/linux/lwfw/lwfw-notif]] — 通知线程中的热重载触发
