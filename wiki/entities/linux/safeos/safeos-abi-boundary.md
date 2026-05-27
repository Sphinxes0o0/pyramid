---
type: entity
tags: [linux, safeos, abi, architecture, header-exposure, api-design, nsv, packet-mmap]
created: 2026-05-26
sources: [safeos-architecture]
---

# SafeOS ABI Boundary — 内部头文件暴露与 API 分层设计

## 定义

SafeOS os-framework 当前存在 **内部头文件暴露问题**：`servers/net/include/nsv/packet_mmap.h` 作为 NSv 内部实现头文件，被 App (tcpdump/net-cap) 通过 CMakeLists.txt 显式依赖，导致 App 直接看到 NSv 内部结构体、指针类型和实现细节，违反分层架构原则。

---

## 当前暴露范围

```
os-framework/
├── servers/net/include/     ← NSv内部头文件
│   ├── nsv/packet_mmap.h    ← App需要访问 (但属于内部)
│   └── nsv/nsv.h
├── libs/os_libs/libcore/    ← 公共库
│   └── include/core/        ← 理论上应暴露
└── apps/tcpdump/
    └── CMakeLists.txt 中添加了:
        ${CMAKE_SOURCE_DIR}/servers/net/include  ← 显式依赖NSv内部
```

### 问题核心：`struct packet_mmap_info`

```c
// NSv 内部定义，但 tcpdump/net-cap 直接依赖
struct packet_mmap_info {
    pid_t               pid;
    int                 socket;
    unsigned int        tp_frame_size;
    uint16_t            rx_buf_offset;
    volatile uint16_t   loop_recevent;
    volatile uint16_t   recved_packets;
    void               *netconn;           // ← NSv 内部指针类型!
};
```

App 需要知道 `rx_buf_offset` 来计算 ringbuf 位置，需要知道 `recved_packets` 来判断有多少包可读。

---

## 暴露判断表

| 头文件类型 | 位置 | 是否应暴露 | 理由 |
|-----------|------|-----------|------|
| `nsv/packet_mmap.h` | servers/net | ❌ 不应 | NSv内部结构，App只应知道函数签名 |
| `nsv/nsv.h` | servers/net | ❌ 不应 | NSv内部常量/宏 |
| `core/syscalls.h` | libs | ⚠️ 部分 | 通用syscall封装，App需要 |
| `netpacket/packet.h` | musllibc | ✅ 应暴露 | 标准POSIX/Linux定义 |

---

## 当前 vs 理想架构对比

### 正确分层 (应该)

```
┌─────────────────────────────────────────┐
│  App (tcpdump/netcap)                   │
│  只通过稳定ABI/API调用                   │
└────────────────┬────────────────────────┘
                 │ 稳定接口
┌────────────────▼────────────────────────┐
│  NSv 服务                              │
│  提供公开头文件                         │
│  (packet_mmap_abi.h)                    │
└─────────────────────────────────────────┘
```

### 当前实际 (有问题)

```
┌─────────────────────────────────────────┐
│  App (tcpdump/netcap)                  │
│  直接看到 NSv 内部源码                  │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  servers/net/src/                       │  ← App不应该看到这个
│  servers/net/include/                   │  ← App不应该依赖这些
└─────────────────────────────────────────┘
```

---

## 根本问题

`nsv/packet_mmap.h` **既是 NSv 内部实现，又是 App 需要的 API**，没有分离：

```c
// 当前: 一个文件同时包含:
// 1. 实现细节 (packet_mmap_info 内部布局)
// 2. API契约 (packet_mmap_setup 函数签名)

// 应该分离为:
// 1. public API: packet_mmap.h (只暴露函数签名+常量)
// 2. private impl: nsv/packet_mmap_impl.h (暴露结构体+常量)
```

---

## 理想改造方案

创建稳定 ABI 接口层：

