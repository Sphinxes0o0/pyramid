---
type: synthesis
tags: [nids, sel4, safeos, architecture, security, vm, lwip, lwfw]
created: 2026-05-25
sources: [safeos-lwip-deep-analysis, safeos-source-analysis, safeos-vdf-analysis, nids-current-architecture]
---

# NIDS 在 seL4/SafeOS 微内核架构下的部署分析

> 分析日期: 2026/05/25
> 基于: safeos-lwip-deep-analysis, safeos-source-analysis, safeos-vdf-analysis, nids-current-architecture

---

## 1. seL4 下的 NIDS 部署架构对比

### 方案 A: NIDS 作为独立 VM

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           seL4 Microkernel                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  NIDS VM    │  │    NSv VM   │  │  App VM 1   │  │  App VM N   │    │
│  │             │  │  (lwIP+FW)  │  │             │  │             │    │
│  │ CaptureThread│  │ nic_rx_thr  │  │             │  │             │    │
│  │ WorkerThread │  │ tcpip_thread│  │             │  │             │    │
│  │ DetectionEng │  │ LWFW        │  │             │  │             │    │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘  └─────────────┘    │
│         │                 │                                              │
│         │ seL4 IPC        │ seL4 IPC                                     │
│         │ (elem_ring)     │ (badge=pid)                                  │
└─────────┼─────────────────┼──────────────────────────────────────────────┘
          │                 │
          │    ┌────────────┘
          │    │ CMA/elem_ring (共享内存)
          ▼    ▼
   ┌─────────────────────────────────┐
   │      NIC Driver (PFE)           │
   └─────────────────────────────────┘
```

**优点**:
- **强隔离**: NIDS 崩溃不影响 NSv 和其他 VM
- **独立重启**: 可单独热重启 NIDS 而不影响网络服务
- **形式化验证边界清晰**: TCB 仅包含 seL4 + NIDS VM
- **权限最小化**: seL4 Capability 仅为 NIDS 分配所需权限

**缺点**:
- **IPC 开销**: 包从 NSv 到 NIDS 需跨 VM 边界，elem_ring 传递 + seL4 notification
- **延迟增加**: 检测路径增加 ~150-710ns × 2 (NSv→NIDS + NIDS→响应)
- **内存重复**: 每个 VM 需独立内存预算
- **共享数据路径复杂**: 流量镜像需 NSv 主动复制到 NIDS

### 方案 B: NIDS 嵌入 NSv

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           seL4 Microkernel                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         NSv VM                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │  nic_rx_thread / tcpip_thread                                   │ │  │
│  │  │         │                                                       │ │  │
│  │  │         ▼                                                       │ │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │ │  │
│  │  │  │ LWFW     │→│ NIDS     │→│ lwIP     │→│ packet   │        │ │  │
│  │  │  │ Ingress  │  │ Detection│  │ Core     │  │ mmap     │        │ │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                         │
│  │  App VM 1   │  │  App VM N   │  │ NIC Driver  │                         │
│  └─────────────┘  └─────────────┘  └─────────────┘                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**优点**:
- **零 IPC**: 检测路径在同一进程内，函数调用无 seL4 IPC 开销
- **共享内存**: 直接访问 CMA buffer，无数据复制
- **低延迟**: 检测延迟仅增加 NIDS 处理时间 (~10-50μs)
- **简单部署**: 无需配置 VM 间共享内存和 capability 授权

**缺点**:
- **TCB 扩大**: seL4 形式化验证范围需包含 NIDS 代码
- **隔离降级**: NIDS 漏洞可能影响 NSv 稳定性
- **单点故障**: NIDS 崩溃可能导致 NSv 崩溃
- **维护复杂度**: NIDS 和 NSv 耦合，升级需整体考虑

### 方案对比矩阵

| 维度 | 方案 A (独立 VM) | 方案 B (嵌入 NSv) |
|------|------------------|-------------------|
| **隔离性** | ✅ 进程级隔离 | ⚠️ 同一进程内 |
| **IPC 延迟** | ❌ ~300-1500ns/包 | ✅ 零 IPC |
| **数据复制** | ❌ elem_ring 传递需拷贝 | ✅ 直接访问 pbuf |
| **TCB 大小** | ✅ NIDS TCB 独立 | ⚠️ 需纳入 NSv TCB |
| **内存开销** | ❌ 额外 VM 内存预算 | ✅ 共享 NSv 内存 |
| **部署复杂度** | ❌ 需配置 VM + capability | ✅ 单一组件 |
| **可维护性** | ✅ 独立升级 | ⚠️ 耦合升级 |
| **故障影响** | ✅ NIDS 崩仅影响自己 | ❌ 可能拖垮 NSv |
| **形式化验证** | ✅ 范围小易验证 | ⚠️ 范围大难验证 |

**结论**: 推荐**方案 A (独立 VM)**，尽管 IPC 开销较高，但安全收益（隔离性 + TCB 最小化）在车载安全场景下是首要考量。

---

## 2. 数据路径分析

### 2.1 镜像流量从 NSv 到 NIDS VM

SafeOS 的 NSv 使用 `elem_ring` 进行 NIC 驱动与 NSv 间的零拷贝传递:
- `used_rx_buf_ring`: NIC → NSv 已收包
- `empty_rx_buf_ring`: NSv → NIC 提供空 buffer

**将流量镜像到 NIDS VM 的可行路径**:

```
路径 1: 额外 elem_ring (推荐)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NIC 驱动 → used_rx_buf_ring → NSv nic_rx_thread
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
                   原始处理路径          elem_ring_put(nids_rx_ring, elem)
                                             │
                                             │ seL4 notification (nic_rx_ntfn)
                                             ▼
                                      NIDS VM nic_rx_thread
                                             │
                                             ▼
                                      NIDS CaptureThread
