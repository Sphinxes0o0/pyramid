# 热切换与策略原子性深度分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw.c`
> 相关文件: `lwfw_notif.c`, `lwfw_data_structure.md`

---

## 1. 热切换机制概述

### 1.1 双缓冲架构

```c
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };

// 运行时指针
lwfw_policy_t *policy;           // 当前活跃策略
lwfw_policy_t *inactive_policy; // 备份/目标策略
```

**设计意图**: 使用两份策略缓冲，实现配置热切换而不中断数据包处理。

### 1.2 两套内存池

| 池 | memp_type | 用途 |
|----|-----------|------|
| `MEMP_LWFW_RULE` | 活跃策略规则 | 当前生效的规则 |
| `MEMP_LWFW_RULE_SWAP` | 热切换备份规则 | 切换时的目标规则 |

**问题**: 规则多时内存占用翻倍。

---

## 2. 热切换流程分析

### 2.1 完整切换序列 (TREE 模式)

```c
// lwfw_config_reset_state -> case LWFW_CONFIG_RELOAD_MANIFEST
#if LWFW_TREE_SEARCH_EN
    // 1. 加锁 - 此后数据包处理被阻塞
    sync_mutex_lock(&lwfw_p->policy_lock);

    // 2. 清理目标策略
    lwfw_policy_clean(lwfw_p->inactive_policy);

    // 3. 深拷贝当前策略到目标策略
    ret = lwfw_copy_policy(lwfw_p->inactive_policy, lwfw_p->policy);
    if (ret != LWFW_ERR_OK) {
        sync_mutex_unlock(&lwfw_p->policy_lock);
        goto err_exit;
    }

    // 4. 重建目标策略的树索引
    lwfw_p->inactive_policy->filter_engine->init(lwfw_p->inactive_policy, 0);

    // 5. 原子交换指针
    lwfw_policy_t *tmp = lwfw_p->policy;
    lwfw_p->policy = lwfw_p->inactive_policy;
    lwfw_p->inactive_policy = tmp;

    // 6. 解锁
    sync_mutex_unlock(&lwfw_p->policy_lock);
#endif
```

### 2.2 深拷贝实现 (`lwfw_copy_policy`)

```c
static int lwfw_copy_policy(lwfw_policy_t *dst_policy, lwfw_policy_t *src_policy)
{
    // 1. 拷贝元数据 (sizeof(lwfw_policy_t) - sizeof(rules))
    memcpy(dst_policy, src_policy,
           __builtin_offsetof(lwfw_policy_t, __lwfw_policy_copy_offset));

    // 2. 逐表拷贝规则
    for (int i = 0; i < LWFW_MAX_COUNT_TABLE; i++) {
        ret = lwfw_copy_rule_table(&dst_policy->rule_tables[i],
                                   &src_policy->rule_tables[i], dst_policy);
        if (err != LWFW_ERR_OK) {
            // 清理已拷贝的表
            for (int j = 0; j < i; j++) {
                lwfw_rule_table_clean(&dst_policy->rule_tables[j], dst_policy->memp_type);
            }
            goto __exit;
        }
    }
    return LWFW_ERR_OK;
}
```

### 2.3 规则表拷贝 (`lwfw_copy_rule_table`)

```c
static int lwfw_copy_rule_table(lwfw_rule_table_t *dst_table,
                                 lwfw_rule_table_t *src_table,
                                 lwfw_policy_t *policy)
{
    // 1. 拷贝表元数据
    memcpy(dst_table, src_table,
           __builtin_offsetof(lwfw_rule_table_t, __lwfw_table_copy_offset));

    // 2. 初始化链表头
    cdlist_init(&dst_table->header);

    // 3. 遍历源规则，深拷贝每条
    cdlist_iter_entry(curr_rule, &src_table->header, next) {
        lwfw_rule_t *new_rule = (lwfw_rule_t *)memp_malloc(policy->memp_type);
        if (!new_rule) {
            return LWFW_ERR_NO_MEM;
        }
        memcpy(new_rule, curr_rule, sizeof(lwfw_rule_t));
        cdlist_add_tail(&dst_table->header, &new_rule->next);
    }
    return LWFW_ERR_OK;
}
```

---

## 3. 深拷贝阻塞时间分析

### 3.1 持锁时间计算

```
总持锁时间 = t_clean + t_copy_metadata + t_copy_rules + t_tree_init + t_swap
```

其中 `t_copy_rules` 是最关键的部分:

```
t_copy_rules = Σ(每条规则的拷贝时间)
             = rule_count × sizeof(lwfw_rule_t) / bandwidth
```

### 3.2 典型值估算

| 参数 | 值 |
|------|-----|
| `sizeof(lwfw_rule_t)` | ~256 bytes (缓存行对齐) |
| 规则数量 | 100 条 |
| MEMP 分配 + 拷贝延迟 | ~1-5 μs/rule |
| **100条规则总时间** | **~100-500 μs** |

