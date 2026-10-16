# 01 · 驱动本体：架构、芯片与固件

## 1. 架构：DEXT + HAL 两层

`AppleSunriseWLAN` 是 macOS 26（Tahoe）引入的 **Apple 自家 MediaTek Wi-Fi 驱动**，由两部分组成：

| 组件 | 形态 | 位置 | 职责 |
|---|---|---|---|
| `AppleSunriseWLAN.dext` | 用户态 DriverKit（dext） | `/System/Library/DriverExtensions/` | Wi-Fi 协议/数据面，与 IO80211 交互 |
| `AppleSunrise.kext` | 内核态 kext（HAL） | **仅 KDK 内有**，系统目录不随附（26.0–26.2 时代） | 硬件抽象层：真正读写 PCIe 寄存器；DEXT 依赖它才能操作硬件 |

> CaseySJ 的关键纠正（2026-01-08）：HAL 是必需的——DEXT 活在用户态，硬件寄存器操作必须由内核态的 HAL 代劳。所以 `AppleSunrise.kext` 本应随正式版驱动一起进驻 System 目录；26.0–26.2 只随 KDK 发布说明"驱动未完工"。

蓝牙对应 `AppleSunriseBluetooth.dext`。

## 2. 芯片与 PCI ID 匹配

| 芯片 | 常见别名 | Wi-Fi PCI ID | 蓝牙：PCIe（新） | 蓝牙：USB（老） |
|---|---|---|---|---|
| MT7921 | — | `14c3:7961` | — | `0x7961` |
| MT7921K | **AMD RZ608** | `14c3:0608` | — | `0x0608` / `0x7961` |
| **MT7922** | **AMD RZ616** | `14c3:7922` | ★ `14c3:792a` | `0x7922` |
| MT7923 | — | `14c3:7923` | `14c3:792b` | — |
| MT7925 / MT7932 | **AMD RZ717**（Wi-Fi 7） | `14c3:7932` / `14c3:7925` | `14c3:793b` | — |

DEXT 的 Info.plist 匹配条件（26.x 实测）：

- `AppleSunriseWLAN.dext`：`IOPCIMatch = 0x792214c3 / 0x792314c3 / 0x793214c3`
- `AppleSunriseBluetooth.dext`：`IOPCIMatch = 0x792a14c3 / 0x792b14c3 / 0x793b14c3`

★ **同一颗 MT7922 有两种硬件形态**：新模块蓝牙挂 PCIe（吃原生 Sunrise 蓝牙驱动），老模块蓝牙走 USB（要靠第三方 `SimpleMtkBtFw`）。

## 3. dext 自带固件（版本差异）

- **26.0–26.2**：Wi-Fi 固件与 `wifi.cfg` **全部缺失**；蓝牙 dext 内只有 `BT_RAM_CODE_MT7922_1_1_hdr.bin`（MT7922）。
- **26.7.1（2026-10 实测）**：
  - `AppleSunriseBluetooth.dext/`：`BT_RAM_CODE_MT7922_1_1_hdr.bin`、`BT_RAM_CODE_MT7923_1_1_hdr_ipc.bin`
  - `AppleSunriseWLAN.dext/IZUBA/`（MT7932 整套 Wi-Fi 固件）：`IZUBA_WIFI_MT7932_patch_mcu_1_2_hdr.bin`、`IZUBA_WIFI_RAM_CODE_MT7932_1_2_idxlog.bin`、`IZUBA_EEPROM_MT7932_1.bin`、`IZUBA_WIFI_MFG_MT7932_2.bin`、`IZUBA_TxPwrLimit_*.dat`、`IZUBA_wifi.cfg`、`IZUBA_db.dat` 等

macOS 是 universal 二进制：`AppleSunriseWLAN` 同时含 ARM 与 x86 两个二进制，说明这是 Apple 为未来产品（传闻中的低价 MacBook / A 系芯片机型）预埋的驱动，x86 黑苹果因此"躺枪"受益。

## 4. 本机自检命令

```bash
# ① 系统有没有原生 Sunrise 驱动？（有 ⇒ 优先原生路线）
ls /System/Library/DriverExtensions/ | grep -i sunrise

# ② 芯片/设备形态
ioreg -l -p IODeviceTree | grep -i "14c3"            # PCIe 侧（含 792a 蓝牙）
system_profiler SPUSBDataType | grep -i -E "MediaTek|0x0e8d"   # USB 侧（老款）

# ③ DEXT 是否加载 / Wi-Fi 接口是否出现
kmutil showloaded --list-only | grep -i sunrise
networksetup -listallhardwareports | grep -A2 -i wi-fi

# ④ 确认卡没被 SSDT 藏掉（vendor-id 应显示 <c3140000>，而非 ffffffff）
ioreg -l | grep -A40 'pci14c3'
```

## 5. 驱动内部调试设施（逆向所得）

- `_wlanDebugInit()` 默认 debug level = `0x2F`（模块位图），privacy level = `0x3E`；由 `_wlanSetDriverDbgLevel()` 设置。
- 日志经 `_kalOsLog` → `vsnprint` → CCLOGStream 输出（logAlert / logCrit / logInfo…），日志前缀形如 `23742172-161-<DRV> [0]request_firmware:...`。
- 驱动识别的 boot-args：`wlan.debug.enable`、`mtk.wlan.hangonstart`（无限循环自锁）、`mtk.wlan.startdelay`（延迟启动）。