```

**关键问题**:
- NSv 需在 `rx_callback` 后同步复制 elem 到 NIDS ring
- 复制开销: 每个包 ~O(1)，主要是 PA 指针写入
- seL4 notification 延迟: ~150-710ns

**路径 2: packet_mmap ring 复制**
- NSv 的 `packet_mmap.c` 在 `tpacket_recv()` 中已有 ring buffer 写入
- 可在写入原始 ring 后，异步复制一份到 NIDS VM 的 ring
- 问题: 引入额外 memcpy (~100-200ns/包)

**路径 3: AF-PACKET 原始 socket**
- NSv 开启 `AF_PACKET` raw socket 监听
- NIDS VM 通过 VNET_OVER_IPC 接收镜像流量
- 问题: 引入完整收包路径，不适合高速场景

### 2.2 seL4 IPC 开销对检测延迟的影响

**seL4 IPC 延迟实测范围** (来自 safeos-lwip-deep-analysis):
- `sel4_signal` + `seL4_Recv`: ~150-710ns/msg
- 单次 RX 包从 NIC 到 NSv: 约 ~6.4μs (含协议栈处理)

**NIDS 检测链路延迟预算**:

| 阶段 | 延迟 (方案 A) | 延迟 (方案 B) |
|------|---------------|---------------|
| NIC DMA → elem_ring | ~0 | ~0 |
| NSv nic_rx_thread | ~150-710ns | ~150-710ns |
| **跨 VM elem_ring 复制** | **~300-1500ns** | N/A |
| NIDS CaptureThread 接收 | ~150-710ns | ~0 (函数调用) |
| ProtocolDecoder | ~1-5μs | ~1-5μs |
| DetectionEngine (AC) | ~5-20μs | ~5-20μs |
| EventEngine | ~1-2μs | ~1-2μs |
| **总计** | **~8-30μs** | **~7-28μs** |

**结论**: 跨 VM IPC 开销 (~1-1.5μs) 在 P99 < 100ms 的目标下可接受，不构成瓶颈。

### 2.3 PFE 硬件分流在 VM 架构下的限制

**PFE (Packet Forwarding Engine) 能力**:
- 硬件 TCP/IP 处理
- VLAN 分发
- QoS 队列管理
- HSA (Hardware Security Accelerator) 集成

**VM 架构下的限制**:

```
┌─────────────────────────────────────────────────────────────────┐
│                        NXP S32G                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      PFE Hardware                         │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │  │
│  │  │ PFE_Enet │  │   HSE   │  │   HSA   │  │  RSS    │    │  │
│  │  │  (1G)   │  │(加密引擎)│  │(安全加速)│  │(分流)   │    │  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │  │
│  └───────┼────────────┼────────────┼────────────┼──────────┘  │
│          │            │            │            │               │
│          │ DMA       │            │            │               │
│          ▼            │            │            │               │
│  ┌──────────────┐    │            │            │               │
│  │  CMA (96MB)  │◄───┘            │            │               │
│  │ elem_ring x4 │                 │            │               │
│  └──────┬───────┘                 │            │               │
│         │                         │            │               │
└─────────┼─────────────────────────┼────────────┼───────────────┘
          │                         │            │
          │ seL4 IPC                │            │
          ▼                         │            │
  ┌───────────────┐                 │            │
  │  NSv VM       │◄────────────────┘            │
  │  (lwIP+LWFW)  │                              │
  └───────┬───────┘                              │
          │ seL4 IPC (VNET_OVER_IPC)             │
          ▼                                      │
  ┌───────────────┐                              │
  │  NIDS VM      │◄─────────────────────────────┘
  │               │   (镜像流量需经 NSv 代理)
  └───────────────┘
