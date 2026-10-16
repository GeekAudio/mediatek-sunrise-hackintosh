# Release Notes — v1.0.0

> 以下内容可直接粘贴为 GitHub Release 的 Description；发布资产（asset）用同目录生成的 `mediatek-sunrise-hackintosh-v1.0.0.zip`。

---

## Apple MediaTek Sunrise Wi-Fi/BT 驱动研究笔记 v1.0.0

整理自 tonymacx86 长帖 *[Wifi 6(E) & Bluetooth 5.x MEDIATEK chipset on native Sequoia & Tahoe](https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/)*（楼主 Elias64Fr；CaseySJ / Sn0wMan / f10 等实测，2025-12 → 2026-10，69+ 页）。原帖开发者均无 GitHub，所有材料以论坛附件散落分发，本仓库将其结构化。

### 📌 一分钟结论

- **macOS 26.7.1+ / MT7922-7923-7932 / 蓝牙走 PCIe**：原生 Sunrise 驱动已自带固件，**什么都不装**，插卡重启先试（原帖第 69 页已有 "It works!" 报告）。
- **Sequoia / 蓝牙走 USB 老模块**：`flxxxxddd/SimpleMtkWlan` + `laobamac/SimpleMtkBtFw`。
- 要 AirDrop / Continuity / 802.11ax：换卡。

### 🧭 收录内容

| 文件 | 内容 |
|---|---|
| `docs/01` | 驱动架构：`AppleSunriseWLAN.dext`（用户态）+ `AppleSunrise.kext`（HAL 内核态）、芯片/PCI ID 表、固件清单、自检命令 |
| `docs/02` | **调试五步法**：注入 HAL kext → `wlan.debug.enable=0x1` → 读日志定位缺失物 → SSDT 伪造 device-id → bless 快照注入固件；含 boot-args 速查与日志判读表 |
| `docs/03` | 蓝牙：`/etc/bluetool` 法、改装 BlueToolFixup、AirPods 无声修复 |
| `docs/04` | 第三方 kext 路线与硬限制（无原生菜单 / 无 AirDrop / 无 802.11ax） |
| `docs/05` | 完整时间线 + 五条可复用经验 |
| `docs/06` | 原帖附件/固件索引（直链） |

### ⚠️ 注意

- 实验性内容：涉及关 SIP / authenticated-root、`bless --create-snapshot` 修改系统卷，务必备份。
- 仓库只含整理文本；固件二进制请从 [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek) 或你自己的系统获取。
- License: MIT。

### 🔗 链接

- 原帖：https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/
  - 蓝牙发现（2025-12-07）：第 1 页
  - 固件缺失日志（2026-01-07/08）：第 7–8 页
  - **调试五步法实证（2026-01-19/20）：第 17–20 页**
  - AirPods 修复（2026-01-11）：第 9 页
  - 原生可用报告（2026-10）：第 69 页

**Full Changelog**: https://github.com/&lt;your-name&gt;/mediatek-sunrise-hackintosh/commits/v1.0.0
