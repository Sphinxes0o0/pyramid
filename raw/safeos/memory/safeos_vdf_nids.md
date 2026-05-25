---
name: safeos_vdf_nids_relation
description: SafeOS与VDF nids项目的架构关系和适配分析
type: reference
---

# SafeOS 与 VDF nids 项目关系

## 仓库关系

| 仓库 | 路径 | 职责 |
|------|------|------|
| SafeOS | `/home/shiyang/nio/nt35/safeos/` | seL4 微内核 + os-framework (NSv网络栈) |
| VDF | `/home/shiyang/nio/nt35/vdf/` | 应用层 recipes，包含 nids IDS |

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

## 关键架构差异

### 内存模型
- **SafeOS**: DSPACE 共享内存对象 + seL4 IPC 授权，不是标准 mmap
- **Linux**: mmap() socket fd → 内核 ring buffer

### API 层级
- **SafeOS NSv**: `packet_mmap_setup()` → `sys_dspace_create()` + `grant_shm_to_net()`
- **Linux**: `setsockopt(PACKET_VERSION)` → `setsockopt(PACKET_RX_RING)` → `mmap()`

## nids 适配 SafeOS 需要的条件

1. CMakeLists.txt 添加 SafeOS os-framework 头文件路径
2. 链接 `core` 库 (含 packet_mmap.c)
3. 实现 SafeOS 专用 capture backend

## 相关文档

- `docs/NSv_analysis.md` — NSv 深度分析
- `docs/af_packet_mmap_summary.md` — AF-PACKET 实现总结与优化建议