```

**限制分析**:

| 限制 | 说明 | 影响 |
|------|------|------|
| **DMA 只能到 CMA** | PFE DMA 目标物理地址由 NSv 分配，VM 无法直接访问 | NIDS 只能通过 NSv 代理获取镜像流量 |
| **RSS 分流到 VM** | PFE RSS 无法直接将包分发到 NIDS VM | NSv 仍是必经节点 |
| **HSE 密钥管理** | 加密操作在 HSE 硬件执行，结果通过 CMA 传递 | NIDS 只能看到明文（镜像后） |
| **VLAN 硬件解析** | PFE 解析 VLAN，但 NSv 软件分发到 netif | NIDS 需处理 VLAN tag |

**建议**:
- PFE 硬件 filters (e.g., Ethertype filter) 可在 NIC 驱动层减轻 NSv 负载
- NIDS 仅接收已解析的 IP 包，不处理 VLAN/ARP 等 L2 帧

---

## 3. seL4 安全优势

### 3.1 形式化验证的 TCB 最小化

seL4 是目前唯一经过形式化验证的微内核:
- **证明**: 内核代码正确性在 math proof 级别
- **TCB**: seL4 内核 (~9K LOC C) 是唯一的可信计算基
- **对比**: Linux TCB 包含数百万行内核代码

**NIDS 在 seL4 上的 TCB**:

| 方案 | TCB 组成 |
|------|----------|
| 方案 A (NIDS VM) | seL4 kernel + NIDS VM 代码 |
| 方案 B (嵌入 NSv) | seL4 kernel + NSv (lwIP + LWFW + NIDS) |

**形式化验证范围对比**:
- 方案 A: 验证 seL4 + NIDS，忽略 NSv 漏洞不影响 NIDS 安全属性
- 方案 B: 验证 seL4 + NSv + NIDS，整体安全属性依赖于所有组件

### 3.2 Capability 系统对 NIDS 的权限隔离

seL4 使用 capability 机制进行访问控制:

```c
// NIDS VM 所需 capability (最小权限原则)
nids_caps = {
    // 内存: 仅 NIDS 自身地址空间
    .nids_frame_caps = seL4_CPageTable_Get(va_space),

    // IPC: 仅与 NSv 通信的端点
    .nsv_ep_cap = seL4_CapEndpoint,  // badge = NSV_PID

    // 共享内存: NIDS 专用 CMA region
    .nids_cma_cap = seL4_CapDMASpace,

    // 通知: NIDS RX notification
    .nids_rx_ntfn = seL4_CapNotification,

    // 无权访问: 其他 VM 内存、硬件资源
};
```

**权限隔离效果**:
- NIDS VM **无法**直接访问 PFE 硬件寄存器
- NIDS VM **无法**访问其他 VM 的内存
- NIDS VM **无法**发送 seL4 syscall 给未知端点
- 即使 NIDS 被攻陷，攻击者也只能在 NIDS VM 沙箱内行动

### 3.3 HSE 硬件信任根的密钥管理

**HSE (Hardware Security Engine)**:
- NXP S32G 集成的硬件加密引擎
- 支持 AES-128/256、SHA、HMAC、RSA
- 用于安全启动、密钥存储、通信加密

**VM 架构下的密钥管理**:
```
┌─────────────────────────────────────────────────────────────────┐
│                      HSE Hardware                                │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │  HSE Key Slot │  │  HSE Crypto   │  │  HSE Master   │       │
│  │  (NVM OTP)   │  │  Engine       │  │  Key (rotated)│       │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘       │
│          │                  │                  │                 │
└──────────┼──────────────────┼──────────────────┼────────────────┘
           │                  │                  │
           │ seL4 IPC        │ DMA              │
           ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                        seL4 Microkernel                          │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 安全属性:                                                    │  │