```
os-framework/
├── include/                      ← 新增: 公共头文件目录
│   └── nsv/
│       └── packet_mmap.h       ← 稳定ABI，只暴露App需要的类型
├── servers/net/
│   ├── include/                 ← 现有内部头文件
│   │   └── nsv/packet_mmap.h   ← 移动到这里作为private
│   └── src/
└── apps/tcpdump/
    └── CMakeLists.txt 改为:
        target_include_directories(tcpdump PRIVATE
            ${CMAKE_SOURCE_DIR}/include  ← 改为公共ABI
        )
```

### 公共头文件 `include/nsv/packet_mmap.h` 应只包含

```c
#ifndef _NSV_PACKET_MMAP_ABI_H_
#define _NSV_PACKET_MMAP_ABI_H_

#include <stdint.h>

// App侧需要的常量
#define DEFAULT_TP_FRAME_SIZE   2048
#define DEFAULT_TP_FRAME_NR     1024
#define DEFAULT_TP_BLOCK_SIZE   4096
#define DEFAULT_TP_BLOCK_NR     512
#define DEFAULT_DSPACE_SIZE     0x400000ul

// App侧需要的只读字段 (通过偏移量访问)
struct packet_mmap_info {
    uint16_t rx_buf_offset;      // ringbuf 偏移
    volatile uint16_t loop_recevent;
    volatile uint16_t recved_packets;
    // 其他字段 App 不应直接访问
};

// 函数签名
int packet_mmap_setup(int socket, void *dsinfo, void **info_out);
int packet_mmap_destroy(void *dsinfo);
int process_packet(void *info, void *buf, int bufsize);

#endif
```

---

## repo 间暴露分析

```
.repo/
├── manifest.xml          ← repo 管理多仓库
├── os-framework/
│   └── servers/net/include/  ← 暴露给了谁?
├── external/
│   └── net-cap/          ← 通过CMakeLists.txt显式引入
├── kernel/               ← 不受影响，seL4内核无关系
└── tools/                ← 构建工具
```

- **os-framework 内部 app (tcpdump 等)**: 会暴露，CMakeLists.txt 显式包含 `servers/net/include`
- **external/ 下的独立项目 (net-cap)**: 通过自己的 CMakeLists.txt 显式引入
- **其他 repo (kernel/tools)**: 不受影响，不链接 os-framework

---

## 问题相关文件

| 文件 | 问题 |
|------|------|
| `servers/net/include/nsv/packet_mmap.h` | 结构体暴露给App |
| `servers/net/src/packet_mmap.c` | NSv内部实现 |
| `os-framework/apps/tcpdump/CMakeLists.txt` | 显式依赖NSv内部 |
| `external/net-cap/CMakeLists.txt` | 显式依赖NSv内部 |
| `os-framework/libs/os_libs/libcore/include/core/` | 公共库边界模糊 |

### 需要新建的改造文件

| 文件 | 作用 |
|------|------|
| `include/nsv/packet_mmap.h` | 稳定ABI头文件 |
| `libs/os_libs/libpacket_mmap/` | packet_mmap公共库 (待创建) |

---

## 改造路线图

1. **短期**: 文档化当前架构约束，新App开发时遵循现有模式
2. **中期**: 将 `packet_mmap.c` 抽取为 `libpacket_mmap` 公共库
3. **长期**: 分离 public ABI 和 private impl，创建 `include/nsv/` 稳定接口层

---

## 相关概念

- [[entities/linux/safeos/safeos-packet-mmap]] — AF-PACKET + TPACKET 抓包实现，受此 ABI 问题直接影响
- [[entities/linux/safeos/safeos-nsv]] — NSv Network Server，内部头文件的来源
- [[entities/linux/safeos/safeos-network-implementation]] — SafeOS 网络实现完整分析，CMA+DS-RING 架构
- [[entities/linux/safeos/safeos-vdf-nids-relation]] — VDF nids 适配受此问题影响
- [[entities/linux/safeos/safeos-lwip-lwfw-plan]] — SafeOS lwIP+LWFW 深度分析计划

## 来源详情

- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents (architecture_notes.md)
