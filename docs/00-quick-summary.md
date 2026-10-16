# Apple MediaTek Sunrise 驱动调试方法总结

> 来源：tonymacx86 帖子 332855 *"Wifi 6(E) & Bluetooth 5.x MEDIATEK chipset on native Sequoia & Tahoe"*（楼主 Elias64Fr，CaseySJ / Sn0wMan / f10 等实测，2025-12 至 2026-10，全文 69+ 页）
> 整理日期：2026-10-16 ｜ 适用：macOS Tahoe 26.x 黑苹果 + MT7921/MT7922/MT7925(7932) 无线模块

---

## 0. 一句话方法

**注入 HAL kext 解锁 DEXT 详细日志 → 从日志反推缺失组件 → 让硬件被驱动匹配（SSDT 伪造 device-id）→ 用系统卷快照补入固件 → 逐项排除。**
本质：不猜，让 Apple 自己的驱动把"它想要什么、卡在哪一步"全部写进日志，再按图索骥。

---

## 1. 驱动本体（先认清目标）

| 组件 | 形态 | 说明 |
|---|---|---|
| `AppleSunriseWLAN.dext` | 用户态 DriverKit | Wi-Fi 本体，位于 `/System/Library/DriverExtensions/`，匹配 `14c3:7922 / 7923 / 7932` |
| `AppleSunrise.kext` | 内核态 kext | HAL 硬件抽象层，**只存在于 KDK**，必须 OpenCore 注入，否则 DEXT 无法操作寄存器（CaseySJ 1 月 8 日更正的认识） |
| `AppleSunriseBluetooth.dext` | 用户态 | 蓝牙走 **PCIe**（`14c3:792a/792b/793b`），不是 USB |

芯片速查：MT7921K = AMD RZ608（`14c3:0608`）、MT7922 = AMD RZ616（`14c3:7922`）、MT7925/MT7932 = RZ717（`14c3:7932`）。

---

## 2. 调试五步循环（方法论核心）

```
① 让 DEXT 加载    注入 AppleSunrise.kext + SSDT 伪造 device-id 0x7922（+PXSX 的 DEXT 属性）
        ↓
② 打开详细日志    boot-arg wlan.debug.enable=0x1，log show 收集
        ↓
③ 读日志定位卡点  "Failed to open 某文件" → 缺什么；"wlanRemove" → 驱动已放弃
        ↓
④ 补缺失件        wifi.cfg + WIFI_MT7922_patch_mcu_1_1_hdr.bin 等，关 SIP 挂载系统卷 + bless 快照写入
        ↓
⑤ 排除法兜底      崩溃用 mtk.wlan.startdelay 缓解；与成功样本（f10 的 26.3 beta 配置）对照差异
```

---

## 3. 完整操作清单（2026-01-19/20 实测版）

1. **装模块**：M.2 MT7922 插 M.2→PCIe 转接卡（或直插主板）。
2. **注入 HAL**：从对应版本 KDK 取 `AppleSunrise.kext`（实测用 `KDK_26.3_25D5087f`），OpenCore `Kernel → Add` 注入 → Tahoe 可正常启动。
3. **开日志**：NVRAM 加 boot-arg `wlan.debug.enable=0x1`。
4. **让硬件被匹配**：
   - DEXT 的 Info.plist 只认 `0x7922`；RZ616 原生是 `0x616` → 用 `SSDT-ARPT_RP04_Mediatek.aml`（帖内附件）把 device-id 伪造成 `0x7922`；其他 root port 改其中 4 行即可；**MT7925 伪造为 `22790000`**。
   - PXSX 下需补 DEXT 相关属性，否则 Info.plist 阶段就不匹配（Elias64Fr 指出 CaseySJ 的配置缺项）。
5. **收集日志**：

   ```bash
   log show --style compact --debug --last boot --predicate 'drv.'
   # info 级消息要加 --info；纯 drv. 谓词可能为空（属正常，换 --info 再试）
   ```

6. **向系统卷补固件**（Recovery 模式）：

   ```bash
   csrutil disable
   csrutil authenticated-root disable
   mkdir ~/mount && sudo mount -o nobrowse /dev/diskXsY ~/mount   # 主 Tahoe 卷，非 data 卷
   # 拷入固件，权限 root:wheel 644：
   #   WIFI_MT7922_patch_mcu_1_1_hdr.bin
   #   WIFI_RAM_CODE_MT7922_1.bin → 按驱动请求改名为 W7922_1.bin
   #   wifi.cfg（Elias64Fr：固件参数配置，可选）
   sudo bless --folder ~/mount/System/Library/CoreServices --bootefi --create-snapshot
   ```

