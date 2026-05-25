# lwct 连接跟踪模块分析

> 代码路径: `libs/util_libs/liblwfw/src/lwct/`
> 头文件: `libs/util_libs/liblwfw/include/lwct/`

---

## 1. 整体架构

```
包进入 lwIP
    │
    ├─► lwct_in(pbuf)           ← 主 hook: 查找/创建连接
    │     ├─ lwct_hash_tuple()    ← 计算五元组哈希
    │     ├─ lwct_lookup_conn()  ← 查找连接
    │     │     └─ 没找到? → lwct_new_conn() ← 创建新连接
    │     ├─ lwct_update_timeout() ← 更新超时
    │     └─ pbuf->_lwct = conn_ptr | state  ← 绑定到 pbuf
    │
    ├─► lwct_confirm(pbuf)       ← 确认连接 (tcp 3-way 后)
    │
    └─► lwct_in() 返回后，lwfw 使用 p->_lwct 判断连接状态
```

---

## 2. 连接状态

### 2.1 连接级别状态 (per-connection)

```c
typedef enum {
  LWCT_SEEN_REPLY_BIT  = 0,  // 已看到双向流量
  LWCT_ASSURED_BIT     = 1,  // 连接表未满时不可删除
  LWCT_CONFIRMED_BIT   = 2,  // 发起包已离开本机
  LWCT_DYING_BIT       = 3,  // 正在删除中
} lwct_status_t;
```

### 2.2 包级别状态 (per-pbuf, 存于 pbuf->_lwct 低3位)

```c
typedef enum {
  LWCT_UNKNOWN         = 0,  // 未建立连接
  LWCT_ESTABLISHED     = 1,  // 已知连接
  LWCT_NEW             = 2,  // 新连接
  LWCT_IS_REPLY        = 3,  // 标志: 是回复方向
  LWCT_ESTABLISHED_REPLY = 4, // ESTABLISHED + IS_REPLY
} lwct_state_t;
```

### 2.3 状态转换

```
TCP:
  NEW → (看到 SYN) → ESTABLISHED
  NEW → (看到 SYN+ACK) → REPLIED → ESTABLISHED
  ESTABLISHED → (看到 RST) → 删除连接

UDP:
  NEW → (看到双向) → REPLIED → ESTABLISHED
```

---

## 3. 哈希表结构

### 3.1 全局结构

```c
struct lwct_conn_table {
  struct cdlist *conn_lists;   // 哈希桶数组
  sys_mutex_t *bkt_locks;      // 桶锁数组
};

#define LWCT_BUCKET_COUNT   (LWCT_LOCK_COUNT * LWCT_BUCKETS_PER_LOCK)
// 默认: 256 locks × 32 buckets/lock = 8192 buckets
```

### 3.2 哈希算法

```c
// lwct_hash.h
LWCT_HASH_INIT_VALUE = 0xdeadbeaf

lwct_hash_tuple(tuple)
  ├─ jhash_3words(src_ip, dst_ip, ...)
  ├─ jhash_3words(src_port, dst_port, proto)
  └─ return hash ^ (hash >> LWCT_HASH_MASK_SHIFT)
```

### 3.3 连接条目 (双向链表)

每个连接在哈希表中占 **两个** 槽位:

```c
struct lwct_conn {
  lwct_tuple_hash tuplehash[2];  // [0]=原始方向, [1]=回复方向
  // 两个方向共享同一个 conn 结构
};

struct lwct_tuple_hash {
  struct lwct_tuple tuple;
  uint32_t hash;
  struct cdlist node;  // 链表节点
};
```

---

## 4. 连接生命周期

### 4.1 创建 (`lwct_new_conn`)

```c
lwct_conn *lwct_new_conn(tuple, dir)
  ├─ conn = malloc(sizeof(lwct_conn))
  ├─ conn->tuplehash[ORIGINAL] = {tuple, hash}
  ├─ conn->tuplehash[REPLY]    = {inverse_tuple, inverse_hash}
  ├─ conn->status = 0
  ├─ conn->timeout = now + unreplied_tmo
  ├─ atomic conn_count++
  ├─ cdlist_add_tail(&conn_lists[bucket], &conn->tuplehash[ORIGINAL].node)
  └─ cdlist_add_tail(&conn_lists[reply_bucket], &conn->tuplehash[REPLY].node)
```

### 4.2 确认 (`lwct_confirm`)

```c
// TCP 三次握手后调用
lwct_confirm(pbuf, dir)
  ├─ conn = pbuf->_lwct & PTR_MASK
  ├─ _set_bit(CONFIRMED_BIT, &conn->status)
  └─ atomic conn_count 实际连接数++
```

### 4.3 更新超时

```c
lwct_update_timeout_acct(conn, pbuf, state, new_timeout, update_acct)
  ├─ conn->timeout = now + new_timeout
  ├─ if (update_acct) lwct_acct_add(conn, dir, packets, bytes)
  └─ if (state == ORIGINAL && !SEEN_REPLY)
       _set_bit(SEEN_REPLY_BIT)
       conn->timeout = now + replied_tmo
```

### 4.4 TCP 特殊处理

```c
lwct_handle_tcp_packet(conn, pbuf, state)
  ├─ if (TCP_RST) conn->timeout = 0  → gc 下次删除
  ├─ if (ASSURED) → timeout = established_tmo (长超时)
  ├─ else if (SEEN_REPLY) → timeout = replied_tmo, set ASSURED
  └─ else → timeout = unreplied_tmo (短超时)
```

---

## 5. 垃圾回收 (GC)

### 5.1 GC 线程

