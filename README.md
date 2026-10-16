# MT7922 Sunrise Pack

> macOS 26（Tahoe）黑苹果 **MediaTek MT7922 / RZ616 Wi-Fi 6E + Bluetooth 5.x** 驱动文件与 OpenCore 补丁合集。
> 基于 Apple 原生 Sunrise 驱动（`AppleSunriseWLAN.dext` + `AppleSunrise.kext`），让原厂 M.2 网卡模块免第三方 kext 被系统识别。

**兼容性**：macOS 26.0 – 26.x ｜ 芯片：MT7922（AMD RZ616）、MT7925（RZ717，device-id 伪装为 `0x7925`）

| 你的情况 | 需要什么 |
|---|---|
| **macOS 26.7.1+，MT7932/RZ717 卡** | ✅ 什么都不需要——WLAN dext 已内置 IZUBA(MT7932) Wi-Fi 固件，插卡重启即可 |
| **macOS 26.7.1+，MT7922/RZ616 卡** | ⚠️ **Wi-Fi 固件仍缺失**（26.7.1 25G241 实测：WLAN dext 内无 `*7922*` 文件）→ 仍需 `firmware/`；蓝牙已内置无需操作 |
| macOS 26.0 – 26.6（任意 MT792x） | 本仓库全部：`firmware/` + `ssdt/` 或 `patches/` + KDK 中的 `AppleSunrise.kext` |
| macOS 15 Sequoia | 不适用原生驱动——走第三方 kext 路线（见文末） |

## 快速开始

```bash
# ① 系统/硬件自检
ls /System/Library/DriverExtensions/ | grep -i sunrise
ioreg -l -p IODeviceTree | grep -i "14c3"
sw_vers -productVersion
```

- 若 `AppleSunriseWLAN.dext` 已存在且系统为 26.7.1+、卡是 MT7932：插卡重启，`kmutil showloaded --list-only | grep -i sunrise` 有输出即完成。
- 其余情况（含 MT7922 的 Wi-Fi 固件仍缺）：按 **[INSTALL.md](INSTALL.md)** 五步安装（约 15 分钟，需要重启两次）。

## 包含内容

| 目录 | 文件 | 说明 |
|---|---|---|
| `firmware/` | `WIFI_MT7922_patch_mcu_1_1_hdr.bin` | Wi-Fi MCU 补丁固件（134 KB） |
| | `WIFI_RAM_CODE_MT7922_1.bin` | Wi-Fi RAM 固件（1.1 MB，写入时改名 `W7922_1.bin`） |
| | `BT_RAM_CODE_MT7922_1_1_hdr.bin` | 蓝牙固件（505 KB，USB 蓝牙走 `/etc/bluetool`） |
| | `SHA256SUMS` | 校验和 |
| `ssdt/` | `SSDT-ARPT_RP04_Mediatek.aml` | device-id 伪装 SSDT（原帖修正版，适用 RP04 端口） |
| `patches/` | `Kernel-Add-AppleSunrise.xml` | 注入 `AppleSunrise.kext` 的 config 片段 |
| | `DeviceProperties-Add.xml` | 免编译 device-id 伪装（OpenCore DeviceProperties） |
| | `boot-args.txt` | 全部相关启动参数 |
| `research/` | — | 社区调试记录（出处与原理，发布包不含） |

## 使用方法

见 **[INSTALL.md](INSTALL.md)**。核心思路：

1. 从 **KDK** 取 `AppleSunrise.kext`（Apple 官方，匹配系统版本）→ OpenCore 注入
2. 用 **SSDT** 或 **DeviceProperties** 把 device-id 伪装成 `0x7922`
3. 关 SIP/authenticated-root，把本仓库固件写入系统卷并 bless 快照
4. 加 boot-arg `wlan.debug.enable=0x1` 排错

## FAQ

