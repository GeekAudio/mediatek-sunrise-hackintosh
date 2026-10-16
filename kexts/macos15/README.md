# kexts/macos15/ — macOS 15 Sequoia 路线（第三方 kext）

Sequoia 没有原生 Sunrise 驱动，此目录提供经过验证的第三方 kext，**固件打包在 kext 内，只动 EFI 分区、完全不碰系统卷**。

| 文件 | 版本 | 来源 | 许可 |
|---|---|---|---|
| `Lilu.kext` | 1.7.2 | [acidanthera/Lilu](https://github.com/acidanthera/Lilu/releases/tag/1.7.2) | BSD |
| `SimpleMtkBtFw.kext` | 1.0.0 | [laobamac/SimpleMtkBtFw](https://github.com/laobamac/SimpleMtkBtFw/releases/tag/1.0.0) | 见 `LICENSE.SimpleMtkBtFw.txt` |
| `SimpleMtkBtPatcher.kext` | 1.0.0 | 同上（依赖 Lilu） | 同上 |
| `SimpleMtkWlan.kext` | 1.2.0 | [flxxxxddd/SimpleMtkWlan](https://github.com/flxxxxddd/SimpleMtkWlan/releases/tag/v1.2.0)（fork，分支 `mt7921-fixes`） | GPLv2 |
| `mtklog.sh` | v1.2.0 | 同上（免 root 读驱动日志） | 同上 |
| `SHA256SUMS.txt` | v1.2.0 | 官方 release 校验和 | — |
| `SimpleMTK-1.0.0.zip` | 1.0.0 | **菜单栏客户端预编译版**（上游无 release，本仓库从 laobamac/SimpleMTKClient 源码编译，内嵌 Sparkle 2.10.0） | — |

四个 kext 二进制校验值（SHA256，取自行内文件）：

```
SimpleMtkWlan:       2737a186ced93b7609a195b8e7cfe7ae03530474261610482907e0b8f4115802
SimpleMtkBtFw:       8079337d3bd2ef9e2d886ca8987272a4823ddb9bfc80502e28f4d075d0fd660b
SimpleMtkBtPatcher:  e481fe38552024c7b95b2cf35d8bf498a0bfa32ddcb3264a8c12239afee9e8b2
Lilu:               ff98508b40a1eb5fb477029db8886412520f8e39965279ea331c99e3d2e3b274
SimpleMtkWlan-1.2.0-RELEASE.zip: bb3a850c...76b775a（官方 SHA256SUMS.txt）
```

## 安装

1. 四个 kext 拷入 `EFI/OC/Kexts/`，`config.plist → Kernel → Add` **按此顺序**（Arch=x86_64）：

   ```
   [1] Lilu.kext               ← 宿主，必须最前
   [2] SimpleMtkBtFw.kext      ← 蓝牙固件加载
   [3] SimpleMtkBtPatcher.kext ← 蓝牙兼容性补丁（依赖 Lilu）
   [4] SimpleMtkWlan.kext      ← Wi-Fi
   ```

2. **装菜单栏客户端**：解压 `SimpleMTK-1.0.0.zip` 得到 `SimpleMTK.app`，拖入 `/Applications/` 即可——**无需编译**（上游仓库无 release，此 zip 由本仓库从源码构建，x86_64 / macOS 10.13+，ad-hoc 签名）。扫描/连网全靠它。
3. 只走 OpenCore 注入；**不要**同时注入其他 USB 蓝牙固件加载器（IntelBluetoothFirmware / BrcmPatchRAM 等会抢设备）。

## 两个致命坑（蓝牙）

1. **不要把 USB 产品名改成 `Bluetooth USB Host Controller`**——会让 `bluetoothd` 走“内置控制器”路径并按 Broadcom 解析，直接 abort。保持产品名原样。
2. **NVRAM 必须同时 Add + Delete 两条 data**（否则 `bluetoothd` 每 ~30s 被看门狗杀掉、蓝牙菜单转圈）：

   ```
   7C436110-AB2A-4BBB-A880-FE41995C9F82
     bluetoothExternalDongleFailed    = 00
     bluetoothInternalControllerInfo  = <14 字节控制器信息>
   ```

3. AirPods 连上无声：A2DP Read Clock 问题，见 `research/notes/docs/03-bluetooth.md`。

## 硬限制（决定值不值得用）

- **以太网式接口（`enX`）**，没有 macOS 原生 Wi-Fi 菜单，扫描/连网靠 SimpleMTK。
- **AirDrop / Handoff / Continuity 不可能**（设计使然，不是 bug）。
- **802.11ax（HE / Wi-Fi 6）不支持**；睡眠/唤醒未测试。

## 排错

- `./mtklog.sh > mtk.txt`：免 root 读驱动常驻日志；看 `rx_hwdec` vs `rx_swdec`。
- 重启后 "not running"：热重启不给 M.2 卡断电，彻底关机再上电。
- 测速时关掉 SimpleMTK 菜单（会触发扫描）。
- 开机 ~2s `element modified after free` panic：测试机上由 NVMeFix.kext 引起，别赖错对象。
- Wi-Fi kext 加载但无接口：卡可能被 `SSDT-Disable_Network` 藏了（`ioreg -l | grep -A40 'pci14c3'`）。
- fork 的 issue 是关闭的，反馈问题回上游 `laobamac/SimpleMtkWlan`。
