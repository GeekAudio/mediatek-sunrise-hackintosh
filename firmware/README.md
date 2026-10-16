# firmware/ — MT7922 固件（linux-firmware 官方）

| 文件 | 大小 | 用途 | 写入位置 |
|---|---|---|---|
| `WIFI_MT7922_patch_mcu_1_1_hdr.bin` | 134 KB | Wi-Fi MCU 补丁 | `AppleSunriseWLAN.dext/` 内 |
| `WIFI_RAM_CODE_MT7922_1.bin` | 1.1 MB | Wi-Fi RAM 固件 | 同上，**改名为 `W7922_1.bin`**（驱动按此名请求） |
| `BT_RAM_CODE_MT7922_1_1_hdr.bin` | 505 KB | 蓝牙固件 | USB 蓝牙放 `/etc/bluetool/` |

校验：`shasum -a 256 -c SHA256SUMS`（在 firmware/ 目录下执行）。

## 来源与许可

- 三个文件均取自官方 linux-firmware 仓库 `mediatek/` 目录：
  <https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek>
- 按 linux-firmware 的再分发条款提供；MediaTek 固件的许可条款见该仓库 `LICENSE.*` 文件。
- 请勿从不可信来源替换这些文件。

## 注意事项

- **`wifi.cfg` 不在本包**：它是 Apple 私有配置（驱动日志 `wlanCfgSet` 报错来源）。
  原帖实测：缺失时刷屏报错但**不致命**；然而仅补固件、缺 wifi.cfg 仍可能导致驱动退出
  （日志末尾 `wlanRemove` + `Disabling PCI device`）。macOS 26.7.1 的 DEXT 内可找到
  MT7932 的 `IZUBA_wifi.cfg` 可作结构参考。
- **macOS 26.7.1+ 无需本目录**：系统 DEXT 已自带固件。
