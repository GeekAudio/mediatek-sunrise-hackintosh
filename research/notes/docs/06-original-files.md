# 06 · 原帖文件索引（“相关文件”）

原帖（tonymacx86 帖子 332855）里的可下载文件全部以**论坛附件**形式散落分布，两位核心开发者（Elias64Fr / CaseySJ）均无 GitHub。本表按帖子页码归档，方便回溯。

## 蓝牙固件 / kext

| 文件 | 来源楼层 | 直链 |
|---|---|---|
| `BT_RAM_CODE_MT7922_1_1_hdr.bin.zip`（楼主附，Linux 版 MT7922 蓝牙固件） | 第 1 页 OP 附件 | https://www.tonymacx86.com/attachments/bt_ram_code_mt7922_1_1_hdr-bin-zip.597230/ |
| `Apple Bluetooth Sunrise Firmware Tahoe 26.2.zip`（楼主从 Tahoe 26.2 蓝牙 DEXT 提取） | OP EDIT #3 附件 | https://www.tonymacx86.com/attachments/apple-bluetooth-sunrise-firmware-tahoe-26-2-zip.597496/ |
| `BlueToolFixup_Mediatek_SRS_Tahoe.kext.zip`（CaseySJ 改装版，同时支持 3 组 VID/PID，第 3 组含 `0E8D:0616`） | 第 17 楼（post 2433923） | https://www.tonymacx86.com/attachments/bluetoolfixup_mediatek_srs_tahoe-kext-zip.597786/ |

## SSDT / config

| 文件 | 来源楼层 | 说明 |
|---|---|---|
| `SSDT-ARPT_RP04_Mediatek.aml` | 第 20 楼（post 2433963，Jan 21 修正版） | 把 Wi-Fi device-id 伪造成 `0x7922`；其他 root port 改 4 行即可；MT7925 改用 `22790000`。直链：https://www.tonymacx86.com/attachments/ssdt-arpt_rp04_mediatek-aml.597811/ |
| `SSDT for RP04`（初版，CaseySJ 自建版参照） | 第 18 楼（post 2433927） | 帖子附件 |
| `config_f10.plist` | 第 19 楼（post 2433942，f10） | f10 的成功配置（iMac20,2 / macOS 26.3 beta / KDK_26.3 kext），复现 AWDL 的关键样本 |

## 调试日志（学习判读的素材）

| 文件 | 来源楼层 | 内容 |
|---|---|---|
| `AppleSunriseWLAN_LOG00.zip` | 第 7 楼（post 2433590） | 注入 `AppleSunrise.kext` 后解锁的首份 `drv.I` 日志 |
| `LogSunrise00.txt.zip` | 第 18 楼（post 2433928） | CaseySJ 自建 SSDT 后的 `request_firmware` 失败日志 |
| `LogSunrise00.txt.zip`（11.5 KB） | 第 20 楼（post 2433949） | `mtk.wlan.startdelay=30` 下 Fenvi AX5400 的日志 |
| `MediaTak-Bluetooth.jpg` | 第 9 楼（post 2433688） | CaseySJ 的 MediaTek 蓝牙工作截图 |

## 外部资源

| 资源 | 链接 |
|---|---|
| Linux MediaTek 固件库（`WIFI_MT7922_*` / `BT_RAM_CODE_MT7922_*` 唯一稳定来源） | https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek |
| 原帖用户购买的 MT7922 M.2 卡（Amazon，~$23） | https://www.amazon.com/dp/B0BXLFX78J |
| Apple KDK（提取 `AppleSunrise.kext`；实测用 `KDK_26.3_25D5087f`） | https://developer.apple.com/download/all/ |
| 第三方 Wi-Fi kext | https://github.com/flxxxxddd/SimpleMtkWlan （分支 `mt7921-fixes`，认准 release） |
| 第三方蓝牙 kext | https://github.com/laobamac/SimpleMtkBtFw |
| Wi-Fi 菜单栏客户端 | https://github.com/laobamac/SimpleMTKClient |

> ⚠️ 本仓库不重新分发上述二进制；请从原帖或上述官方渠道获取，并自行承担实验风险。
