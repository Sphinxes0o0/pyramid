---
type: synthesis
tags: [safeos, vdf, sel4, virtualization, build-system]
created: 2026-05-25
sources: [safeos-architecture]
---

# SafeOS VDF (Virtual Device Framework) on seL4 — Architecture Analysis

## 1. 整体架构

### 1.1 VDF 在 SafeOS 中的定位

VDF (Virtual Device Framework) 是 SafeOS seL4 虚拟化架构中的**镜像构建与配置编排系统**，而非传统意义上的运行时虚拟化框架。它负责将来自多个远程 conan recipe 的二进制包（safeos-base、hypervisor、vm-a53、mrtos 等）组装成可启动的 disk image。

```
SafeOS seL4 层次结构（从上到下）:
┌─────────────────────────────────────┐
│  User VMs (Android/QNX/Linux/...)   │  ← 运行在 soa-framework 管理的 VM 中
├─────────────────────────────────────┤
│  soa-framework (Service Framework)  │  ← 服务编排层,管理 VM 间通信
├─────────────────────────────────────┤
│  safeos-base (seL4 微内核 + libc)   │  ← seL4 微内核
├─────────────────────────────────────┤
│  hypervisor (NPT 虚拟化管理)         │  ← NPT = NIO Protected Table?
├─────────────────────────────────────┤
│  mrtos / vm-a53 (RTOS 虚拟机)        │  ← ARMv8 Cortex-A 实时虚拟机
├─────────────────────────────────────┤
│  Boot: uboot + ATF + OP-TEE         │
└─────────────────────────────────────┘

VDF 在上述架构中的角色:
  VDF 不在运行时栈中 — 它是一个 conan-based build system,
  负责配置 + 编排 + 生成最终 disk image (GPT 分区镜像).
```

### 1.2 与 NSv / lwIP 的关系

- **VDF 不直接使用 NSv 或 lwIP**。VDF 是构建系统，网络配置以 YAML/JSON 配置文件形式生成，最终被部署到镜像的 `/etc/network/` 目录。
- 网络功能由运行在 VM 中的服务（soa-framework 管理的应用）使用，VDF 负责生成这些配置文件（如 `host_vlan.yaml`、`topology.yaml`、`switch.yaml`）。
- XCP (XCP on Ethernet) 配置用于**标定/测量数据上传**，走专用以太网通道。

### 1.3 模块划分

```
vdf-sel4/
├── modules/            # Python conan 扩展模块 (DiskImage, StageManager, NetConf, Xcp...)
│   ├── diskimage.py    # GPT/RAW 磁盘镜像生成
│   ├── stage/          # 服务启动阶段管理 (stage groups, vehicle-specific services)
│   │   ├── stage_manager.py   # 解析 deployed services, 生成 domain.json
│   │   └── dac.py      # DAC (Discretionary Access Control) passwd/group 生成
│   ├── netconf/        # 网络配置生成 (DNS resolv.conf, host_vlan.yaml)
│   ├── xcp.py          # XCP (标定协议) 后构建生成 A2L/hex/json
│   ├── secpol.py       # 安全策略打包
│   ├── variant_service/# 车型变体配置 (VC_vehicleProject 等)
│   └── signature.py    # 镜像签名
├── vft-config/         # 虚拟功能配置 (per-vehicle CMake)
│   ├── entryplus/       # LEO/Cetus 平台 (QINLING 配置)
│   └── vehicleid/      # 车型特定网络/服务配置 (0x3001 等)
├── xcp_user/           # XCP 代码生成工具链 (A2L parser, ELF parser, generator)
│   └── tools/code_gen/ # Python XCP post-build 工具
├── rootfs/             # 静态 rootfs 文件 (safeos/linux base)
├── config/             # CodingMapSimple.json, sysuser.json, stage config
├── test/               # 构建验证测试
└── conanfile.py       # 主 conan recipe (VDFSafeOS)
```

### 1.4 seL4 虚拟化层次

