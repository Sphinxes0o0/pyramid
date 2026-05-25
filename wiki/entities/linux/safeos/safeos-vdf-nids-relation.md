---
type: entity
tags: [safeos, vdf, nids, af-packet, packet-mmap, dspace, sel4, architecture]
created: 2026-05-25
sources: [safeos-architecture]
---

# SafeOS 与 VDF nids 项目关系

## 定义

SafeOS 是运行 seL4 微内核 + NSv 网络栈的嵌入式操作系统，VDF (Vehicle Distributed Framework) 是应用层 recipes，包含 nids (Network-based Intrusion Detection System) 入侵检测应用。当前 nids 需要适配 SafeOS 才能使用其 AF-PACKET 抓包后端。

---

## 仓库关系

| 仓库 | 路径 | 职责 |
|------|------|------|
| SafeOS | `/nio/nt35/safeos/` | seL4 微内核 + os-framework (NSv网络栈) |
| VDF | `/nio/nt35/vdf/` | 应用层 recipes，包含 nids IDS |

---

## SafeOS os-framework 结构

- `os-framework/servers/net/` — NSv 网络服务器 (lwIP 用户态栈)
- `os-framework/servers/net/include/nsv/` — NSv 头文件 (packet_mmap.h 等)
- `os-framework/apps/tcpdump/` — tcpdump 应用
- `external/net-cap/` — net-cap 抓包应用
- `libs/os_libs/libcore/` — 核心库 (dspace, ringbuffer, ds_ring)

## VDF nids 结构

- `vdf/applications/recipes/nids/` — NIDS 入侵检测应用
- 当前分支: `feat/capture-afpacket-mmap`
- 构建系统: 独立 CMake，不在 SafeOS os-framework 内

---

## 关键架构差异

### 内存模型

| 方面 | SafeOS | Linux |
|------|--------|-------|
| **内存共享** | DSPACE 共享内存对象 + seL4 IPC 授权，不是标准 mmap | mmap() socket fd → 内核 ring buffer |
| **API 层级** | `packet_mmap_setup()` → `sys_dspace_create()` + `grant_shm_to_net()` | `setsockopt(PACKET_VERSION)` → `setsockopt(PACKET_RX_RING)` → `mmap()` |

### API 层级对比

- **SafeOS NSv**: `packet_mmap_setup()` → `sys_dspace_create()` + `grant_shm_to_net()`
- **Linux**: `setsockopt(PACKET_VERSION)` → `setsockopt(PACKET_RX_RING)` → `mmap()`

---

## nids 适配 SafeOS 可行性

### 功能层面

| 模块 | 功能 | SafeOS 兼容性 |
|------|------|--------------|
| **Decoder** (ipv4.cc, tcp.cc, udp.cc 等) | 纯 packet 数据解析 | ✅ 兼容 |
| **Preprocess** (packet_meta.cc) | packet 元数据提取 | ✅ 兼容 |
| **Flow hash** (flow_hash.cc) | 流哈希计算 | ✅ 兼容 |
| **Stats collector** | 统计收集 | ✅ 兼容 |
| **af_packet_mmap capture** | 从 ring buffer 读包 | ❌ 需重写 SafeOS 后端 |

### 适配障碍

1. **nids 当前期望标准 Linux AF-PACKET API**，而 SafeOS 使用自定义 DSPACE 方案
2. **nids CMakeLists.txt 没有 SafeOS os-framework 头文件路径**
3. **nids 无法链接 `core` 库**（包含 packet_mmap.c 实现）

### nids 适配 SafeOS 需要的条件

1. CMakeLists.txt 添加 SafeOS os-framework 头文件路径
2. 链接 `core` 库 (含 packet_mmap.c)
3. 实现 SafeOS 专用 capture backend

**结论**: nids decoder 部分功能完全兼容，但 capture 层需要为 SafeOS 实现专用 backend，使用 `packet_mmap_setup()` + `rb_read()` API。

---

## 当前架构问题

### 内部头文件暴露

```
os-framework/
├── servers/net/include/     ← NSv内部头文件
│   ├── nsv/packet_mmap.h    ← App需要访问 (但属于内部)
│   └── nsv/nsv.h
├── libs/os_libs/libcore/    ← 公共库
│   └── include/core/         ← 理论上应暴露
└── apps/tcpdump/
    └── CMakeLists.txt 中添加了:
        ${CMAKE_SOURCE_DIR}/servers/net/include  ← 显式依赖NSv内部
```

**问题**: `nsv/packet_mmap.h` **既是 NSv 内部实现，又是 App 需要的 API**，没有分离。

### 正确分层 (应该)

```
┌─────────────────────────────────────────┐
│  App (nids / tcpdump / netcap)         │
│  只通过稳定ABI/API调用                   │
└────────────────┬────────────────────────┘
                 │ 稳定接口
┌────────────────▼────────────────────────┐
│  libpacket_mmap (公共库)                │
│  - packet_mmap_setup()                 │
│  - process_packet()                    │
│  - 稳定的数据结构 (packet_mmap_info)   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  NSv 服务                              │
│  - packet_mmap_set_ring() (private)    │
│  - tpacket_recv callback (private)     │
│  - DSPACE 管理 (private)               │
└─────────────────────────────────────────┘
```

---

## 优化建议

### 短期 (文档化)
- 文档化当前架构约束，新 App 开发时遵循现有模式

### 中期 (代码复用)
- 将 `packet_mmap.c` 抽取为 `libpacket_mmap` 公共库

### 长期 (API 稳定性)
- 分离 public ABI 和 private impl，创建 `include/nsv/` 稳定接口层
- 优先创建稳定 ABI 层 (`include/nsv/`)，抽取 `libpacket_mmap` 共享库

---

## 相关概念

- [[entities/linux/safeos/safeos-packet-mmap]] — AF-PACKET + TPACKET 抓包实现详解
- [[entities/linux/safeos/safeos-nsv]] — NSv Network Server 完整分析
- [[entities/linux/safeos/safeos-network-implementation]] — SafeOS 网络实现完整分析
- [[entities/linux/lwip/lwip-packet-mmap]] — lwIP packet_mmap 实现
- [[entities/linux/lwip/lwip-raw-socket]] — RAW socket / AF-PACKET 绑定

## 来源详情

- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents
