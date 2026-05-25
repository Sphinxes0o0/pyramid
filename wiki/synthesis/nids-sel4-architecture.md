---
type: synthesis
tags: [nids, sel4, safeos, architecture, security, af-packet, vlan-mirror]
created: 2026-05-25
sources: [domain.json, nids_conf.yaml]
---

# NIDS on SafeOS/seL4 — 实际部署架构 (已验证)

> 修正日期: 2026/05/25
> 重要更正: 之前的分析将 NIDS 误判为独立 VM + 共享内存 ring，实际为单 VM 内的 native daemon

---

## 1. 部署模式：单 VM + Native Daemon

**事实 (已验证)**:
- NIDS 是主 domain 内的 native daemon 进程 (`/usr/bin/nids`)
- 52 个服务之一，启动依赖 `nlog_manager`
- **不是独立 VM**，无独立的 seL4 虚拟地址空间
- **不使用共享内存 ring**，抓包方式为 `AF_PACKET` on `PFE.VLAN1`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           seL4 Microkernel                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        主 Domain (Main VM)                            │  │
│  │                                                                      │  │
│  │   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │  │
│  │   │  NSv (lwIP)  │    │    NIDS      │    │  其他 50 个  │          │  │
│  │   │              │    │  (native     │    │   服务        │          │  │
│  │   │  tcpip_thread│    │   daemon)    │    │              │          │  │
│  │   │  LWFW        │    │              │    │              │          │  │
│  │   │              │    │ CaptureThread│    │              │          │  │
│  │   │              │    │ WorkerThread │    │              │          │  │
│  │   │              │    │ DetectionEng │    │              │          │  │
│  │   └──────────────┘    └──────────────┘    └──────────────┘          │  │
│  │          │                   │                                         │  │
│  │          │ AF_PACKET         │ AF_PACKET                               │  │
│  │          │ (VLAN1 mirror)    │ (VLAN1 mirror)                          │  │
│  │          ▼                   ▼                                         │  │
│  │   ┌──────────────────────────────────────────────────────────────┐   │  │
│  │   │                    PFE Hardware                               │   │  │
│  │   │  PFE.VLAN1 ──→ VLAN mirror port ──→ AF_PACKET socket         │   │  │
│  │   └──────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  App VM 1   │  │  App VM N   │  │  RTOS VMs  │  │  其他 VM    │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 与之前分析的关键差异

| 错误假设 (之前) | 正确事实 (现在) |
|----------------|----------------|
| NIDS 是独立 VM | NIDS 是 native daemon，同主 domain |
| 使用 elem_ring 共享内存 | 使用 AF_PACKET on PFE.VLAN1 |
| NSv IPC 传递镜像流量 | VLAN mirror port 镜像，无需 IPC |
| 跨 VM 隔离 + IPC 开销 | 同进程内，无 IPC 开销 |
| 需配置 CMA capability | 无需特殊内存配置 |

---

## 2. 流量路径

### 2.1 抓包机制：AF_PACKET on PFE.VLAN1

```
┌─────────────────────────────────────────────────────────────────┐
│                         PFE Hardware                            │
│                                                                 │
│   PFE.VLAN1 ──→ VLAN mirror port ──→ AF_PACKET socket          │
│                         │                                       │
│                         │ (硬件镜像，不涉及 CPU)                  │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    NIDS Process                                 │
│                                                                 │
│   AF_PACKET socket                                              │
│        │                                                        │
│        ▼                                                        │
│   CaptureThread                                                 │
│        │ PacketBuffer                                           │
│        ▼                                                        │
│   WorkerThread                                                  │
│        │ DecodeResult                                           │
│        ▼                                                        │
│   DetectionEngine (AC)                                          │
│        │ AlertEvent                                            │
│        ▼                                                        │
│   EventEngine ──→ LWFW SOA Report                              │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**:
- NIDS 通过标准 Linux `AF_PACKET` raw socket 接收镜像流量
- `capture_backend: "af"` 表示使用 AF_PACKET 后端
- PFE.VLAN1 的 VLAN mirror port 由硬件镜像到 socket，无需 NSv 参与
- 流量镜像路径: PFE.VLAN1 → 硬件镜像 → AF_PACKET socket → NIDS

### 2.2 与 NSv/lwIP 的关系

**同域进程，无 IPC**:
- NIDS 和 NSv 都在主 domain 内，是平等的 native 进程
- NIDS 通过 VLAN mirror 获取流量，不依赖 NSv 转发
- NSv 不知道 NIDS 的存在（无 IPC channel）
- 两者的唯一关联是共享 PFE.VLAN1 的镜像流量

```
PFE.VLAN1
    │
    ├──→ NSv (通过自己的 AF_PACKET 或 netif)
    │
    └──→ NIDS (通过 AF_PACKET 镜像)

两者互不感知，无共享内存，无 IPC
```

---

## 3. 资源约束

### 3.1 内存限制

| 状态 | RSS 上限 | 说明 |
|------|----------|------|
| **stop** (暂停) | 80 MB | 健康检查触发 pause 时的上限 |
| **resume** (恢复) | 56 MB | pause 后自动 resume 的上限 |

**内存分布 (典型值)**:

```yaml
nids_memory:
  text: 2MB              # 代码 + 只读数据
  data: 4MB              # 全局变量 + 堆
  stack: 512KB × 2       # CaptureThread + WorkerThread
  packet_pool:
    small: 2048 × 256B  = 512KB
    std:    2048 × 2KB   = 4MB
    large:   512 × 16KB  = 8MB
  detection_engine:
    port_group_index: 64KB
    content_matcher:   1MB    # AC automaton
    detection_filter:  160KB
    port_scan_tables:  512KB
  event_queue: 1MB
  spsc_queue:   2MB
  # 总计: ~23MB 典型静态 footprint