**结论**: 100 条规则时持锁约 0.5ms，可接受。但如果规则数量达到 1000+，可能达到 5-10ms，这会对实时性造成影响。

### 3.3 树索引重建时间

```c
lwfw_p->inactive_policy->filter_engine->init(lwfw_p->inactive_policy, 0);
// 在 tree_entry.c 中调用:
//   - lwfw_build_hs_tree() 构建 hyperscan 树
//   - hs_split_node() 递归分割节点
//   - hs_gen_segs() 生成段
```

树构建复杂度: O(n log n) 其中 n 是规则数量。对于 1000 条规则，估计需要 5-20ms。

---

## 4. 严重问题：热重载窗口期

### 4.1 问题代码 (lwfw_notif.c:82-91)

```c
if (__atomic_load_n(&fw_ctrl_p->cfg_in_reloading, __ATOMIC_SEQ_CST)) {
    // !!!!! 严重问题 !!!!!
    // 这里先把 Ingress 防火墙 DISABLE
    if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_ENABLE) {
       lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state = LWFW_STATE_DISABLE;  // ← 防火墙失效！

       lwfw_config_reload_manifest(fw_ctrl_p->cfg_path);  // ← 这期间无防护！

       lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state = LWFW_STATE_ENABLE;  // ← 恢复
    } else {
       lwfw_config_reload_manifest(fw_ctrl_p->cfg_path);
    }
    __atomic_store_n(&fw_ctrl_p->cfg_in_reloading, false, __ATOMIC_SEQ_CST);
}
```

### 4.2 问题分析

| 项目 | 内容 |
|------|------|
| 问题 | 热重载期间 Ingress 防火墙完全失效 |
| 影响 | 所有入口数据包在重载期间不被过滤 |
| 时长 | 取决于 `lwfw_config_reload_manifest` 执行时间 |
| 严重性 | **P0 - 严重** |

### 4.3 正确实现应该是

```c
// 建议: 使用双策略切换，不修改当前策略
lwfw_policy_t *new_policy = lwfw_p->inactive_policy;

// 1. 在新策略上加载配置
lwfw_copy_policy(new_policy, lwfw_p->policy);
lwfw_config_reload_manifest_into(new_policy);
new_policy->filter_engine->init(new_policy, 0);

// 2. 原子切换
sync_mutex_lock(&lwfw_p->policy_lock);
lwfw_policy_t *tmp = lwfw_p->policy;
lwfw_p->policy = new_policy;
lwfw_p->inactive_policy = tmp;
sync_mutex_unlock(&lwfw_p->policy_lock);
```

---

## 5. 深拷贝错误恢复问题

### 5.1 问题

`lwfw_copy_policy` 在拷贝过程中如果失败，`dst_policy` 已经是部分修改状态，无法回滚到之前的状态。

```c
static int lwfw_copy_policy(lwfw_policy_t *dst_policy, lwfw_policy_t *src_policy)
{
    // 如果在拷贝 table[3] 时失败
    for (int i = 0; i < LWFW_MAX_COUNT_TABLE; i++) {
        err = lwfw_copy_rule_table(&dst_policy->rule_tables[i], ...);
        if (err != LWFW_ERR_OK) {
            // 此时 dst_policy 的 table[0-2] 已经修改，无法回滚！
            for (int j = 0; j < i; j++) {
                lwfw_rule_table_clean(&dst_policy->rule_tables[j], ...);
            }
            goto __exit;
        }
    }
}
```

### 5.2 影响

- 目标策略损坏
- 如果此时切换到损坏的目标策略，防火墙行为不确定
- 只能重启防火墙恢复

### 5.3 建议修复

```c
// 方案1: 先拷贝到临时 buffer，成功后再替换
static int lwfw_copy_policy_safe(lwfw_policy_t *dst_policy, lwfw_policy_t *src_policy)
{
    lwfw_policy_t tmp = {0};
    int err = LWFW_ERR_OK;

    // 1. 先清空临时策略
    lwfw_policy_clean(&tmp);

    // 2. 拷贝到临时策略
    err = lwfw_copy_policy_impl(&tmp, src_policy);
    if (err != LWFW_ERR_OK) {
        lwfw_policy_clean(&tmp);
        return err;
    }

    // 3. 原子替换
    memcpy(dst_policy, &tmp, sizeof(lwfw_policy_t));
    return LWFW_ERR_OK;
}

// 方案2: 使用事务性更新
```

---

## 6. RCU 无锁切换可行性分析

### 6.1 RCU 原理

Read-Copy-Update (RCU) 允许多个读者无锁并发访问数据，只有在 writer 完成时才需要同步。

