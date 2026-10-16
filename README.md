# MediaTek Sunrise Wi-Fi / Bluetooth on macOS Tahoe — 黑苹果 MediaTek 原生驱动研究笔记

[中文](#中文说明) | [English](#english-summary)

本仓库整理 **Apple 原生 MediaTek Wi-Fi 6E / Bluetooth 5.x 驱动（内部代号 Sunrise）** 的发现过程、调试方法与截至 2026-10 的现状。

**唯一来源**：tonymacx86 长帖 *[Wifi 6(E) & Bluetooth 5.x MEDIATEK chipset on native Sequoia & Tahoe](https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/)*（楼主 [Elias64Fr](https://www.tonymacx86.com/members/elias64fr.2347319/)，69+ 页，CaseySJ / Sn0wMan / f10 等大量实测）。原帖两位核心开发者均**没有 GitHub 仓库**，所有 kext / 固件 / SSDT 附件都只在帖内分发，本仓库是对其方法论的搬运与结构化整理。

> ⚠️ 实验性内容。涉及关闭 SIP / authenticated-root、修改系统卷，操作前务必备份 EFI 与数据。

## 一分钟结论（TL;DR）

| 场景 | 方案 |
|---|---|
| **macOS 26.7.1+**，MT7922/MT7923/MT7932，蓝牙走 PCIe（`14c3:792a`） | **什么都不装**，插卡重启先试。新版 DEXT 已自带固件，27 楼后有 "It works!" 实测 |
| macOS Sequoia / 蓝牙走 USB 的老模块 | 第三方蓝牙 `SimpleMtkBtFw` +（可选）Wi-Fi `flxxxxddd/SimpleMtkWlan` |
| 需要 AirDrop / Handoff / Continuity / 802.11ax | 换卡（Broadcom 原生 / Intel + AirportItlwm） |

## 调试方法五步循环（核心）

```
① 让 DEXT 加载    注入 KDK 里的 AppleSunrise.kext + SSDT 把 device-id 伪造成 0x7922
        ↓
② 打开详细日志    boot-arg wlan.debug.enable=0x1
        ↓
③ 读日志定位卡点  "Failed to open 某文件" → 缺什么；"wlanRemove" → 驱动已放弃
        ↓
④ 补缺失件        wifi.cfg + 固件，关 SIP 挂载系统卷 + bless 快照写入
        ↓
⑤ 排除法兜底      mtk.wlan.startdelay 缓解崩溃；与成功样本（f10）对照差异
```

## 关键 boot-args

| boot-arg | 作用 |
|---|---|
| `wlan.debug.enable=0x1` | 解锁 `AppleSunriseWLAN.dext` 详细日志（调试开关，非启用开关） |
| `mtk.wlan.startdelay=30` | DEXT 延迟 30s 启动，避开开机崩溃（Fenvi AX5400 实测值；AXE59BT 需 45） |
| `mtk.wlan.hangonstart=1` | 驱动挂起自锁，保证能进系统（终极手段） |

## 文档导航（docs/）

| 文档 | 内容 |
|---|---|
| [docs/01-driver-architecture.md](docs/01-driver-architecture.md) | 驱动架构（DEXT + HAL）、芯片/PCI ID 匹配表、自带固件清单、本机检查命令 |
| [docs/02-debug-method.md](docs/02-debug-method.md) | **调试五步法完整操作清单**、系统卷固件注入、日志判读表、2026-01 调试结论 |
| [docs/03-bluetooth.md](docs/03-bluetooth.md) | 蓝牙路线：`/etc/bluetool` 固件法、改装 BlueToolFixup、AirPods 无声修复 |
| [docs/04-third-party.md](docs/04-third-party.md) | 第三方 kext 路线（SimpleMtkWlan / SimpleMtkBtFw）与其硬限制 |
| [docs/05-timeline.md](docs/05-timeline.md) | 2025-12 发现 → 2026-01 调试 → 2026-10 原生可用的完整时间线 |

## 固件文件速查

| 文件 | 用途 | 来源 |
|---|---|---|
| `WIFI_MT7922_patch_mcu_1_1_hdr.bin` | Wi-Fi MCU 补丁 | linux-firmware / 26.7+ 系统自带 |
| `WIFI_RAM_CODE_MT7922_1.bin`（需改名 `W7922_1.bin`） | Wi-Fi RAM 代码 | 同上 |
| `wifi.cfg` | Wi-Fi 配置（Nan*/Awdl 参数，256 项配置表） | 同上 |
| `BT_RAM_CODE_MT7922_1_1_hdr.bin` | 蓝牙固件（USB 路线放 `/etc/bluetool/`） | linux-firmware / Apple DEXT 提取 |

## 声明与致谢

- 全部事实性内容来自上述公开帖子（Elias64Fr 的逆向 + CaseySJ/Sn0wMan/f10 的实测），本仓库仅做整理，不含任何 Apple 私有代码或固件二进制。
- 固件二进制请从 [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek) 或你自己的系统中获取。
- License: MIT（仅限本仓库的整理文本）。

---

## English Summary

This repo documents **Apple's native MediaTek Wi-Fi/BT driver (codename "Sunrise")** found in macOS Tahoe, distilled from the [tonymacx86 thread #332855](https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/).

- Driver = `AppleSunriseWLAN.dext` (user-space DriverKit) + `AppleSunrise.kext` (kernel-space HAL, KDK-only, inject via OpenCore).
- Supports MT7922/MT7923/MT7932 over PCIe (`14c3:7922/7923/7932`); Bluetooth can also live on PCIe (`14c3:792a`).
- Jan 2026 debugging found the DEXT shipped without Wi-Fi firmware (`wifi.cfg`, `WIFI_MT7922_patch_mcu_1_1_hdr.bin`...). Method: inject HAL kext → `wlan.debug.enable=0x1` → read logs → patch device-id via SSDT → inject firmware via `bless` snapshot.
- As of macOS 26.7.1 (Oct 2026), firmware ships inside the DEXT and native support reportedly works with zero installation.
- Neither Elias64Fr nor CaseySJ (the thread's developers) has a public GitHub; their work was distributed as forum attachments only.