```

### 3.2 CPU 限制

| 状态 | CPU 上限 | seL4 板基准 |
|------|----------|-------------|
| **stop** (暂停) | 30% | idle: 1-3% |
| **resume** (恢复) | 16% | flood: 15-50% |

**CPU 分布 (典型值)**:

| 线程 | CPU 占比 | 说明 |
|------|----------|------|
| CaptureThread | ~5% | AF_PACKET 收包 |
| WorkerThread | ~8% | 检测逻辑 |
| EventEngine | ~2% | 事件上报 |
| **总计** | ~15% | 正常流量下 |

### 3.3 监控：Health Monitor 自动 pause/resume

```
Health Monitor 监控流程:

CPU > 30% 持续 N 秒
    │
    ▼
pause NIDS process
    │
    ▼
RSS 限制 80MB → 等待内存回收
    │
    ▼
CPU 下降至 16% 以下
    │
    ▼
resume NIDS process
    │
    ▼
RSS 上限调整为 56MB
```

---

## 4. seL4 在此架构下的安全优势

虽然 NIDS 不是独立 VM，seL4 仍提供以下安全保证：

### 4.1 形式化验证的隔离保证

| 层级 | 说明 |
|------|------|
| **seL4 微内核** | ~9K LOC C，形式化验证，数学证明正确性 |
| **主 Domain** | 所有 native 进程在同一 VM 内，seL4 保护 VM 边界 |
| **进程隔离** | NIDS 是标准 Linux 进程，受 Linux MMU 保护 |
| **Capability** | seL4 capability 控制 NIDS 对硬件的访问权限 |

**关键**: seL4 保证 VM 间隔离，但 NIDS 和 NSv 是同 VM 内的不同进程，隔离依赖 Linux 进程边界。

### 4.2 Capability 权限控制

```
NIDS process 的 seL4 capabilities:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ 可访问: PFE.VLAN1 的 AF_PACKET socket
✓ 可访问: nlog_manager (日志服务)
✓ 可访问: SOA 接口 (LWFW 联动)
✗ 不可访问: PFE 硬件寄存器 (需通过 kernel driver)
✗ 不可访问: 其他 VM 的内存
✗ 不可访问: HSE 密钥存储
```

### 4.3 与独立 VM 架构的对比

| 方面 | 独立 VM 方案 (原错误分析) | Native Daemon 方案 (实际) |
|------|--------------------------|---------------------------|
| **隔离级别** | seL4 VM 隔离 | Linux 进程隔离 |
| **IPC** | seL4 IPC ~300-1500ns/包 | 无 (VLAN mirror) |
| **TCB** | seL4 + NIDS VM | seL4 + Linux kernel |
| **配置复杂度** | 高 (VM + capability) | 低 (标准进程) |
| **故障影响** | NIDS 崩仅影响自己 VM | NIDS 崩可能影响主 VM |
| **seL4 验证范围** | 仅 NIDS VM | 整个主 VM |

---

## 5. LWFW 联动接口

NIDS 与 LWFW 通过 SOA 接口通信:

```
NIDS DetectionEngine ──→ AlertEvent ──→ SOA JSON ──→ LWFW Agent
                                                        │
                                                        ▼
                                               lwfw_policy_insert_rule()
                                                        │
                                                        ▼
                                               inactive_policy ↔ policy 原子切换
```

**阻断指令格式**:

```yaml
block_rule:
  protocol: SOA JSON over seL4 IPC
  fields:
    - action: "block" | "rate_limit" | "log_only"
    - src_ip: string
    - dst_ip: string
    - src_port: uint16 | "any"
    - dst_port: uint16 | "any"
    - proto: "tcp" | "udp" | "icmp" | "any"
    - duration_sec: uint32 (0 = 永久)
    - reason: string (SID/gid/msg)
```

**响应时间目标**:
- NIDS 检测 → LWFW 生效: < 10ms
- LWFW 反馈 → NIDS: < 1s (轮询)

---

## 6. 总结

| 决策点 | 实际方案 | 说明 |
|--------|----------|------|
| **部署模式** | Native daemon (单 VM) | 52 个服务之一 |
| **抓包方式** | AF_PACKET on PFE.VLAN1 | capture_backend: "af" |
| **流量来源** | VLAN mirror port | 硬件镜像，无需 IPC |
| **与 NSv 关系** | 同域进程，无 IPC | 共享 VLAN1 镜像 |
| **RSS 限制** | 80MB (stop) / 56MB (resume) | Health monitor 控制 |
| **CPU 限制** | 30% (stop) / 16% (resume) | seL4 板 idle 1-3%, flood 15-50% |

**原分析错误根源**:
- 假设 NIDS 使用 NSv 的 elem_ring 架构
- 误判 NIDS 为独立安全 VM
- 实际上 NIDS 是标准 native 进程，使用标准 AF_PACKET

**seL4 安全收益**:
- VM 间隔离保护其他 VM 不受主 VM 影响
- Capability 机制限制 NIDS 的硬件访问权限
- 形式化验证的内核保证底层隔离正确性

**风险与缓解**:

| 风险 | 缓解措施 |
|------|----------|
| NIDS 崩溃影响主 VM | Health monitor 自动重启 |
| NSv 异常导致 VLAN mirror 中断 | NIDS 检测到流量中断后告警 |
| 内存不足影响其他服务 | RSS 限制 + Health monitor pause |
