# INSTALL — MT7922 Sunrise Pack 安装说明

适用：macOS 26.0–26.6 + MT7922（RZ616）M.2 模块，想让系统**原生** Sunrise 驱动工作。
全程约 15 分钟，重启 2 次；**操作前备份 EFI 分区**。

## 0. 前置检查

```bash
sw_vers -productVersion                 # 记录版本
ioreg -l -p IODeviceTree | grep -i 14c3 # 记录 Wi-Fi 卡 PCI 路径中的 root port（如 PCI0RP04）
ls /System/Library/DriverExtensions/ | grep -i sunrise   # 应看到 AppleSunriseWLAN.dext
```

记下你的 Wi-Fi 卡挂在哪个 root port（`ioreg` 输出 / Hackintool PCIe 列表 / `lspci` 均可）。
本包 SSDT 默认 **RP04**，其他端口需要重新编译（见 `ssdt/README.md`），或改用第 3 步的 DeviceProperties 法。

## 1. 取 `AppleSunrise.kext`（本仓库已内置两版，从官方 KDK 原样提取）

- 系统 **26.3 – 26.6** → 用 `kexts/macos26/AppleSunrise-26.3.kext`（KDK 26.3/25D5087f，原帖同款）
- 系统 **26.7.x+** → 用 `kexts/macos26/AppleSunrise.kext`（KDK 26.7.1/25G241）
- 系统 **26.0 – 26.2** → 自行下载对应 build 的 KDK 提取（方法见 `kexts/macos26/README.md`）

拷入 `EFI/OC/Kexts/`，config.plist 按 `patches/Kernel-Add-AppleSunrise.xml` 增加 `Kernel → Add` 条目。

## 2. 设备伪装（二选一）

**A. SSDT（推荐，原帖方案）**：`ssdt/SSDT-ARPT_RP04_Mediatek.aml` 拷入 `EFI/OC/ACPI/`，
config.plist `ACPI → Add` 启用（详见 `ssdt/README.md`）。

**B. DeviceProperties（免编译）**：按 `patches/DeviceProperties-Add.xml` 在
`DeviceProperties → Add` 中加入你的网卡路径与 `device-id`（小端 base64 已备好）。

## 3. 写入固件（Recovery 模式）

```bash
csrutil disable
csrutil authenticated-root disable
mkdir ~/mount && sudo mount -o nobrowse /dev/diskXsY ~/mount   # 主系统卷，非 data 卷
D=~/mount/System/Library/DriverExtensions/com.apple.AppleSunriseWLAN.dext
sudo cp firmware/WIFI_MT7922_patch_mcu_1_1_hdr.bin $D/
sudo cp firmware/WIFI_RAM_CODE_MT7922_1.bin $D/W7922_1.bin     # 必须改名
sudo chown root:wheel $D/*.bin && sudo chmod 644 $D/*.bin
sudo bless --folder ~/mount/System/Library/CoreServices --bootefi --create-snapshot
reboot
```

`/dev/diskXsY` 用磁盘工具确认（主卷，其下有快照子树）；`wifi.cfg` 为 Apple 私有文件，
本包不提供，缺失时驱动刷屏 `wlanCfgSet` 报错但**不致命**。

## 4. 启动参数

按 `patches/boot-args.txt` 加入 NVRAM boot-args：

```
wlan.debug.enable=0x1        # 解锁驱动详细日志（排错用，稳定后可去掉）
mtk.wlan.startdelay=30       # 进系统前 DEXT 崩溃时加（个别卡需 45）
```

## 5. 蓝牙（可选）

- **蓝牙走 PCIe（14c3:792a）**：26.x 原生驱动直接认，无需本包固件。
- **蓝牙走 USB（0E8D:xxxx）**：`sudo mkdir -p /etc/bluetool && sudo cp firmware/BT_RAM_CODE_MT7922_1_1_hdr.bin /etc/bluetool/`；
  或在 EFI 注入改装 BlueToolFixup（原帖 post 2433923 附件，见 `research/notes/docs/03-bluetooth.md`）。

## 验证

```bash
kmutil showloaded --list-only | grep -i sunrise          # DEXT 已加载
networksetup -listallhardwareports | grep -A2 -i wi-fi   # Wi-Fi 接口出现
log show --style compact --debug --last boot --info --predicate 'drv.' | tail -50
```

日志出现 `request_firmware ... Failed to find firmware file` → 固件没写对位置/名字；
`wlanRemove` + `Disabling PCI device` → 驱动放弃设备（缺 wifi.cfg 或伪装不匹配）；
判读全表见 `research/notes/docs/02-debug-method.md`。

## 回滚

- 改回 RELEASE 版 OpenCore、删 boot-arg、禁用 SSDT/Kernel-Add 条目即可恢复原状；
- 系统卷改动会随启用 SIP + authenticated-root 重启自动清理，无需手动还原。

## 路线 B：macOS 15 Sequoia（第三方 kext，不碰系统卷）

Sequoia 没有原生 Sunrise 驱动，用 `kexts/macos15/`：

1. 将 `Lilu.kext`、`SimpleMtkBtFw.kext`、`SimpleMtkBtPatcher.kext`、`SimpleMtkWlan.kext` 拷入 `EFI/OC/Kexts/`，`Kernel → Add` **按 Lilu → BtFw → BtPatcher → Wlan 顺序**启用；
2. 解压 `kexts/macos15/SimpleMTK-1.0.0.zip`，把 `SimpleMTK.app` 拖入 `/Applications/`（**预编译成品，无需编译**）——扫描/连网靠它；
3. 蓝牙按 `kexts/macos15/README.md` 处理两个致命坑（产品名勿改 + 两条 NVRAM data）；
4. 排错用 `kexts/macos15/mtklog.sh`。

限制：以太网式接口、无 AirDrop/Handoff/Continuity、无 802.11ax。详见 [kexts/macos15/README.md](kexts/macos15/README.md)。

## 不想写系统卷？替代方案

按推荐顺序：

| 方案 | 做法 | 代价 |
|---|---|---|
| **换 MT7932（RZ717）卡** | 26.7.1 已内置其 Wi-Fi 固件，插卡即用 | 换硬件 |
| **第三方 kext 路线** | `SimpleMtkWlan` + `SimpleMtkBtFw`（固件打包在 kext 内，只动 EFI） | 无原生 Wi-Fi 菜单、无 AirDrop/Continuity、无 802.11ax |
| **只要蓝牙** | `/etc/bluetool` + `BT_RAM_CODE_MT7922_1_1_hdr.bin`（26.7.1+ 已内置免操作） | — |
| **仍用原生驱动** | 本 INSTALL 第 3 步 | 需一次 bless 快照；本身可回滚，`wifi.cfg` 可不写 |

> 注：曾传言“升级到 26.7.1 就自带全部固件”并不准确——实测 26.7.1(25G241) 只含 MT7932 的 Wi-Fi 固件与 MT7922 的**蓝牙**固件，MT7922 的 **Wi-Fi 固件仍缺失**。