```
平台支持:
  qemu-arm-virt    — 软件仿真
  vdc1a / vdcb1   — NIO 内部开发板
  s32g3-vdc1-prem — Apollo/Cetus (Premium)
  s32g3-vdc1-lite — DOM/Blanc (Lite)
  m57-evb / m57-vdf — M57 demo 板

seL4 配置 target_os: sel4 或 linux
```

---

## 2. 核心模块详解

### 2.1 modules/diskimage.py — 磁盘镜像生成

**职责**: 使用 `conannio.diskgen` 将多个 recipe 的 deploy 文件组装为 GPT 分区镜像。

**关键流程**:
```
disk_image() → pre_rootfs() → diskimage_export() → post_rootfs() → do_disk_image()
```

**关键类 `DiskImage`**:
- `new_disk()`: 根据 YAML 配置（GPT/RAW/MBR）创建 `diskgen.DiskImage`
- `new_partition()`: 合并 recipe 中的 partition 继承配置
- `collect_deploy_all()`: 收集所有 recipe 的 deploy 文件（包括动态链接库路径）
- `do_disk_image()`: 导出镜像、生成 `output.yml`、打包 tar.gz

**磁盘布局** (s32g3):
```
emmc (GPT):
  ├── uboot_a / uboot_b     (RAW, 5M)
  ├── hsm_a / hsm_b         (RAW, HSP region)
  ├── spl_a / spl_b
  ├── rtos_a / rtos_b
  ├── atf_a / atf_b
  ├── optee_a / optee_b
  ├── boot_a / boot_b        (FAT, kernel + dtb)
  └── rootfs_a / rootfs_b    (EXT4, 主文件系统)
```

### 2.2 modules/stage/stage_manager.py — 服务编排

**职责**: 解析 conan recipe deploy 的 services，根据车型生成 `/etc/stage/domain.json`。

**关键概念**:
- **Stage**: 服务分组（BOOT, MOUNT, RUN, FOTA 等）
- **Service**: 每个 service 有 `stage`、`credential`（运行用户/组/umask）等属性
- **Vehicle-specific services**: 某些服务仅在特定车型/平台启用

**关键流程**:
```
gen_configs(disk)
  ├── get_deployed_services()      # 从 recipe deploy 解析
  ├── get_predefined_services()    # 静态禁用列表
  ├── get_vehicle_names()          # 读 CodingMapSimple.json
  ├── get_vehicle_specific_service_names()  # 按 platform (premium/lite) 过滤
  ├── gen_domain_config()          # 生成 domain.json
  └── gen_sys_dac()                # 通过 dac.py 生成 /etc/passwd, /etc/group
```

**输出**: `/etc/stage/domain.json`
```json
{
  "service": [
    {"name": "xxx", "stage": "BOOT", "credential": {"user": "root", "umask": "022"}}
  ],
  "stage": [
    {"name": "BOOT", "groups": ["service1", "service2"]}
  ]
}
```

### 2.3 modules/netconf/ — 网络配置生成

**host_vlan.yaml 配置的 VLAN 层次**:
```
PFE (NIO 1G 以太网 PHY)
├── PFE        (native, 172.20.20.0/24, mgmt)
├── PFE.VLAN1  (172.20.1.0/24, Diag External)
├── PFE.VLAN2  (172.20.2.0/24, PEU Report)
├── PFE.VLAN5  (172.20.5.0/24, Signal-based PDU)
├── PFE.VLAN10 (172.20.10.0/24, 默认骨干通信, IP forwarding)
├── PFE.VLAN11 (172.20.11.0/24, SoA General)
├── PFE.VLAN40 (172.20.40.0/24, NIO-Paid Services)
└── PFE.VLAN41 (172.20.41.0/24, Power-Swap Service)
```

**DNS 策略**: 支持 country-specific fallback，调用 `generate_conf.py` 生成 `resolv.conf`。

**switch.yaml**: RTL9071CP 物理交换机配置（端口镜像、健康检查、心跳等）

