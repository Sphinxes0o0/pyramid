---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW LWCT GC Analysis

## 定义

LWCT GC (Garbage Collection) 线程负责定期扫描连接表，删除超时或过期的连接条目，释放内存资源。GC 线程每 3 秒扫描一次，最多扫描 1024 个条目后主动让出 CPU。

## GC 线程架构

```c
void lwct_gc_thread(void *arg) {
  lwct_get_hash_tbl(&ct_hash_tbl, &bkt_size);

  while (1) {
    // 1. 扫描所有桶
    while (bkt1 < bkt_size) {
      lock1 = LWCT_BKT_IDX_TO_LOCK_IDX(bkt1);
      if (lwct_ct_single_trylock(lock1) == 0) {
        cdlist_iter_entry_safe(cur_tuple, tmp_tuple, &ct_hash_tbl[bkt1], node) {
          if (lwct_should_gc(ct_conn)) {
            lwct_set_dying_bit(ct_conn);
            cdlist_del(&ct_conn->tuplehash[ORIGINAL].node);
            cdlist_del(&ct_conn->tuplehash[REPLY].node);
            lwct_conn_put(ct_conn);
          }
        }
        lwct_ct_single_unlock(lock1);
      }

      if (scanned >= lwct_parameters.lwct_gc_entry_per_scan) {
        yield();  // 让出 CPU
      }
    }

    // 2. 等待下次扫描
    sys_time_sleep(tv);
    bkt1 = 0;
  }
}
```

## 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `LWCT_BUCKET_COUNT` | 8192 | 哈希桶总数 |
| `LWCT_LOCK_COUNT` | 256 | 锁数量 |
| `LWCT_BUCKETS_PER_LOCK` | 32 | 每锁管理桶数 |
| `LWCT_GC_SCAN_INTERVAL` | 3000ms | 扫描间隔 |
| `LWCT_GC_ENTRY_PER_SCAN` | 1024 | 每轮最大扫描数 |

## 删除条件

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

## 早期 GC (水位触发)

```c
// 当连接数达到 80% 水位时触发早期 GC
lwct_conn_water_level = (lwct_parameters.lwct_conn_count * 4) / 5;

if (conn_count >= lwct_conn_water_level) {
  early_gc = true;
  if (early_gc && _test_bit(LWCT_ASSURED_BIT, &ct_conn->status) &&
      lwct_should_early_gc(ct_conn, tcp_threshold, udp_threshold)) {
    ct_conn->timeout = 0;  // 下次 GC 即删除
  }
}
```

## 引用计数与内存管理

```c
// 引用计数操作 (原子)
static inline int lwct_ref_inc(struct lwct_conn *conn) {
  return __atomic_fetch_add(&conn->refcnt, 1, __ATOMIC_RELAXED);
}

static inline int lwct_ref_dec_and_test(struct lwct_conn *conn) {
  int old = __atomic_fetch_sub(&conn->refcnt, 1, __ATOMIC_RELEASE);
  return old == 1;  // 可以释放
}

static inline void lwct_conn_put(struct lwct_conn *conn) {
  if (conn && lwct_ref_dec_and_test(conn)) {
    lwct_destroy(conn);  // 实际释放内存
  }
}
```

## 已知问题

| 问题 | 严重性 | 说明 |
|------|--------|------|
| GC 线程退出无重启 | P1 | sleep 失败后线程直接退出，连接永不删除 |
| 锁争用导致删除延迟 | P2 | 获取第二个桶锁失败时跳过删除 |
| 引用计数初始化非原子 | P1 | `conn->refcnt = 1` 非原子初始化 |

## 相关概念

- [[entities/linux/lwfw/lwfw-lwct]] — LWCT 模块整体
- [[entities/linux/lwfw/lwfw-lwct-interaction]] — LWFW 与 LWCT 交互
- [[entities/linux/lwfw/lwfw-stats]] — GC 统计 (gc_reclaim_fail)
