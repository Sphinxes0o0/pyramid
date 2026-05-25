# lwct GC 线程与内存管理深度分析

> 代码路径: `libs/util_libs/liblwfw/src/lwct/lwct_core.c`
> 头文件: `libs/util_libs/liblwfw/include/lwct/lwct.h`

---

## 1. GC 线程架构

### 1.1 线程启动

```c
// lwct_core.c:1121
sys_thread_new(LWCT_GC_THREAD_NAME, lwct_gc_thread, NULL,
               LWCT_GC_THREAD_STACKSIZE, LWCT_GC_THREAD_PRIO);
```

### 1.2 扫描间隔

```c
// 默认: 3000ms (3秒)
scan_interval = lwct_parameters.lwct_gc_scan_interval;

if (scan_interval >= MAX_TIMER_VAL_PER_SECOND) {
    tv.tv_sec = scan_interval / MAX_TIMER_VAL_PER_SECOND;
    tv.tv_nsec = (scan_interval % MAX_TIMER_VAL_PER_SECOND) * NS_IN_MS;
} else {
    tv.tv_sec = 0;
    tv.tv_nsec = scan_interval * NS_IN_MS;  // < 1s 粒度
}
```

---

## 2. GC 扫描算法

### 2.1 扫描循环

```c
void lwct_gc_thread(void *arg) {
    lwct_get_hash_tbl(&ct_hash_tbl, &bkt_size);

    while (1) {
        // 1. 扫描所有桶
        while (bkt1 < bkt_size) {
            lock1 = LWCT_BKT_IDX_TO_LOCK_IDX(bkt1);
            if (lwct_ct_single_trylock(lock1) == 0) {  // 获取桶锁
                cdlist_iter_entry_safe(cur_tuple, tmp_tuple, &ct_hash_tbl[bkt1], node) {
                    // 检查并删除过期连接
                    if (lwct_should_gc(ct_conn)) {
                        lwct_set_dying_bit(ct_conn);
                        // 从两个桶中删除
                        cdlist_del(&ct_conn->tuplehash[ORIGINAL].node);
                        cdlist_del(&ct_conn->tuplehash[REPLY].node);
                        lwct_conn_put(ct_conn);  // 释放引用
                    }
                }
                lwct_ct_single_unlock(lock1);
            }

            // 每轮扫描上限，防止长时间持锁
            if (scanned >= lwct_parameters.lwct_gc_entry_per_scan) {
                yield();  // 让出 CPU
            }
        }

        // 2. 等待下次扫描
        sys_time_sleep(tv);
        bkt1 = 0;  // 重置，开始下一轮扫描
    }
}
```

### 2.2 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `LWCT_BUCKET_COUNT` | 8192 | 哈希桶总数 |
| `LWCT_LOCK_COUNT` | 256 | 锁数量 |
| `LWCT_BUCKETS_PER_LOCK` | 32 | 每锁管理桶数 |
| `LWCT_GC_SCAN_INTERVAL` | 3000ms | 扫描间隔 |
| `LWCT_GC_ENTRY_PER_SCAN` | 1024 | 每轮最大扫描数 |

### 2.3 分桶锁设计

```
桶索引 → 锁索引映射:
lock_idx = bucket_idx / LWCT_BUCKETS_PER_LOCK

例如: bucket 33 → lock 33/32 = 1
```

---

## 3. 删除条件判断

### 3.1 lwct_should_gc()

```c
bool lwct_should_gc(struct lwct_conn *conn)
{
    if (lwct_is_dying(conn))
        return true;  // 已标记删除

    if (!lwct_is_confirmed(conn) && lwct_is_expired(conn, lwct_conn_confirm_tmo))
        return true;  // 未确认且超时

    if (lwct_is_confirmed(conn) && lwct_is_expired(conn, protocol_timeout))
        return true;  // 已确认但协议超时

    return false;
}
```

### 3.2 超时阈值

