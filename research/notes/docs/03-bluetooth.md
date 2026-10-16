# 03 · 蓝牙路线（比 Wi-Fi 更早成功）

MediaTek 蓝牙是整条故事线的**开篇**：2025-12-07，Elias64Fr 逆向 `bluetoothd`（Tahoe 26.0 x86-64，Ghidra）时发现了 MediaTek 设备支持列表。

## A. 原生路线：`/etc/bluetool` 固件法

macOS 的 `bluetoothd` 原生认以下 USB 蓝牙设备（Tahoe 26.0–26.2 共 7 个）：

| 厂商 | idVendor : idProduct | 芯片 |
|---|---|---|
| Foxconn ×5 | `0489:E0D8` / `0489:E0E0` / `0489:E0F1` / `0489:E0F2` / `0489:E0F6` | MT7921/MT7922 |
| MediaTek | `0E8D:7922` | MT7922 |
| Azurewave | `13D3:3568` | MT7922 |

（Sequoia 15.7.2 只认 Foxconn `0489:E0D8` 一个。）

方法：

```bash
sudo mkdir -p /etc/bluetool
# 从 linux-firmware 取 BT_RAM_CODE_MT7922_1_1_hdr.bin（文件名必须以 BT_RAM_CODE_MT7922 开头）
sudo cp BT_RAM_CODE_MT7922_1_1_hdr.bin /etc/bluetool/
```

`/etc/bluetool` 并非系统默认目录，需手动创建；固件需随时间更新（跟随 linux-firmware 仓库）。

## B. 改装 BlueToolFixup（原帖实际使用的方式）

- Elias64Fr / CaseySJ 的做法：改装 `BlueToolFixup.kext`，把设备列表里第 3 组 VID/PID 改成自己模块的值（该版本同时支持 3 组 VID/PID）。
- CaseySJ 实测：MT7922 模块蓝牙部分 `idVendor=0x0E8D`、`idProduct=0x616`，改装版 BlueToolFixup 注入后**蓝牙直接可用**。
- Sn0wMan 等多人证实；AirPods 可连接。
- ⚠️ 不要与原版 BlueToolFixup 同时注入；不要另加其他抢同一 USB 蓝牙设备的固件加载器。

## C. AirPods 无声修复（CaseySJ → Sn0wMan，2026-01-11）

现象：能连接、无声音。修复步骤：

1. Tahoe 系统设置 → 蓝牙 → AirPods → "i" → **忘记此设备**
2. iPhone/iPad 靠近 AirPods 开盒，按屏幕提示**在 iOS 侧重配**
3. 回 Mac 菜单栏蓝牙下拉重新连接 → 恢复正常

补充：YouTube 在 Firefox 下切换耳机可能短暂无声（Safari/Opera 无此问题）——浏览器侧问题，非驱动。

## D. 第三方路线（USB 蓝牙老模块，非原生）

老模块蓝牙走 USB 且设备 ID 不在原生列表时，用 `laobamac/SimpleMtkBtFw`（有 1.0.0 release）：

1. `Lilu.kext` → `SimpleMtkBtFw.kext` → `SimpleMtkBtFw.kext` 依赖的 `SimpleMtkBtPatcher.kext`，按此顺序注入
2. 只走 OpenCore 注入，勿与其他 BT 固件加载器并存
3. 两个致命坑：① 不要把 USB 产品名改成 `Bluetooth USB Host Controller`（会走"内置控制器"路径，被当作 Broadcom 而 abort）；② 必须在 NVRAM 同时 Add + Delete 两条键（`bluetoothExternalDongleFailed=00`、`bluetoothInternalControllerInfo=<14 字节>`），否则 `bluetoothd` 每 ~30s 被看门狗杀掉、蓝牙菜单一直转圈

详见 [docs/04-third-party.md](04-third-party.md)。

## E. 2026-10 现状

macOS 26.7.1 的 `AppleSunriseBluetooth.dext` 已自带 `BT_RAM_CODE_MT7922_1_1_hdr.bin` / `BT_RAM_CODE_MT7923_1_1_hdr_ipc.bin`——**蓝牙走 PCIe 的新模块已开箱即用**，上面 A/B 两法只对老模块 / 老系统有意义。