7. **崩溃缓解**（0x7922 伪装后 DEXT 每 ~40s 崩溃重启并拖垮系统）：

   | boot-arg | 作用 |
   |---|---|
   | `mtk.wlan.startdelay=30` | 延迟 30 秒启动（Fenvi AX5400 实测可进系统） |
   | `mtk.wlan.startdelay=45` | 同上，华硕 PCE-AXE59BT 需要 45 |
   | `mtk.wlan.hangonstart=1` | 驱动挂起自锁，保证能进系统（终极手段） |

---

## 4. 日志判读表（Debug 的核心）

| 日志行 | 含义 | 对策 |
|---|---|---|
| `DK: AppleSunriseWLAN::start ok` | DEXT 已 attach，HAL 注入成功 | — |
| `Failed to open .../wifi.cfg` | 缺配置文件 | Dext 内补入 |
| `Failed to open .../WIFI_MT7922_patch_mcu_1_1_hdr.bin` | 缺 MCU 补丁固件 | 系统卷快照注入 |
| `request_firmware: Failed to find firmware file` | 同上（另一条路径） | 同上 |
| `wlanCfgSet: wifi config there is no empty entry`（刷屏，LlwLatencyDbg/Nan*/AwdlGenCap） | wifi.cfg 缺失，配置表仅 256 项 | Elias64Fr：**不致命**，但放入固件后会出现更严重错误 |
| `wlanDownloadPatch: FW[2] load error!` | 固件加载失败 | 补固件 |
| `wlanRemove` + `Disabling PCI device` | 驱动放弃并移除设备 | 调试失败的终点信号 |
| `IOUserServer(com.apple.AppleSunriseWLAN) has crashed too many times` panic | DEXT 崩溃循环（Sn0wMan 0x7922 卡实录） | startdelay / hangonstart 缓解 |
| `corecapture: Missing ivars in CCLogPipe` | CoreCapture 日志管道创建问题 | 观察项，非阻塞错误 |

---

## 5. 调试结论（2026-01 定格）

- **根因**：26.0–26.2 的 DEXT 里 **Wi-Fi 固件和 wifi.cfg 全部缺失**，驱动在启动早期即失败。
- **只补固件不够**：缺 wifi.cfg 时先刷屏 `wlanCfgSet` 报错，补了固件后驱动仍以 `wlanRemove` 收场。
- **原生 0x616 即使伪装成 0x7922 也展不开设备树**（反汇编显示 0x616 被同等配置，但行为不同）。
- **最接近成功的样本**：f10 在 macOS 26.3 beta + KDK_26.3 kext 上出现**完整 AWDL 支持**；CaseySJ 尝试复现（对照机型 iMac20,2 vs MacPro7,1、RestrictEvents.kext、HDMI 属性、USB 映射方式差异）。
- **蓝牙侧独立可用**（不依赖 Wi-Fi 调通）：
  - Tahoe 26.0–26.2 认 7 个 USB 设备（Foxconn `0489:E0D8/E0E0/E0F1/E0F2/E0F6`、MediaTek `0E8D:7922`、Azurewave `13D3:3568`）；
  - 固件 `BT_RAM_CODE_MT7922_1_1_hdr.bin` 放 `/etc/bluetool/`（取自 linux-firmware），或直接用改装 `BlueToolFixup`（改第 3 组 VID/PID）；
  - AirPods 无声的修复（CaseySJ 给 Sn0wMan）：**忘记设备 → iPhone/iPad 重新配对 → Mac 菜单栏重连**。

---

## 6. 现状与建议（2026-10）

- **macOS 26.7.1+、蓝牙走 PCIe 的 MT7922/7923/7932 模块：什么都不装**，插卡重启先试。26.7.1 的 DEXT 已自带蓝牙固件（MT7922/7923）和 MT7932（IZUBA）Wi-Fi 固件；帖子第 69 页已有 "It works!" 实测报告。上述 1 月的手动注入流程**已被 Apple 后续更新取代**。
- **Sequoia 或蓝牙走 USB 的老模块**：走第三方路线——`flxxxxddd/SimpleMtkWlan`（fork，分支 `mt7921-fixes`，认准 release，勿用上游主线）+ `laobamac/SimpleMtkBtFw` + SimpleMTK 菜单栏客户端。注意：先确认卡没被 `SSDT-Disable_Network` 藏掉（`ioreg -l | grep -A40 'pci14c3'`），注意 `en0` 归属变化。代价：无原生 Wi-Fi 菜单、无 AirDrop/Handoff/Continuity、无 802.11ax。
- **要 AirDrop / Continuity / Wi-Fi 6**：换 Broadcom 或 Intel 卡。
- **分发渠道备注**：Elias64Fr / CaseySJ **均无 GitHub**，所有 kext/固件/SSDT 以帖子附件形式发布在该帖内。

---

## 7. 参考

- 主帖：https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/ （调试方案在第 17–20 页；第 8 页是固件缺失发现初期讨论）
- Linux 固件库：https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek
- 本机技能库参考：`~/.workbuddy-ai/skills/opencore-config-audit/references/mediatek-wifi.md`