│  │ - NSv: 有权调用 HSE进行加解密                                │  │
│  │ - NIDS: 无权直接访问HSE (无法获取密钥材料)                   │  │
│  │ - 通信加密: NSv 作为代理，NIDS 只能看到明文流量镜像          │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**NIDS 视角**:
- NIDS 接收到的流量已是解密后的明文 (IP 层)
- LWFW 连接追踪状态也存储在 NSv 内存中
- NIDS 只能通过 SOA 接口与外部通信，无直接 HSE 访问

---

## 4. 约束与挑战

### 4.1 VM 间共享内存的配置复杂度

**elem_ring 跨 VM 共享配置**:

```c
// NSv 侧: 创建共享 CMA region
cma = cma_create("nids-shared-cma", SIZE_16MB);
nids_ring = elem_ring_create(&cma, NIDS_RING_SIZE);

// 将 capability 传给 NIDS VM
seL4_CNode_Copy(
    dest: nids_vm.cspace,
    dest_depth: seL4_WordBits,
    src: nsv.cspace,
    src_index: cma_cap,
    src_depth: seL4_WordBits,
    rights: seL4_ReadWrite // NIDS 仅需读写
);

// NIDS VM 侧: 映射共享内存
nids_rx_ring_va = seL4_DMAPageMap(
    cap: nids_cma_cap,
    rights: seL4_AllRights
);
```

**配置复杂度来源**:
1. seL4 虚拟内存布局需提前规划
2. NIDS VM 的启动顺序必须在 NSv 之后
3. Capability 传递需在 VM 间建立 trust channel (通过 soa-framework)
4. 故障恢复后需重新建立共享内存映射

### 4.2 DMA 无法直接从 VM 访问硬件

**问题描述**:
- PFE DMA 目标地址由 NSv 分配 (CMA 区域)
- NIDS VM 无法直接给 PFE 编程，发送 DMA 目标地址
- 所有流量必须经过 NSv 代理

**架构影响**:
```
原始路径:
  PFE DMA → CMA → NSv → App

镜像路径 (方案 A):
  PFE DMA → CMA → NSv → elem_ring 复制 → NIDS VM
                      ↑
                  这个复制是必需的
```

**seL4 设备虚拟化选项**:
1. **NSv 代理** (当前方案): NSv 负责所有硬件交互，NIDS 获取镜像
2. **直通 (Passthrough)**: NIDS VM 获得 PFE 直接访问权限 → 牺牲隔离性
3. **SMMU 虚拟化**: 每个 VM 有独立 SMMU 上下文 → 配置复杂

### 4.3 内存预算: 512MB 下的分区

**512MB 内存预算分析**:

| 组件 | 静态占用 | 动态峰值 | 说明 |
|------|----------|----------|------|
| seL4 kernel | ~2MB | ~2MB | 固定 |
| NSv (lwIP+LWFW) | ~40MB | ~80MB | 含 CMA 96MB (可配置) |
| NIDS VM | ~20MB | ~60MB | 含 PacketPool + DetectionEngine |
| 其他 VM (Android/QNX) | ~200MB | ~300MB | 车型配置 |
| RTOS VMs (mrtos/vm-a53) | ~50MB | ~50MB | 固定 |
| **总计** | ~312MB | ~492MB | 接近上限 |

**NIDS 内存预算细分**:

```yaml
nids_memory_budget:
  text_area: 2MB           # 代码 + 只读数据
  data_area: 4MB           # 全局变量 + 堆
  stack: 512KB × 2         # CaptureThread + WorkerThread
  packet_pool:
    small_slots: 2048 × 256B   = 512KB
    std_slots:    2048 × 2KB   = 4MB
    large_slots:   512 × 16KB  = 8MB
  detection_engine:
    port_group_index:  64KB
    content_matcher:   1MB    # AC automaton
    detection_filter:  160KB
    port_scan_tables:  512KB
  event_queue: 1MB
  spsc_queue:   2MB
  # 总计: ~23MB 静态 + ~20MB 峰值 = ~43MB
```

**建议**:
- CMA 区域在 NSv 和 NIDS 间按需分配 (NSv 60%, NIDS 40%)
- NIDS PacketPool 大小可根据实际流量调整
- 预留 20% 内存余量应对峰值

---

## 5. 部署建议

### 5.1 推荐架构

