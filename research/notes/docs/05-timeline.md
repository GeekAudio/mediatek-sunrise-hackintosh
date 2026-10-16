# 05 · 时间线：从发现到开箱即用

| 日期 | 事件 |
|---|---|
| 2025-12-07 | Elias64Fr 逆向 Tahoe 26.0 `bluetoothd`，发现 MediaTek 蓝牙支持（7 个 USB 设备）；`/etc/bluetool` 固件法成形；发帖（原帖诞生，标题只有 Bluetooth） |
| 2025-12-07~08 | 发现 Tahoe 内 **`AppleSunriseWLAN.dext`**（原帖 EDIT #2）——Apple 自家 MediaTek Wi-Fi 驱动首次曝光；CaseySJ 联系到传闻中的低价 MacBook（A 系芯片） |
| 2025-12-08 | tidbit：BlueToolFixup 改装版让 MT7922 蓝牙可用（BT 部分 `0E8D:0616`） |
| 2026-01-07~08 | Ghidra 逆出 `_wlanDebugInit`（默认 log level `0x2F`）；**注入 KDK 的 `AppleSunrise.kext` 后 DEXT `start ok`**，但报 `Failed to open wifi.cfg` / `WIFI_MT7922_patch_mcu_1_1_hdr.bin`；CaseySJ 厘清 HAL 架构（用户态 DEXT + 内核态 HAL）；结论：等 Apple 补固件 |
| 2026-01-09~11 | 大批用户购入 MT7922 卡（AliExpress ~$13 / Amazon ~$23）；Sn0wMan 报告 AirPods 无声，CaseySJ 给出"忘记→iOS 重配→Mac 重连"修复法 |
| 2026-01-19~20 | **调试高峰**：CaseySJ 三步法（装卡 → 注入 `AppleSunrise.kext` → `wlan.debug.enable=0x1`）；`log show` 收日志；SSDT 伪造 device-id 0x7922（MT7925→`22790000`）；固件清单确认；`wlanCfgSet` 刷屏分析；系统卷 bless 快照注入固件；崩溃缓解 `mtk.wlan.startdelay=30/45`、`mtk.wlan.hangonstart=1`；Sn0wMan 的 0x7922 卡 IOUserServer panic 实录 |
| 2026-01-20 | f10 在 **macOS 26.3 beta + KDK_26.3 kext** 上出现**完整 AWDL 支持**截图——全程最接近成功的样本；CaseySJ 开始逐项对照复现 |
| 2026-01-21 | Elias64Fr 发布修正版 `SSDT-ARPT_RP04_Mediatek.aml`（附改 root port 方法） |
| 2026-01-23 | 社区等 1 月 27–29 Apple 发布会（期待随固件的新品，未如愿） |
| 2026 春~夏 | Tahoe 26.3–26.6 陆续更新；蓝牙固件随 dext 落地（26.2 已含 MT7922 蓝牙固件） |
| 2026-10-14/15 | 原帖第 69 页：**26.7.1 上 "It works!"**——原生 Sunrise Wi-Fi 报告可用；SimpleMtkBtFw（USB 蓝牙）在同一张卡上反而未通 |
| 2026-10-16 | 本仓库整理 |

## 教训 / 可复用的经验

1. **DEXT ≠ kext**：DriverKit  dext 的崩溃会被 `IOUserServer` 看门狗兜住重试，但反复崩溃仍能拖垮系统（0x7922 案例）；HAL 缺失/版本不匹配会在 info.plist 匹配之前就失败。
2. **日志级别位图**：`0x2F` 这类模块位图 + `wlan.debug.enable=0x1` 才能放出 `drv.I` 详细日志——没有日志时先怀疑"没解锁"而非"没报错"。
3. **改系统卷的正确姿势**：关 SIP + authenticated-root → 挂**主卷**（快照在其子树上，非 data 卷）→ 拷文件（`root:wheel` 644）→ `bless --create-snapshot`；不需要编译内核。
4. **device-id 是驱动匹配的第一道门**：先 `ioreg` 看实际 ID，再决定 SSDT 伪装目标值。
5. **区分硬件形态**：同一颗 MT7922，蓝牙在 PCIe 还是 USB，决定了你该用原生驱动还是第三方 kext。