```c
// RCU 写法
writer:
    new_policy = build_new_policy();
    rcu_assign_pointer(lwfw_p->policy, new_policy);
    synchronize_rcu();  // 等待所有旧读者完成
    free_old_policy(old_policy);

reader:
    rcu_read_lock();
    policy = rcu_dereference(lwfw_p->policy);
    // 使用 policy...
    rcu_read_unlock();
```

### 6.2 在 lwfw 中应用

**当前架构问题**:
- `policy_lock` 是 mutex，读者和写者互斥
- 深拷贝期间所有数据包处理被阻塞

**RCU 改造可行方案**:

```c
// 全局指针 (原子操作)
extern volatile lwfw_policy_t *lwfw_policies[2];
volatile int current_policy_idx = 0;

// 切换时
int new_idx = 1 - current_policy_idx;
build_policy(&lwfw_policies[new_idx]);  // 在新缓冲构建
// 等待所有数据包处理完成
synchronize_rcu();
// 原子切换
current_policy_idx = new_idx;

// 数据包处理路径 (无锁)
rcu_read_lock();
lwfw_policy_t *policy = rcu_dereference(lwfw_policies[current_policy_idx]);
// 使用 policy 进行过滤...
rcu_read_unlock();
```

### 6.3 风险与挑战

| 项目 | 内容 |
|------|------|
| seL4 支持 | 需要确认 seL4 是否提供 RCU 原语 |
| 延迟释放 | synchronize_rcu() 可能阻塞较长时间 |
| 内存压力 | 旧策略需要等到所有读者完成后才能释放 |
| 复杂度 | 引入 RCU 会增加代码复杂度 |

### 6.4 替代方案

**方案 A: 读写锁分离**
- 读者之间不互斥，只在写者之间互斥
- 深拷贝期间数据包处理可继续（读旧策略）

**方案 B:Copy-on-Write (COW)**
- 策略结构共享，只在修改时复制
- 读操作直接访问共享结构，无锁

**方案 C: 分批拷贝**
- 将深拷贝分多帧完成，每帧处理少量规则
- 每帧之间释放锁，允许数据包处理

---

## 7. 策略一致性保证

### 7.1 当前一致性模型

| 场景 | 一致性保证 |
|------|-----------|
| 规则修改 | 整个规则表原子替换 |
| 热切换 | 双缓冲 + 原子指针交换 |
| 规则命中统计 | 原子操作 `__atomic_fetch_add` |
| 状态标志 | `__atomic_load_n` / `__atomic_store_n` |

### 7.2 潜在一致性问题

**问题 1: 规则表切换与规则内部状态**

```c
// 假设规则 A 被命中，增加 hit_cnt
curr_rule->hit_cnt++;  // 原子操作

// 但如果此时发生热切换...
// curr_rule 指向的是旧策略的规则
// 新策略中的对应规则 hit_cnt 仍然是 0
```

**问题 2: 规则表状态不一致**

```c
// 切换过程中
lwfw_p->policy = inactive;  // 刚刚切换
lwfw_p->inactive_policy = old;  // 旧策略

// 此时如果检查 state:
lwfw_p->policy->rule_tables[IN_TABLE].state  // 新策略
lwfw_p->inactive_policy->rule_tables[OUT_TABLE].state  // 旧策略

// 两个表来自不同时间点的快照！
```

### 7.3 建议

1. **添加版本号**: 每个策略有单调递增的版本号，过滤时检查版本
2. **统一切换**: Ingress/Egress 应作为一个整体切换，不分开
3. **双策略同时就绪**: 切换前确保新策略完全初始化

---

## 8. 优化建议汇总

| 优先级 | 问题 | 建议 | 复杂度 |
|--------|------|------|--------|
| **P0** | 热重载窗口期无防护 | 使用双策略切换，不 DISABLE 防火墙 | 高 |
| **P0** | 深拷贝无错误恢复 | 使用临时 buffer 或事务性更新 | 中 |
| **P1** | 深拷贝持锁时间长 | 改为 RCU/COW 或分批拷贝 | 高 |
| **P1** | 策略版本不一致 | 添加策略版本号 | 低 |
| **P2** | 双内存池占用大 | 考虑共享只读数据 | 中 |

---

## 9. 关键代码位置

| 文件 | 函数/行号 | 描述 |
|------|-----------|------|
| `lwfw.c:1157` | `lwfw_copy_policy` | 策略深拷贝 |
| `lwfw.c:1132` | `lwfw_copy_rule_table` | 规则表拷贝 |
| `lwfw.c:1285-1297` | 热切换主流程 | 持锁 + 拷贝 + 切换 |
| `lwfw_notif.c:82-91` | 热重载窗口期问题 | DISABLE 防火墙后重载 |
| `lwfw.c:2205` | `lwfw_init` | 全局变量初始化 |
