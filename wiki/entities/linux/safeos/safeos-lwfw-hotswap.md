---
type: entity
tags: [safeos, lwfw, firewall, hot-reload, dual-buffer, atomic-swap, policy]
created: 2026-05-27
sources: [safeos-lwfw]
---

# LWFW 热切换与策略原子性

## 定义

LWFW 防火墙使用双缓冲架构实现配置热切换：两份策略缓冲 (`lwfw_policy` 和 `lwfw_policy_swap`)，通过原子指针交换实现不中断数据包处理的热更新。

## 双缓冲架构

```c
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };         // 活跃策略
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP }; // 备份策略
lwfw_policy_t *policy;           // 当前活跃策略
lwfw_policy_t *inactive_policy; // 备份/目标策略
```

## 热切换流程 (TREE 模式)

```c
// 1. 加锁 — 此后数据包处理被阻塞
sync_mutex_lock(&lwfw_p->policy_lock);

// 2. 清理目标策略
lwfw_policy_clean(lwfw_p->inactive_policy);

// 3. 深拷贝当前策略到目标策略
lwfw_copy_policy(lwfw_p->inactive_policy, lwfw_p->policy);

// 4. 重建目标策略的树索引
lwfw_p->inactive_policy->filter_engine->init(lwfw_p->inactive_policy, 0);

// 5. 原子交换指针
lwfw_policy_t *tmp = lwfw_p->policy;
lwfw_p->policy = lwfw_p->inactive_policy;
lwfw_p->inactive_policy = tmp;

// 6. 解锁
sync_mutex_unlock(&lwfw_p->policy_lock);
```

## 已知问题

### P0: 热重载窗口期无防护

```c
// lwfw_notif.c:82-91 — 严重问题
if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
    lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state = LWFW_STATE_DISABLE;  // ← 防火墙失效！
    lwfw_config_reload_manifest(fw_ctrl_p->cfg_path);  // ← 重载期间无防护！
    lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state = LWFW_STATE_ENABLE;  // ← 恢复
}
```

**问题**: 热重载期间 Ingress 防火墙完全失效。

### 深拷贝无错误恢复

`lwfw_copy_policy` 拷贝过程中如果失败，`dst_policy` 已是部分修改状态，无法回滚。

## 优化方向

| 方案 | 说明 | 复杂度 |
|------|------|--------|
| **RCU 无锁切换** | 读者无锁并发，写者原子替换 | 高 |
| **读写锁分离** | 读者之间不互斥，深拷贝期间可继续读旧策略 | 中 |
| **Copy-on-Write** | 策略结构共享，只在修改时复制 | 中 |
| **分批拷贝** | 深拷贝分多帧完成，每帧之间释放锁 | 低 |

## 相关概念

- [[sources/safeos-lwfw]] — LWFW 防火墙完整分析
- [[lwfw-index]] — LWFW 模块索引
- [[safeos-index]] — SafeOS NSv 架构索引
