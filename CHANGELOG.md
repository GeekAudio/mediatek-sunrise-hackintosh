# Changelog

本项目遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 与语义化版本。

## [1.0.0] - 2026-10-16

首个正式版。内容整理自 tonymacx86 帖子 332855（2025-12 至 2026-10，69+ 页实测讨论）。

### 新增
- `README.md`：双语项目说明 + 一分钟结论 + 五步调试法总览
- `docs/00-快速总结.md`：一页式总结
- `docs/01-driver-architecture.md`：Sunrise 驱动架构（DEXT + HAL）、芯片/PCI ID 匹配表、固件清单、自检命令、驱动内部调试设施
- `docs/02-debug-method.md`：**调试五步法完整操作清单**（注入 HAL kext、`wlan.debug.enable=0x1`、SSDT device-id 伪装、日志收集、系统卷 bless 快照注入固件）、boot-args 速查、日志判读表、2026-01 调试结论
- `docs/03-bluetooth.md`：蓝牙三条路线（`/etc/bluetool` 原生法、改装 BlueToolFixup、第三方 SimpleMtkBtFw）+ AirPods 无声修复
- `docs/04-third-party.md`：第三方 Wi-Fi kext 路线（`flxxxxddd/SimpleMtkWlan` 等）与硬限制、排错速查
- `docs/05-timeline.md`：从发现到 26.7.1 开箱即用的完整时间线 + 五条可复用经验
- `docs/06-original-files.md`：原帖附件/固件/配置文件索引（含直链）
- `RELEASE.md`：v1.0.0 Release Notes（可直接粘贴到 GitHub Release）
- `LICENSE`：MIT

### 说明
- 本仓库只收录**整理文本**，不含任何 Apple 私有代码或固件二进制；固件请从 linux-firmware 或自己的系统获取。
- 内容为社区实验记录，涉及关闭 SIP / 修改系统卷，操作前请备份。
