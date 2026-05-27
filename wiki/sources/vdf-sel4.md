---
type: source
source-type: github
created: 2026-05-27
title: "VDF seL4 — 基于 seL4 微内核的 NVOS 发行版"
date: 2026-05-27
size: small
path: raw/vdf/vdf-sel4
summary: "VDF seL4: 基于seL4微内核的NVOS发行版，编译(niobuild)、切换版本(vsel4.xx.xx)、刷版(sel4flash)"
tags: [vdf, sel4, nvos, embedded, vehicle, build, flash]
sources: []
---

# VDF seL4 — seL4 微内核 NVOS

## 概述

VDF seL4 是基于 seL4 微内核的 NVOS (NIO Vehicle Operating System) 发行版，用于 NIO 车辆的嵌入式系统。

## 核心工具

### niobuild — 构建工具

```bash
pip3 install niobuild -i https://artifactory.nioint.com/artifactory/api/pypi/dd-pypi-all-virtual/simple -U
```

### 版本切换

```bash
niobuild -d version -v vsel4.00.06.00 -c RC_05 -u
```

### 编译

```bash
niobuild build -e platform=vdc1a -enpt_version=rl010333 -enpt_platform=lite     # DOM 车型
niobuild build -e platform=vdc1a -enpt_version=rl010333                       # LEO 车型
```

### 刷版

```bash
sel4flash -b board_s -w pre_norflash    # 刷 pre_norflash
sel4flash -b board_u -w emmc            # 刷 emmc
sel4flash -b board_u -w emmc_boot       # 刷 emmc_boot
sel4flash -b board_u -w norflash        # 刷 norflash
```

## 编译问题排查

```bash
rm -rf ~/.docker_local    # 清理 Docker 缓存
rm -rf ~/.conan           # 清理 Conan 缓存
pip3 uninstall niobuild   # 重装 niobuild
pip3 install niobuild -i https://artifactory.nioint.com/artifactory/api/pypi/dd-pypi-all-virtual/simple -U
```

## 参考文档

- [seL4 SDK使用指南](https://nio.feishu.cn/docx/WRd2dasVmop0hPxxHtwcWNs6nfh)
- [基于sel4的NT3应用开发流程](https://nio.feishu.cn/wiki/wikcncPtpyMhVlDdHkFx9w3j4kg)
- [测试FAQs](https://nio.feishu.cn/docs/doccnHNKrpLjXIe7wqGHuXExqae)

## 相关页面

- [[vdf-index]] — VDF 模块索引
- [[entities/linux/safeos/safeos-nsv]] — SafeOS NSv 网络服务器
