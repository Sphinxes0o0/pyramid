---
type: synthesis
tags: []
created: 2026-05-28
---
# lwIP pbuf 直接暴露给 NIDS 工程方案

## 1. pbuf 流三阶段

```
ALLOC ────────────────────────────────────────────────────────
  ↓
NIC DMA → pbuf_alloc() → ip4_input(p, inp)   [阶段1: 入口]
                                              ↓
                                     ┌──────────────┐
                                     │ HW checksum  │
                                     │ offload 已完 │
                                     └──────────────┘
                                              ↓
阶段2: ip4_input() 内部 ─────────────────────────────────────
  507: LWIP_HOOK_IP4_INPUT(p, inp)    ← NIDS 插入点A (最早)
  737: lwct_main_hook(p)              (conntrack, 已有)
  743: lwfw ingress_filter(p)         (firewall, 已有)
  790: raw_input(p)                  → raw_pcb recv 回调
                                     → tpacket_recv()  [packet_mmap]
                                     → NIDS raw_pcb recv  [插入点B]
  803/809/815: udp_input / tcp_input / icmp_input
                                              ↓
FREE (出错/丢弃路径) ─────────────────────────────────────────
  pbuf_free(p)                       [阶段3: 释放]
```

**关键结论**：
- `ip4_input` 人口处 pbuf 尚未被任何协议层消费
- `pbuf` 所有权属于 lwIP，直到 `pbuf_free()` 才释放
- `raw_input()` 之后 pbuf 仍归 lwIP，raw recv 回调不许 free

---

## 2. 三处可插入 hook 点

| 位置 | 函数 | 特点 | 推荐度 |
|------|------|------|--------|
| **A** | `LWIP_HOOK_IP4_INPUT` (ip4.c:507) | IP 层最早期，hook 返回1则 lwIP 认为包已被吃 | **最推荐** |
| **B** | `raw_pcb->recv` 注册为 NIDS raw PCB | 挂在 raw_input 循环中，与 packet_mmap 同级 | 次推荐 |
| **C** | `lwct_main_hook` / `lwfw ingress_filter` | 已有 conntrack/firewall，需要合入 | 视已有架构 |

---

## 3. 最小改动方案（伪代码）

### 改动1: lwipopts.h (m57_opts / s32g_opts)

```c
// 在 LWIP_HOOK_IP4_ROUTE 附近添加
#define LWIP_HOOK_IP4_INPUT          nids_hook_ip4_input
// 返回 0 = 继续正常处理; 返回 1 = hook 已接管 pbuf(本方案返回0)
int nids_hook_ip4_input(struct pbuf *p, struct netif *inp);
```

### 改动2: lwip_interface.h

```c
#if defined(LWIP_HOOK_IP4_INPUT)
int nids_hook_ip4_input(struct pbuf *p, struct netif *inp)
{
    // 不返回1! pbuf 继续归 lwIP 处理
    // NIDS 侧 clone 一份自行处理
    nids_dispatch_pbuf(p);   // 内部: pbuf_clone → 送 ring buffer
    return 0;               // 放行，不拦截正常路径
}
#endif
```

### 改动3: 新文件 nids_hook.c（minimal，不改 lwIP 核心）

```c
// os-framework/servers/net/src/nids_hook.c

#include "lwip/pbuf.h"
#include "lwip/raw.h"
#include "core/ringbuffer.h"

static struct ringbuf *nids_rb;   // NIDS ring buffer
static struct raw_pcb *nids_pcb; // NIDS raw PCB (备用方案B)

// 方案A: 被 LWIP_HOOK_IP4_INPUT 调用
void nids_dispatch_pbuf(struct pbuf *p)
{
    if (!nids_rb) return;

    // clone: 复制 payload 数据，NIDS 独立拥有 clone
    struct pbuf *clone = pbuf_clone(PBUF_RAW, PBUF_POOL, p);
    if (!clone) return;

    // 写入 NIDS ring buffer（用户空间 mmap 读取）
    rb_write(nids_rb, clone->payload, clone->tot_len);

    // NIDS 用户空间线程: 从 ring 读出 → 分析 → pbuf_free(clone)
    // (不在此上下文做，避免阻塞 tcpip_thread)
}

// 方案B(备用): raw PCB recv 回调
// - 注册协议号 = 0xFF (所有协议) 或特定协议
// - 由 nids_raw_recv() 返回 0 (不 eaten)，lwIP 继续处理
static u8_t nids_raw_recv(void *arg, struct raw_pcb *pcb,
                          struct pbuf *p, const ip_addr_t *addr)
{
    nids_dispatch_pbuf(p);
    return 0; // 不吃包，lwIP 继续
}

void nids_hook_init(struct ringbuf *rb)
{
    nids_rb = rb;
    nids_pcb = raw_new(IP_PROTO_RAW);
    if (nids_pcb) {
        raw_recv(nids_pcb, nids_raw_recv, NULL);
        // bind 到 INADDR_ANY 监听所有接口
    }
}
```

---

## 4. 核心设计决策

### pbuf Clone 必要性

```
NIDS hook 不返回1 (不接管所有权) → 必须 clone
原因: lwIP 同一 pbuf 在 ip4_input 结束后继续被 raw_input/UDP/TCP 引用
若 NIDS 直接持有原 pbuf 指针:
  - lwIP pbuf_free(p) 后 → dangling pointer
  - NIDS 分析时崩溃
```

### Clone 开销可接受

```c
pbuf_clone(PBUF_RAW, PBUF_POOL, p)
- PBUF_POOL: 从内存池分配，零拷贝 copy-up
- tot_len 小 (MTU 1500B typical)
- 路径在 tcpip_thread，非数据面关键路径
```

### NIDS vs packet_mmap 并存

`tpacket_recv` 和 `nids_raw_recv` 可同时注册为 raw PCB recv 回调（raw.c 循环遍历所有 pcbs），两者独立收到同一 pbuf，各自 clone 自己的副本。

---

## 5. 改动量汇总

| 文件 | 改动类型 | 行数 |
|------|----------|------|
| `m57_opts/lwipopts.h` | 添加 `#define LWIP_HOOK_IP4_INPUT` | 1 |
| `include/nio/lwip_interface.h` | 添加 `nids_hook_ip4_input()` 实现 | ~10 |
| `servers/net/src/nids_hook.c` | **新增文件** | ~60 |
| `servers/net/CMakeLists.txt` | 添加 nids_hook.c | 1 |

**零改动**: lwIP 核心 (ip4.c / raw.c / pbuf.c)

---

## 6. 调用时序（关键路径）

```
NIC DMA IRQ
  → netif_rx() / input_thread()
    → tcpip_inpkt(pbuf, netif, ip4_input)
      → ip4_input(p, inp)
        → LWIP_HOOK_IP4_INPUT(p, inp)     [NIDS: clone + ring]
        → [lwct_main_hook]                 (已有)
        → [lwfw ingress_filter]           (已有)
        → raw_input(p)
          → tpacket_recv() [packet_mmap]  (已有)
          → nids_raw_recv()  [NIDS]        (备用路径B)
        → udp_input / tcp_input / ...
          → pbuf_free(p)
```