```c
void lwct_gc_thread()
{
  while (1) {
    sys_time_sleep(gc_scan_interval);  // 默认 3s

    for (bkt = bkt1; bkt < bkt_size; bkt++) {
      lock(bucket);
      cdlist_iter_entry_safe(cur_tuple, tmp, bucket, node) {
        ct_conn = lwct_tuplehash_to_conn(cur_tuple);

        // 达到水位线? 记录远端 IP
        if (conn_count >= water_level && first_time)
          record_remote_ip();

        // 设置 DYING bit
        if (lwct_should_gc(ct_conn)) {
          lwct_set_dying_bit(ct_conn);

          // 双桶删除
          del_from_list(orig_bucket);
          del_from_list(reply_bucket);
          conn_put(ct_conn);  // 引用计数减1，可能释放
        }
      }
      unlock(bucket);

      // 每轮扫描上限
      if (scanned >= gc_entry_per_scan)  // 默认 1024
        yield();
    }
  }
}
```

### 5.2 删除条件

```c
lwct_should_gc(conn)
  ├─ if (DYING) return true
  ├─ if (!CONFIRMED && age > conn_confirm_tmo) return true
  ├─ if (CONFIRMED && age > protocol_timeout) return true
  └─ return false
```

### 5.3 早期 GC (水位触发)

```c
// 当连接数达到 80% 水位时触发早期 GC
if (conn_count >= lwct_conn_water_level) {
  // 对 ASSURED 连接缩短超时
  if (age > tcp_threshold_45%)
    conn->timeout = 0;  // 下次 GC 即删除
}
```

### 5.4 远端 IP 记录

```c
// 水位触发时，记录出现次数最多的远端 IP/Port
static void lwct_update_remote_ip(tuphash)
  ├─ 检查是否已记录
  ├─ 是 → appear_counts++
  └─ 否 → 添加新记录 (最多 LWCT_MAX_REMOTE_IP=16)

static void lwct_record_remote_ip()
  └─ qsort 按 appear_counts 降序
     └─ 打印 top 5 到日志
```

---

## 6. 与 lwfw 的交互

### 6.1 绑定到 pbuf

```c
// lwct_in() 末尾
pbuf->_lwct = (uintptr_t)conn | lwct_state;

// lwfw 使用
pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);
```

### 6.2 lwfw 规则匹配

```c
// lwfw.c:check_rule()
if ((rule->flags & LWFW_RULE_FLAGS_CT_STATE) &&
    rule->ct_state != info->ct_state) {
  return false;  // 连接状态不匹配
}
```

### 6.3 未跟踪包的兜底

```c
// lwfw.c:ip4_filter()
if (lwct_enable == 1 && !p->_lwct) {
  // 未建立连接的包
  if (ct_oot_action == PASS)
    return ERR_OK;  // 放行
}
```

---

## 7. 协议扩展

### 7.1 扩展机制

```c
struct lwct_ext_type {
  uint16_t id;          // LWCT_EXT_ACCT 等
  uint16_t len;         // 扩展数据长度
  uint16_t align;       // 对齐要求
  void (*destroy)(lwct_conn*);
};

void *lwct_ext_add(lwct_conn *ct, lwct_ext_id id)
  ├─ realloc(ct->ext, newlen)
  ├─ 在 ext 末尾分配新扩展空间
  └─ 返回扩展数据指针
```

### 7.2 计费扩展

```c
struct lwct_conn_acct {
  struct lwct_conn_counter {
    uint64_t packets;
    uint64_t bytes;
  } counter[2];  // [0]=ORIGINAL, [1]=REPLY
};
```

---

## 8. 配置参数

```c
lwct_parameters {
  lwct_gc_scan_interval:    3000 ms    // GC 扫描间隔
  lwct_gc_entry_per_scan:   1024        // 每轮扫描上限
  lwct_conn_confirm_tmo:    30 s        // 未确认连接超时
  lwct_bucket_count:        8192        // 哈希桶数量
  lwct_lock_count:          256         // 锁数量
  lwct_conn_count:          8192        // 最大连接数
  lwct_tcp_unreplied_tmo:   180 s       // TCP 未回复超时
  lwct_tcp_replied_tmo:     180 s       // TCP 已回复超时
  lwct_tcp_established_tmo: 10800 s     // TCP 已建立超时
  lwct_udp_unreplied_tmo:   60 s        // UDP 未回复超时
  lwct_udp_replied_tmo:     60 s        // UDP 已回复超时
  lwct_udp_established_tmo:  10800 s     // UDP 已建立超时
  lwct_icmp_unreplied_tmo:  10 s        // ICMP 超时
  lwct_icmp_replied_tmo:    10 s        // ICMP 回复超时
}
```

---

## 9. 已知问题

### 9.1 位操作非原子

```c
// lwct_common.h:277-342
static inline void ___set_bit(nr, addr) {
  unsigned long mask = BIT(nr);
  unsigned long *p = BIT_WORD(nr) + addr;
  *p |= mask;  // 非原子操作！
}
```

多个线程同时操作同一连接的 status 位时可能丢失更新。lwct 本身是单线程（GC 在独立线程），但 lwfw 调用 lwct_in 时数据包处理可能并发。

### 9.2 引用计数问题

```c
// lwct_core.c:400
lwct_conn_put(ct_conn);
if (refcnt == 0)
  free(ct_conn);
```

引用计数使用 `atomic` 操作，但 conn 结构分配使用 `malloc`/`free`。在高并发场景下，频繁的内存分配/释放可能造成碎片化。

### 9.3 ICMP 处理简化

ICMP 只有一种超时时间（`lwct_icmp_unreplied_tmo`），没有区分 query/response。ping 的 request 和 reply 被视为同一连接，reply 超时才删除。

### 9.4 UDP 双向确认延迟

UDP "双向确认"机制依赖 lwct 看到两个方向的包才设置 SEEN_REPLY。如果单向流量后超时，连接被删除，但应用可能还在等待回复。
