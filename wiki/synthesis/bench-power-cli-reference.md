---
type: reference
tags: [labgrid, bench, power, relay, usbrelay, phase1]
created: 2026-05-29
---

# Bench 电源控制 CLI 参考

> 从 scbench11 (10.110.38.156) 实测采集

## relay CLI

```
usage: relay [-h] {01,02,03,04,05,06,07,08,all} {on,off}
```

- 8 通道 + `all`
- 仅支持 on/off，无状态查询、无 cycle
- **需要 sudo**
- Channel 映射（scbench11）:

| Channel | 用途 |
|---------|------|
| 01 | CCC 主电源 |
| 02 | CCC 主电源 |

## usbrelay CLI

设备: USBRelay8 (16c0:05df) @ /dev/hidraw2
序列号: 6QMBS (每台 NUC 不同)

用法: `usbrelay <DEVICE_ID>_<CHANNEL>=<0|1>`
例: `usbrelay 6QMBS_1=0` (断) , `usbrelay 6QMBS_1=1` (通)

无需 sudo。

Channel 映射（来自 board.yml）:

| Channel | 用途 |
|---------|------|
| 1 | 启动模式选择 bit0 |
| 2 | 启动模式选择 bit1 |

模式编码:
- uboot boot: CH1=0, CH2=0
- serial boot: CH1=1, CH2=1

## Power Cycle 时序

从 board.yml 的 reboot 命令:

```
① relay 01 off + relay 02 off     (主电源断电)
② sleep 5                         (等待电容放电)
③ usbrelay 设置启动模式            (0/0 = uboot, 1/1 = serial)
④ sleep 5                         (等待模式稳定)
⑤ relay 01 on + relay 02 on       (主电源上电)
```

总计约 10 秒 cycle。

## Domain 访问方式

| Domain | 访问方式 | 地址 |
|--------|---------|------|
| VDF | telnet | 172.20.10.1 |
| CDF | adb + serial | - |
| ADF | serial /dev/ttyUSB0 + socctrl | - |
| SAF | SSH | 见 saf*.sh |
| Zone | telnet | 192.168.1.5:5400 |