### 2.4 modules/xcp.py + xcp_user/ — XCP 标定协议

**XCP** (Universal Calibration Protocol) 用于 ECU 在线标定和数据采集。

**工具链** (post-build):
```
generator.sh
  ├── repos_resolver.py    # 解析 xcp_config/repos.json
  ├── gen_xcp.py           # 主生成器入口
  │   ├── elf_hex_parser.py    # 从 ELF 提取 .xcp_cal_seg.* sections
  │   ├── a2l_parser/          # ASAM MCD-2MC A2L 文件解析
  │   ├── a2l_process.py        # A2L 处理 + 地址绑定
  │   └── dcm_merge_hex.py      # 合并用户 DCM hex
  └── 输出:
        ├── xcp_a2l_gen_log.txt
        ├── <VEHICLE>/xcp_config.json
        └── <VEHICLE>/*.hex, *.a2l
```

**xcp_config.json 结构**:
- `xcp_stack_static_config`: XCP 协议栈静态配置（DAQ、STIM、TIMESTAMP）
- `xcp_process_cluster_config`: 每个测量进程集群的配置（ELF 文件、memory segments、A2L 路径）

### 2.5 modules/variant_service/ — 车型变体管理

**职责**: 管理不同车型（LEO、Cetus、DOM、Blanc）的服务配置变体。

**配置链**:
```
collect_deploy_variants() → gen_variant_service_config()
  → /etc/variant-service/variant-service_conf.json
     └── config_link: { module_name: [ { symlink_path, target_path_prefix, target_path_suffix } ] }
```

### 2.6 vft-config/ — 虚拟功能配置 (CMake 层)

```
vft-config/
├── CMakeLists.txt          # find_package(soa-framework), find_package(safeos-base)
├── entryplus/              # LEO/Cetus premium 平台
│   ├── CMakeLists.txt      # foreach(VEH) install vehicle config
│   └── QINLING/etc/        # QINLING 平台特定配置
│       ├── comm-control/   # 通信控制配置 (TSP endpoint)
│       ├── dcl/            # Data Collection Log 配置 (syslog, soa_adapter, camera, gnss...)
│       ├── soa-recorder/   # SoA 数据记录器配置
│       └── sysm/           # System Manager 配置
└── vehicleid/              # 车型 ID 特定配置
    ├── CMakeLists.txt      # foreach(0x2001/0x2002/...) install config
    └── 0x3001/etc/network/ # 交换机/拓扑配置
```

---

## 3. 数据路径

### 3.1 VM 间通信机制

VM 间通信通过 **soa-framework** (Service-Oriented Architecture framework) 管理，配置在 `vft-config/entryplus/QINLING/etc/` 下:

- **comm-control**: TSP 云端通信控制（HTTP endpoint、app_id、secret）
- **soa-recorder**: SoA 数据记录（`/var/log/soarec`），支持本地缓冲 + 远程 UDP 上传
- **dcl**: Data Collection Log（syslog、camera、gnss、imu、raw-data...）
- **sysm**: 系统管理数据配置（0x41F1 等 CAN 消息路由）

**通信拓扑**（从 topology.yaml）:
```
物理网络:
  PFE (1G PHY) ←→ VDF_SW (RTL9071CP) ←→ 各 ECU (ZONE_*, CDF_*, ADF_*)

VM 内部:
  VDF_VM_M ←→ VSW_VDF (VM_SWITCH) ←→ VDF_VM_L
  VDF_SW   ←→ VSW_CDF (VM_SWITCH) ←→ CDF_CLUSTER, CDF_HOST, SAF
```

### 3.2 与 NSv 网络栈的集成

VDF **不直接集成** NSv 或 lwIP。VDF 生成的配置文件（host_vlan.yaml、topology.yaml）描述了**静态网络拓扑**，由运行在 seL4 VM 中的网络服务读取和使用。