**推荐: 方案 A 增强版 — NIDS 独立 VM + 优化共享内存路径**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           seL4 Microkernel                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        NIDS VM (安全 VM)                            │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │ CaptureThread          │ WorkerThread                          │ │   │
│  │  │ ┌─────────┐           │ ┌─────────────┐  ┌───────────────┐ │ │   │
│  │  │ │PcapSource│           │ │ProtocolDec │→│DetectionEngine│ │ │   │
│  │  │ │ elem_ring│←──────────┼─│(DecodeResult)│→│(RuleMatcher) │ │ │   │
│  │  │ │  get    │           │ │             │  │PortScanInsp. │ │ │   │
│  │  │ └────┬────┘           │ └─────────────┘  └───────┬───────┘ │ │   │
│  │  │      │ elem_ring      │                           │         │ │   │
│  │  │      │ put (to pool)  │                           │         │ │   │
│  │  │      ▼                │                           ▼         │ │   │
│  │  │ ┌─────────────┐       │                    ┌───────────┐    │ │   │
│  │  │ │ PacketPool │       │                    │EventEngine│    │ │   │
│  │  │ └─────────────┘       │                    └─────┬─────┘    │ │   │
│  │  └────────────────────────────────────────────────────┼─────────┘ │   │
│  └────────────────────────────────────────────────────────┼──────────┘   │
│                                                             │              │
│  ┌─────────────────────────────────────────────────────────┼──────────┐   │
│  │                        NSv VM                           │          │   │
│  │  ┌──────────────────────────────────────────────────────┼────────┐ │   │
│  │  │ nic_rx_thread ──────────────────────────────────────────────→│ │   │
│  │  │      │                    LWFW                        │      │ │   │
│  │  │      ▼              ┌──────────┐                     │      │ │   │
│  │  │  elem_ring          │Ingress   │                     │      │ │   │
│  │  │  (used_rx)         │Filter    │─────────────────────┘      │ │   │
│  │  │      │              └──────────┘                            │ │   │
│  │  │      │                                                    │ │   │
│  │  │      ├─────────────────────────────────────────────────────│─│───┘   │
│  │  │      │ elem_ring (nids_mirror_ring)                       │ │       │
│  │  │      │ seL4_signal(nids_rx_ntfn)                          │ │       │
│  │  │      │                                                    │ │       │
│  │  └──────┼────────────────────────────────────────────────────┼─────────┘   │
│  └─────────┼────────────────────────────────────────────────────┼────────────┘
│            │                                                    │
│            │ CMA (共享内存)                                       │
│            ▼                                                    │
│  ┌─────────────────┐                                            │
│  │   NIC Driver    │◄───────────────────────────────────────────┘
│  │     (PFE)      │         elem_ring (nids_mirror_ring)
│  └─────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

**核心设计决策**:

1. **独立 VM 部署**: 安全隔离优先，IPC 开销可接受
2. **专用镜像 ring**: `nids_mirror_ring` 与 NSv 自用 ring 分离，避免竞争
3. **异步镜像**: NSv 在 `rx_callback` 完成后异步复制到 NIDS ring，不阻塞主路径
4. **NIDS 零拷贝**: NIDS 直接从 `nids_mirror_ring` 读取 PA，通过 CMA VA 访问

### 5.2 与 LWFW 的联动接口设计

**检测 → 阻断 联动架构**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           NIDS VM                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  DetectionEngine.Inspect()                                          │ │
│  │       │                                                              │ │
│  │       │ 检测结果                                                      │ │
│  │       ▼                                                              │ │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │ │
│  │  │AlertPolicy  │───→│  LWFW Agent │───→│ SOA Report  │             │ │
│  │  │(cooldown)   │    │(阻断指令)    │    │(JSON event) │             │ │
│  │  └─────────────┘    └──────┬──────┘    └─────────────┘             │ │
│  └────────────────────────────┼────────────────────────────────────────┘ │
└───────────────────────────────┼───────────────────────────────────────────┘
                                │ seL4 IPC
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                             NSv VM                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ LWFW 阻断指令处理                                                      │ │
│  │      │                                                                 │ │
│  │      │ lwfw_agent_evt_consume()                                        │ │
│  │      │                                                                 │ │
│  │      ▼                                                                 │ │
│  │ ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │ │ 1. 解析事件: event_type = SID / 1_000_000                         │  │ │
│  │ │    2_xxx_xxx → NetworkScan → 添加到 LWFW block list               │  │ │
│  │ │    1_xxx_xxx → AttemptedDos → LWFW rate limit                     │  │ │
│  │ │    3_xxx_xxx → AttemptedRecon → LWFW DROP rule                    │  │ │
│  │ │                                                                    │  │ │
│  │ │ 2. 调用 lwfw_policy_insert_rule()                                 │  │ │
│  │ │    action = DENY, match = {src_ip/dst_ip/port}                    │  │ │
│  │ │                                                                    │  │ │
│  │ │ 3. 原子切换: inactive_policy ↔ policy                            │  │ │
│  │ └──────────────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