| 协议 | unreplied_tmo | replied_tmo | established_tmo |
|------|---------------|-------------|----------------|
| TCP | 30s | 180s | 7200s (2h) |
| UDP | 60s | 180s | 300s (5min) |
| ICMP | 10s | - | 30s |

### 3.3 早期 GC (水位触发)

```c
// 当连接数达到 80% 水位时触发
lwct_conn_water_level = (lwct_parameters.lwct_conn_count * 4) / 5;

if (conn_count >= lwct_conn_water_level) {
    early_gc = true;
    // 缩短 ASSURED 连接的 timeout
    if (early_gc && _test_bit(LWCT_ASSURED_BIT, &ct_conn->status) &&
        lwct_should_early_gc(ct_conn, tcp_threshold, udp_threshold)) {
        ct_conn->timeout = 0;  // 下次 GC 即删除
    }
}
```

---

## 4. 引用计数与内存管理

### 4.1 连接结构

```c
struct lwct_conn {
    int32_t refcnt;                      // 引用计数
    uint64_t timeout;                    // 过期时间
    struct lwct_tuple_hash tuplehash[2]; // [0]=原始, [1]=回复
    uint64_t status;                     // 状态位图
    union {
        struct lwct_ext *ext;           // 扩展数据
        uint32_t stats[2];
    };
};
```

### 4.2 引用计数操作

```c
// 增加引用
static inline int lwct_ref_inc(struct lwct_conn *conn) {
    return __atomic_fetch_add(&conn->refcnt, 1, __ATOMIC_RELAXED);
}

// 减少引用
static inline int lwct_ref_dec(struct lwct_conn *conn) {
    return __atomic_fetch_sub(&conn->refcnt, 1, __ATOMIC_RELEASE);
}

// 减少并测试是否为 0
static inline int lwct_ref_dec_and_test(struct lwct_conn *conn) {
    int old = __atomic_fetch_sub(&conn->refcnt, 1, __ATOMIC_RELEASE);
    if (old == 1) {
        return true;  // 可以释放
    }
    return false;
}
```

### 4.3 释放引用

```c
static inline void lwct_conn_put(struct lwct_conn *conn) {
    if (conn && lwct_ref_dec_and_test(conn)) {
        lwct_destroy(conn);  // 实际释放内存
    }
}
```

### 4.4 内存分配

```c
static struct lwct_conn *__lwct_conn_alloc(...) {
    // 检查连接数限制
    prev_conn_count = __atomic_fetch_add(&lwct_conn_count, 1, __ATOMIC_RELAXED);
    if (prev_conn_count >= lwct_get_conn_size()) {
        goto __no_more_conn;
    }

    // 从内存池分配
    conn = memp_malloc(MEMP_LWCT_CONN);
    if (!conn) {
        LWCT_PRINTF_ERROR("malloc MEMP_LWCT_CONN failed\n");
        goto __no_more_conn;
    }

    memset(conn, 0, sizeof(struct lwct_conn));
    conn->refcnt = 1;  // 初始引用计数为 1
    return conn;
}
```

---

## 5. GC 删除流程

### 5.1 双桶删除

```c
if (lwct_should_gc(ct_conn)) {
    // 1. 标记为 DYING
    lwct_set_dying_bit(ct_conn);

    // 2. 确定要删除的两个桶
    if (cur_tuple->tuple.dir == LWCT_DIR_ORIGINAL)
        bkt2 = LWCT_HASH_TO_BKT_INDEX(ct_conn->tuplehash[REPLY].hash);
    else
        bkt2 = LWCT_HASH_TO_BKT_INDEX(ct_conn->tuplehash[ORIGINAL].hash);

    // 3. 获取第二个桶的锁
    lock2 = LWCT_BKT_IDX_TO_LOCK_IDX(bkt2);

    // 4. 如果两个锁相同或获取成功，删除连接
    if (lock2 == lock1 || lwct_ct_single_trylock(lock2) == 0) {
        cdlist_del(&ct_conn->tuplehash[ORIGINAL].node);
        cdlist_del(&ct_conn->tuplehash[REPLY].node);
        lwct_conn_put(ct_conn);  // 引用计数减 1，可能触发释放

        if (lock2 != lock1)
            lwct_ct_single_unlock(lock2);
    } else {
        // 获取第二个锁失败，跳过这次删除
        LWCT_STATICS_INC(g_lwct_stats.gc_reclaim_fail);
    }
}
```