**VLAN + QoS 映射** (host_vlan.yaml):
```
Queue 0-4: SP (Strict Priority)
Queue 5:   CBS Class B, 250Mbps (AVB 视频)
Queue 6:   CBS Class A, 500Mbps (ADAS 数据)
Queue 7:   SP, non-preemptable (安全关键帧)
```

### 3.3 性能特征

| 维度 | 描述 |
|------|------|
| **构建时延** | 镜像打包 + XCP 后构建总时间由 recipe 数量决定 |
| **XCP DAQ** | 通过 Ethernet (XCP on UDP) 上传标定数据，UDP port 15000 |
| **网络吞吐** | PFE 1G PHY，VLAN 隔离，QoS 8 队列 |
| **SoA 录制** | 本地 5×1MB buffer + 远程 UDP (port 15000)，dir 上限 200MB 本地 / 1200MB 远程 |
| **启动阶段** | Stage-based 服务启动（BOOT→MOUNT→RUN→FOTA），支持 poweroff 依赖校验 |

---

## 4. 构建流程

```
niobuild build -e platform=<TYPE> -enpt_version=<VER> -enpt_platform=<PLAT>
    │
    ├─ conan install + build
    │     ├─ StageManager.gen_configs()     → /etc/stage/domain.json
    │     ├─ VariantsService.gen_variant_... → /etc/variant-service/...
    │     ├─ NetConf.gen_resolv_conf()      → /etc/dns/*/resolv.conf
    │     ├─ Xcp.gen_xcp()                  → xcp_gen/ (A2L/hex)
    │     ├─ SecPol.secpol_pack()           → policy.bin
    │     └─ DiskImage.do_disk_image()      → emmc.img + output.yml
    │
    └─ 镜像签名 (Secure mode only)
          ├─ do_secure_sign() → emmc.img signed
          └─ uboot_env.ini included
```

**关键构建变量**:
- `TARGET_NPT_PLATFORM`: premium / lite
- `TARGET_NPT_VERSION`: 如 `rl010333`
- `TARGET_VEH_CODE`: 车型代码
- `TARGET_VEH_LIST`: 逗号分隔车型列表

---

## 5. 安全模型

### 5.1 DAC (Discretionary Access Control)
- `dac.py` 解析 `sysuser.json`，生成 `/etc/passwd` 和 `/etc/group`
- 服务 credential 定义运行用户/组/supplementary_groups/umask
- 系统用户 UID 200-999 不可登录

### 5.2 Security Policy
- `secpol.py` 调用 `secpol_tool` 打包安全策略为 `policy.bin`
- 配置在 `safeos-base/platform/<PLAT>/boot/`

### 5.3 Secure Boot
- `Secure=True` 时启用 SECBOOT（uboot + mrtos + hypervisor 签名验证）
- `OSF_TEST=True` 时禁用安全验证用于测试

---

## 6. 关键文件索引

| 文件 | 作用 |
|------|------|
| `conanfile.py` | 主 recipe，定义镜像构建流程 |
| `modules/diskimage.py` | 磁盘镜像生成核心 |
| `modules/stage/stage_manager.py` | 服务编排 + domain.json 生成 |
| `modules/stage/dac.py` | /etc/passwd/group 生成 |
| `modules/netconf/netconf.py` | DNS 配置生成 |
| `modules/netconf/generate_conf.py` | resolv.conf 模板引擎 |
| `modules/xcp.py` | XCP 后构建入口 |
| `xcp_user/tools/code_gen/generator.py` | XCP A2L/hex 生成器 |
| `modules/variant_service/variant.py` | 车型变体配置 |
| `vft-config/entryplus/QINLING/etc/comm-control/config.json` | TSP endpoint 配置 |
| `vft-config/vehicleid/0x3001/etc/network/topology.yaml` | 网络拓扑（hosts/switches/links） |
| `modules/netconf/host_vlan.yaml` | VLAN/IP/QoS/DNS/路由配置 |
| `config/stage/CodingMapSimple.json` | 车型编码映射 |
| `config/stage/sysuser.json` | 用户/组 DAC 定义 |