**Q：macOS 26.7.1+ 还要写固件吗？**
分芯片：26.7.1（25G241）实测——蓝牙 dext 已含 `BT_RAM_CODE_MT7922_1_1_hdr.bin`，WLAN dext 已含 **IZUBA(MT7932)** 一套 Wi-Fi 固件；但 WLAN dext 内**没有** `*7922*` 文件，所以 **MT7922 的 Wi-Fi 固件仍要按本包补齐**（日志仍会 `Failed to open WIFI_MT7922_patch_mcu_1_1_hdr.bin`）。

**Q：不想动系统卷，写入固件有替代方案吗？**
有，按推荐顺序：

1. **换 MT7932（RZ717）卡**——26.7.1 已内置其全部 Wi-Fi 固件，真正零操作。
2. **走第三方 kext 路线**：`SimpleMtkWlan` + `SimpleMtkBtFw` 把固件打包在 kext 里，只动 EFI 分区、**完全不碰系统卷**。代价：以太网式接口（无原生 Wi-Fi 菜单）、无 AirDrop/Handoff/Continuity、无 802.11ax。
3. **只要蓝牙的话**：`/etc/bluetool` 放 `BT_RAM_CODE_MT7922_1_1_hdr.bin` 即可（26.7.1+ 连这都不用，蓝牙固件已内置）。
4. 仍要用原生驱动：本包的 bless 快照法本身就是**可回滚**写法（快照启动、恢复 SIP 后重启自动清理），比长期关闭 SIP 安全；且 `wifi.cfg` 可不写（非致命），只需两个 `.bin`。

**Q：Wi-Fi 认到了但连不上/没菜单？**
原生 Sunrise 走系统原生 Wi-Fi 菜单；排错时加 `patches/boot-args.txt` 里的 `wlan.debug.enable=0x1`，
用 `log show --style compact --debug --last boot --info --predicate 'drv.'` 看日志。
常见错误含义见 `research/notes/docs/02-debug-method.md` 的日志判读表。

**Q：注入 kext 后开机崩溃 / DEXT 反复重启？**
加 `mtk.wlan.startdelay=30`（部分卡需 45）；终极手段 `mtk.wlan.hangonstart=1`。

**Q：日志刷屏 `wlanCfgSet ... no empty entry`？**
缺 `wifi.cfg`（Apple 私有配置，不随本包分发）。实测不致命；但仅补固件、缺 wifi.cfg 仍可能失败。

**Q：AirDrop / Handoff 能用吗？**
原生路线理论上支持 AWDL（社区在 26.3 beta 上见过完整 AWDL 证据），但 MT7922 实测成功率有限，
不敢保证，请以实际体验为准。

**Q：我的网卡是 Intel/Broadcom？**
本包仅适用 MediaTek MT792x。

## 备选：第三方路线

Sequoia、或蓝牙走 USB 的老模块：`flxxxxddd/SimpleMtkWlan`（Wi-Fi，认准 release）+ `laobamac/SimpleMtkBtFw`（蓝牙）。
代价：无原生 Wi-Fi 菜单、无 AirDrop/Continuity、无 802.11ax。

## 致谢

- **Elias64Fr**（逆向 bluetoothd / DEXT、发现固件缺失、固件注入法、SSDT 修正版）
- **CaseySJ**（HAL/DEXT 架构分析、全套实测与排错、boot-args）
- **Sn0wMan / f10 / gandem / jaymonkey**（硬件实测与日志）
- 原帖：<https://www.tonymacx86.com/threads/wifi-6-e-bluetooth-5-x-mediatek-chipset-on-native-sequoia-tahoe.332855/>
- 固件来源：<https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek>

## License

- 仓库文本与补丁：[MIT](LICENSE)
- `firmware/*.bin`：linux-firmware 再分发条款（见 `firmware/README.md`）
- `ssdt/SSDT-ARPT_RP04_Mediatek.aml`：原作者 Elias64Fr 于上述帖子公开发布，使用时请署名
