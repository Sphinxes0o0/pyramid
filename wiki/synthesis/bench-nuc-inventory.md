---
type: inventory
tags: [bench, nuc, hardware, power, serial]
created: 2026-05-29
updated: 2026-05-29
---

# Bench NUC 台账

> 实时采集于 2026-05-29

## 可达 Bench（8 台）

| Bench | IP | Kernel | relay | usbrelay | CCC | Serial | sudo NOPASSWD |
|-------|-----|--------|-------|----------|-----|--------|:---:|
| scbench2 | 10.110.32.11 | 6.8.0-111 | ❌ 未安装 | ✅ HW341 | ✅ | ttyUSB0,1 | ❌ |
| scbench3 | 10.110.38.148 | 6.2.0-26 | ✅ | ✅ ID:"1" hidraw0 | ✅ | ttyUSB0,1 | ✅ |
| scbench5 | 10.110.36.120 | 6.2.0-26 | ✅ | ✅ 6QMBS | ✅ | ttyUSB0 | ✅ |
| scbench7 | 10.110.36.253 | 6.8.0-50 | ✅ | ✅ **特殊CLI** | ❌ | — | ? |
| scbench8 | 10.110.38.97 | 5.15.0-117 | ✅ | ✅ (无设备) | ✅ | ttyUSB0,1 vdf1a vdf1m | ✅ |
| scbench9 | 10.160.43.25 | 6.5.0-18 | ✅ | ✅ (无设备) | ❌ | — | ✅ |
| scbench11 | 10.110.38.156 | 6.2.0-26 | ✅ | ✅ 6QMBS hidraw2 | ✅ | — | ✅ |
| DevTower | 10.110.16.87 | 6.17.0-14 | ❌ | ❌ | ❌ | — | N/A |

## 不可达（5 台，SSH timeout）

| Bench | IP |
|-------|-----|
| scbench1 | 10.110.180.243 |
| scbench4 | 10.110.181.194 |
| scbench6 | 10.110.36.22 |
| scbench10 | 10.110.37.30 |
| desknuc | 10.110.16.238 |

## relay CLI 统一格式

所有有 relay 的 bench 使用相同接口：
```
relay {01,02,03,04,05,06,07,08,all} {on,off}
```
需要 `sudo`，但 scbench3/5/8/9/11 已配 NOPASSWD（`sudo -n relay` 直接返回 usage）。

## usbrelay 设备 ID 差异（关键！）

| Bench | Device ID | 路径 | CLI 格式 |
|-------|-----------|------|----------|
| scbench2 | HW341 | — | `usbrelay HW341_N=0/1` |
| scbench3 | 1 | /dev/hidraw0 | `usbrelay 1_N=0/1` |
| scbench5 | 6QMBS | — | `usbrelay 6QMBS_N=0/1` |
| scbench11 | 6QMBS | /dev/hidraw2 | `usbrelay 6QMBS_N=0/1` |
| scbench7 | — | — | `usbrelay {01..08} {on,off}` (**格式不同!**) |
| scbench8 | (未检测到) | — | 不确定 |
| scbench9 | (未检测到) | — | 不确定 |

⚠️ scbench7 的 usbrelay 是 `usbrelay 01 on` 格式，与其他 bench 完全不同。

## Power Channel 映射（来自 board.yml）

所有已配置 bench 使用相同映射：
```
relay 01, 02    → CCC 主电源 (sudo required)
usbrelay CH1,2  → 启动模式: 0/0=uboot, 1/1=serial boot
```

## Serial 设备

| Bench | 串口 |
|-------|------|
| scbench2 | /dev/ttyUSB0, ttyUSB1 |
| scbench3 | /dev/ttyUSB0, ttyUSB1 |
| scbench5 | /dev/ttyUSB0 |
| scbench8 | /dev/ttyUSB0, ttyUSB1, **vdf1a, vdf1m** ⭐ |
| scbench7/9/11 | 无 |