**联动接口定义**:

```yaml
nids_lwfw_interface:
  # NIDS → LWFW 阻断指令 (通过 SOA)
  block_rule:
    protocol: SOA JSON over seL4 IPC
    fields:
      - action: "block" | "rate_limit" | "log_only"
      - src_ip: string (CIDR 或具体 IP)
      - dst_ip: string
      - src_port: uint16 | "any"
      - dst_port: uint16 | "any"
      - proto: "tcp" | "udp" | "icmp" | "any"
      - duration_sec: uint32 (0 = 永久)
      - reason: string (SID/gid/msg)

  # LWFW → NIDS 反馈 (通过 SOA)
  block_feedback:
    - blocked_count: uint64
    - last_block_time: timestamp
    - rule_id: uint32
```

**响应时间目标**:
- NIDS 检测到攻击 → LWFW 生效: < 10ms
- LWFW 反馈阻断统计 → NIDS: < 1s (轮询)

### 5.3 启动顺序与故障恢复

**启动顺序 (Stage-based)**:

```json
{
  "stage": [
    {
      "name": "BOOT",
      "services": [
        "seL4_kernel",
        "nic_driver",
        "vm-a53_rtos"
      ]
    },
    {
      "name": "MOUNT",
      "services": [
        "NSv",
        "NSv:lwIP_ready",
        "NSv:LWFW_ready"
      ]
    },
    {
      "name": "RUN",
      "services": [
        "NIDS",
        "NIDS:Capture_ready",
        "NIDS:Worker_ready",
        "NIDS:RuleSet_loaded",
        "NIDS:LWFW_link_established"
      ]
    }
  ]
}
```

**故障恢复流程**:

| 故障场景 | 检测方式 | 恢复动作 |
|----------|----------|----------|
| NIDS VM 崩溃 | NSv: elem_ring 无消费，超时检测 | NSv 继续收包，SOA 上报事件 |
| NIDS CaptureThread 卡死 | HealthMonitor: CPU 占用 0% + 无包处理 | SOA 重启 NIDS VM |
| NIDS WorkerThread 过载 | HealthMonitor: CPU ≥ 30% 持续 5s | 自动降级: 减少检测规则、增大 PacketPool |
| NSv → NIDS ring 断开 | NIDS: elem_ring_get 返回 0 超时 | 重新建立共享内存映射 |
| NIDS LWFW 联动超时 | LWFW Agent: SOA 请求无响应 | 降级为仅告警，不阻断 |

**SOA 健康检查协议**:

```yaml
nids_health_protocol:
  heartbeat_interval_ms: 1000
  timeout_ms: 5000
  checks:
    - NIDS: CaptureThread 运行 + PacketPool 有可用 slot
    - NIDS: WorkerThread 运行 + SPSC queue 未溢出
    - LWFW: 规则数 > 0 + last_rule_update < 60s
    - Link: NIDS ↔ LWFW SOA channel 正常
```

---

## 6. 总结

| 决策点 | 推荐方案 | 理由 |
|--------|----------|------|
| **VM 部署** | 方案 A (独立 NIDS VM) | 隔离性优先，IPC 开销可接受 |
| **数据路径** | 专用 nids_mirror_ring + 异步复制 | 零侵入 NSv 主路径，避免竞争 |
| **LWFW 联动** | SOA JSON over seL4 IPC | 解耦设计，NSv 无需感知 NIDS 内部结构 |
| **启动依赖** | NSv 先于 NIDS，LWFW ready 后启动 NIDS | 确保镜像 ring 已建立 |
| **内存预算** | NIDS 静态 ~23MB，峰值 ~43MB | 预留 512MB 的 ~8%，不影响其他 VM |

**风险与缓解**:

| 风险 | 缓解措施 |
|------|----------|
| IPC 延迟影响实时检测 | 实测 < 2μs，低于 P99 100ms 目标的 2% |
| VM 间共享内存配置复杂 | 通过 soa-framework 封装，VDF 构建时统一配置 |
| NIDS VM 故障影响监控覆盖 | SOA 健康检查 + 自动重启，监控不中断 |
| PFE DMA 无法直通 NIDS | 接受 NSv 代理架构，镜像流量足够满足 NIDS 需求 |
