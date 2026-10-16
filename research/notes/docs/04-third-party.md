# 04 · 第三方 kext 路线（Sequoia / 老模块兜底）

与"Apple 原生 Sunrise"并行的另一条路：社区自研 kext。适用场景——系统是 Sequoia（无 Sunrise），或模块蓝牙走 USB。

## 仓库分工（认准 fork，别clone错）

| 项目 | 管什么 | Release | 备注 |
|---|---|---|---|
| ★ `flxxxxddd/SimpleMtkWlan`（fork，分支 `mt7921-fixes`） | Wi-Fi（PCIe） | ✅ v1.1.0 / v1.2.0 | **第三方 Wi-Fi 就用它**；修好了 MT7921/RZ608 attach 后过不了流量的问题，补上 802.11n/ac |
| `laobamac/SimpleMtkWlan`（上游） | Wi-Fi | ❌ 无 | bring-up 中，必须自编译 |
| `laobamac/SimpleMtkBtFw`（fork: `flxxxxddd/SimpleMtkBtFw`） | 蓝牙固件加载 | ✅ 1.0.0 | 只负责把固件塞进控制器，之后交给 macOS 蓝牙栈 |
| `laobamac/SimpleMTKClient` | Wi-Fi 菜单栏客户端 | ❌ 无（需 Xcode 编） | **必需**——驱动本身没有原生 Wi-Fi 菜单 |

注意：fork 的 issue 是关的，反馈问题回上游；fork 默认分支是 `mt7921-fixes` 不是 `main`。

## Wi-Fi 安装要点

1. 取 release 的 `SimpleMtkWlan-x.y.z-RELEASE.zip`，kext 放 `EFI/OC/Kexts/`，`config.plist → Kernel → Add` 加一条（Arch=x86_64）。
2. ★ **先确认卡没被 SSDT 藏掉**：很多预制 EFI 带 `SSDT-Disable_Network_*.aml` 把网卡 `vendor-id/device-id` 改成 `0xFFFF`，不禁用它驱动永远匹配不上：
   ```bash
   ioreg -l | grep -A40 'pci14c3'   # 应看到 <c3140000>，而不是 ffffffff
   ```
3. 装 SimpleMTK 客户端 → 重启 → 菜单栏加入网络。

## 三条硬限制（决定值不值得用）

1. **以太网式接口（`enX`）而非原生 Wi-Fi**：扫描/连网全靠 SimpleMTK 客户端，没有系统原生 Wi-Fi 菜单。
2. **AirDrop / Handoff / Continuity 不可能**（设计使然，不是 bug）。
3. **802.11ax（HE）不支持**；睡眠/唤醒未测试。

## 实测能力（fork README）

✅ WPA2-PSK、硬件 RX 解密、扫描、802.11n/ac（VHT80）、A-MPDU、开机自动加入、5 GHz 优先、DMA stall 看门狗自愈。
实测吞吐：MT7921K（RZ608）5 GHz VHT80 约 30–50 Mbit/s；MT7922 在 Tahoe 上曾只有 legacy 20 MHz。

## 排错速查

- 驱动带常驻 ring buffer，`./tools/mtklog.sh > mtk.txt` 免 root 可读；周期性 `stats` 行看 `rx_hwdec` vs `rx_swdec`。
- 重启后 "not running"：热重启不给 M.2 卡断电，彻底关机再上电。
- 测速必关 SimpleMTK 菜单（会触发扫描）。
- 开机 ~2s `element modified after free` panic：测试机上由 NVMeFix.kext 引起，别赖错对象。
- 2.4 GHz 无线键鼠卡顿：同频段射频共存，非驱动 bug。
- 装后 `en0` 归属可能变化，注意网络服务顺序。

## 什么时候别用第三方

- 系统是 macOS 26+ 且芯片在 7922/7923/7932 → **先试原生**（见 README TL;DR）。
- 需要 AirDrop/Continuity/802.11ax → 换卡（Broadcom 原生 / Intel + AirportItlwm）。