scbench8 有 VDF 专用串口 (`vdf1a`, `vdf1m` — CH9344 芯片)，说明接了 VDF 硬件。

## CCC Testbench 配置

有 CCC 的 bench 使用默认配置 `default_ccc_B1_leo.yaml`:
```yaml
vdf:
  - ecu_name: "VdfMPUSel4Hw"
    hardware: "B1"
    serial:
      port: "/dev/vdf1-a"
    dut_power:
      dp_on: "power_ccc_a_on"
      dp_off: "power_ccc_a_off"
```

`power_ccc_a_on/off` 是包装函数（非 relay 直接调用），需要进一步查具体实现。

## bench_tool（核心发现）

所有 bench 使用相同代码库 `/home/nio/dev-tools/dps-dev-bench/`：

| 文件 | md5 一致性 | 说明 |
|------|:---:|------|
| `bench_tool` | **4/5 一致** | 主脚本，scbench8 有自定义版本 |
| `bench_config.json` | 全不同 | 每台 bench 独立配置 |
| `bench1/relay` | **5/5 一致** | relay 控制模块 |
| `bench1/board.yml` | **5/5 一致** | 板卡配置 |
| `~/.local/bin/bench_tool` wrapper | **5/5 一致** | 路径封装器 |

### bench_tool CLI 接口（5 台完全一致）

```
bench_tool -p [zone] [state]    # 电源控制
  zone: ccc, fzone, lzone, rzone, xzone, bench
  state: on, off

bench_tool -s [port]            # 串口连接
  port: vdfa, vdfm, vdf2, fte, ftm, ree, rem, lee, lem, rie, rim, xzone

bench_tool -f [param]           # 烧录 VDF A core
  param: emmc, uboot, all

bench_tool -m                   # 烧录 VDF M core
bench_tool -r                   # 重启 CCC + 连接串口
bench_tool --ping               # 检测 ECU 网络
bench_tool --debug              # dry-run 模式
```

### 两套系统对比

| | ccc_testbench | dps-dev-bench (bench_tool) |
|:---|:---|:---|
| 位置 | `/home/nio/ccc_testbench/` | `/home/nio/dev-tools/dps-dev-bench/` |
| 电源控制 | `sudo relay 01 off && sudo relay 02 off` | `bench_tool -p ccc off` (内部调用 relay+usbrelay) |
| 串口端口 | `/dev/vdf1-a` | `/dev/ttyCH9344USB0` |
| usbrelay | HW341_1=0（board.yml 写死） | 6QMBS_1=0（实际 device ID） |
| 抽象层 | 无（直接调 relay） | 有（zone 抽象） |
| 适用范围 | 仅 CCC 主电源 | ccc + 6 个 zone + bench |

### ⚠️ bench1/board.yml vs ccc_testbench board.yml

两者是两套独立的配置：
- `ccc_testbench/flash_tool/board.yml` — relay 01,02 + sudo，面向 CCC 主板
- `dps-dev-bench/bench1/board.yml` — relay 08 + usbrelay 6QMBS，面向 bench 整体

### Phase 1 关键启示

**不需要直接包装 relay/usbrelay**。bench_tool 已经提供了统一的 zone-level 电源抽象：
- `bench_tool -p ccc on/off` → Phase 1 直接用这个
- 适配到 ExternalPowerDriver：`cmd_on="bench_tool -p ccc on"` 即可

scbench2 没有 relay 二进制但有 bench_tool，说明 bench_tool 内部可能走不同路径。

## 账号密码

| 类型 | user/pass | 适用 Bench |
|------|-----------|------------|
| 默认 | nio / nioforfota@123 | scbench3-11 |
| scbench2 专用 | nio / NIOyilai321 | scbench2 |
| root | root / nioforfota@123 | 部分 bench |
| DevTower | sphinx / sphinx | DevTower |
