# gauguin-mainline

> Xiaomi Redmi Note 9 Pro 5G（国行 / Mi 10T Lite / Mi 10i，codename **gauguin**，SM7225 骁龙 750G）
> 目标：**主线 Linux 6.12 LTS 内核 + 最新 AOSP（Android 17）**，替代 4.19 下游内核。

## 设备/平台要点

| 项目 | 值 |
|---|---|
| SoC | Qualcomm SM7225（SD750G，SM7250 同族，Kryo 570 / Adreno 619） |
| 关键有利条件 | 与 **Fairphone 4** 同 SoC，mainline 已有 `sm7225` 平台与 FP4 DTS 基础 |
| boot 格式 | boot.img header v2，pagesize 4096 |
| 官方 ROM 内核 | 4.19（LineageOS 直至 lineage-24.0 仍在用） |
| mainline 支持组织 | `sm6350-mainline` / `sm7150-mainline`（GitHub org） |

## 上游资源索引（已核实）

### 内核（close-to-mainline）
- [sm6350-mainline/linux](https://github.com/sm6350-mainline/linux) — SM6350/SM7225 主线内核 fork，**pmOS 官方内核包 `linux-postmarketos-qcom-sm6350` 的源**（tag `v6.11.0-sm6350`），gauguin 设备树随包携带
- 分支覆盖：`sm6350-6.12.y`（**6.12 LTS，本项目目标基线**）一直到 `sm6350-7.2.y`（2026-10 仍在推进）
- [UsrBinLuna/linux-sm6350](https://github.com/UsrBinLuna/linux-sm6350) — gauguin mainline 主力开发者（Zephyr1112/UsrBinLuna）的工作 fork
- gauguin DTS：`arch/arm64/boot/dts/qcom/sm7225-xiaomi-gauguin.dts`（`deviceinfo_dtb=qcom/sm7225-xiaomi-gauguin` 证实存在）

### 设备/固件
- [UsrBinLuna/pmaports-gauguin](https://github.com/UsrBinLuna/pmaports-gauguin) — pmOS 设备包（deviceinfo / APKBUILD）
- [UsrBinLuna/GauguinFirmware](https://github.com/UsrBinLuna/GauguinFirmware) — Adreno a619 固件（GMU/SQE/zap，复用 FP4 固件方案）
- [Nura Wiki: xiaomi-gauguin](https://wiki.nura.eco/wiki/Xiaomi_Mi_10i_(xiaomi-gauguin)) — 硬件 mainline 状态表（2026-10-14 更新）

### AOSP 侧
- [LineageOS/android_device_xiaomi_gauguin](https://github.com/LineageOS/android_device_xiaomi_gauguin) — 官方支持 lineage-18.1 → **lineage-24.0（Android 17）**
- [LineageOS/android_kernel_xiaomi_gauguin](https://github.com/LineageOS/android_kernel_xiaomi_gauguin) — 下游 4.19 内核（active，2026-10）
- [PixelOS gauguin](https://pixelos.net/download/gauguin) — 16.2（Android 16），月更，2026-06 仍活跃
- [TheMuppets/proprietary_vendor_xiaomi_gauguin](https://github.com/TheMuppets/proprietary_vendor_xiaomi_gauguin) — vendor blobs

## 硬件 mainline 现状（Nura Wiki 实测）

| 硬件 | 芯片 | mainline 状态 |
|---|---|---|
| 屏幕（IPS LCD） | Novatek NT36672C（DSC video 面板） | ✅ 可用 |
| 3D 加速 | Adreno 619（freedreno） | ✅ 可用 |
| USB | 外设模式 | ✅ 部分（仅 device 模式） |
| UFS | SK hynix | ✅ 可用 |
| 触摸 | NT36672C (SPI) | ❌ 半成品（sm7150-mainline 驱动改到一半） |
| WiFi/BT | WCN3991 | ❌ 无驱动 |
| 音频 | WCD9375 + Cirrus CS35L41 | ❌ 无驱动 |
| 电池/充电 | SMB1395 + PM7250B | ❌ 无驱动 |
| 传感器 | ICM-40607 等 | ❌ 无驱动 |
| 指纹 | Goodix GF36xx | ❌ 无驱动 |
| 相机 | IMX682 / HM2 / OV16A1Q（OV8856 有驱动可用） | ❌ 主摄无驱动 |
| Modem/GPS | SDR735 | ❌ 完全无支持 |

## 技术路线

**路线 A（验证场）**：`pmbootstrap` + pmaports-gauguin，先以 postmarketOS 把 6.12 内核跑起来
（`fastboot boot` 临时引导，不刷机），驱动开发、外设攻坚都在这里进行。

**路线 B（目标形态）**：LineageOS 24.0（Android 17）设备树 + 主线 6.12 内核替换 4.19：
1. 用 `sm6350-6.12.y` 构建内核 + `sm7225-xiaomi-gauguin` DTB
2. `mkbootimg --header_version 2` 打包 boot.img（cmdline 参考 deviceinfo：`clk_ignore_unused pd_ignore_unused regulator_ignore_unused` 等）
3. 屏蔽依赖下游内核接口的 HAL（camera / sensors / GPS / fingerprint），保留 framework + SurfaceFlinger(freedreno) + generic ramdisk
4. 每攻破一个外设驱动，同步回 AOSP 构建

## 阶段计划

- **P0** 解锁 BL（Mi Unlock，168h 冷却期，期间勿退账号/重启流程）；备好 9008 救砖与 UART
- **P1** pmbootstrap 构建 pmOS gauguin，`fastboot boot` 验证屏幕/GPU/USB
- **P2** 内核基线切到 `sm6350-6.12.y`（6.12 LTS），产出可复现的 boot.img
- **P3** LineageOS 24.0 源码树内替换内核为 mainline 6.12，开机进 UI
- **P4** 外设攻坚（按收益排序）：触摸 NT36672C → WiFi/BT WCN3991（与 WCN3990 同族，ath10k SNOC 是可试方向）→ 音频 WCD9375 → 电池/充电 → modem → 相机
- **P5** 发布：GitHub Releases 提供 boot.img / 说明 / 内核源 fork

## 风险与边界

- 触摸、WiFi、BT、音频、modem、相机当前在 mainline 均不可用——"焕然一新"指系统底座（6.12 LTS + Android 17 框架 + freedreno），日用完整度取决于 P4 推进
- SM7225 的 5G modem 在 mainline 无任何支持，短期内无法恢复
- AOSP HAL 与主线内核 ABI 差异集中在 vendor HAL；framework 侧无影响

## 上游致谢

UsrBinLuna (Zephyr1112)、DomHeadroom、sm7150-mainline / sm6350-mainline 社区、postmarketOS、LineageOS 团队。