### 5.2 锁获取策略

| 场景 | 策略 |
|------|------|
| `lock2 == lock1` | 同一锁，不需要额外获取 |
| `lock2 != lock1` 且获取成功 | 先解锁 lock1，再获取 lock2，再删除 |
| 获取 lock2 失败 | 放弃本次删除，统计 `gc_reclaim_fail` |

### 5.3 删除失败处理

```c
// 如果无法获取第二个桶的锁，连接不会被删除
// 会留在哈希表中，下一轮 GC 再尝试
// 统计 `gc_reclaim_fail` 计数
```

---

## 6. 潜在问题分析

### 6.1 GC 线程退出无重启

```c
// lwct_core.c:422
(void)sys_time_sleep(tmsv_get_svc_ep(), &tv);
// 如果 sleep 失败，线程直接退出，无重启机制
```

**影响**: GC 线程退出后，连接永远不会删除，连接表会逐渐满。

### 6.2 锁争用可能导致删除延迟

```c
// 如果 lock2 获取失败，连接保留
// 可能导致连接表持续满
```

**影响**: 高并发时，GC 删除可能跟不上新连接创建速度。

### 6.3 引用计数竞态

```c
// refcnt 是 int32_t，不是原子类型
// __atomic_fetch_add/sub 是原子操作，但初始化和赋值不是
conn->refcnt = 1;  // 非原子！
```

**风险**: 如果在多核同时分配连接时出现竞态，可能导致计数错误。

### 6.4 早期 GC 延迟

```c
// 早期 GC 只设置 timeout = 0
// 但删除只在下一轮 GC 才执行
// 如果连接创建速度快，80% 水位可能持续
```

---

## 7. 优化建议

### 7.1 GC 线程监控

```c
// 添加 GC 线程存活监控
if (sys_time_sleep(...) < 0) {
    LWCT_PRINTF_ERROR("GC thread sleep failed, restarting...");
    // 重新创建 GC 线程
}
```

### 7.2 锁获取优化

```c
// 当前: 简单 retry
// 建议: 添加退避策略，避免 CPU 空转
for (int retry = 0; retry < MAX_LOCK_RETRY; retry++) {
    if (lwct_ct_single_trylock(lock2) == 0) break;
    // 短暂让出 CPU
    sched_yield();
}
```

### 7.3 引用计数初始化原子化

```c
// 当前: refcnt = 1 非原子
// 建议: 使用原子初始化
__atomic_store_n(&conn->refcnt, 1, __ATOMIC_RELAXED);
```

### 7.4 批量删除优化

```c
// 当前: 逐个删除
// 建议: 批量标记 DYING 后统一删除
```

---

## 8. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `lwct_core.c` | 297 | `lwct_gc_thread` GC 线程主循环 |
| `lwct_core.c` | 334 | 水位检查 |
| `lwct_core.c` | 369 | 早期 GC 阈值检查 |
| `lwct_core.c` | 373 | `lwct_should_gc` 判断 |
| `lwct_core.c` | 375 | `lwct_set_dying_bit` 标记 |
| `lwct_core.c` | 398 | 从桶中删除 |
| `lwct_core.c` | 400 | `lwct_conn_put` 释放引用 |
| `lwct_core.c` | 965 | `__lwct_conn_alloc` 分配 |
| `lwct.h` | 36 | `struct lwct_conn` 定义 |
| `lwct.h` | 133-188 | 引用计数操作 |
