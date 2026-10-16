# Changelog

## [1.0.0] - 2026-10-16

首个发布版。同时覆盖 **macOS 26 原生路线**与 **macOS 15 Sequoia 第三方路线**。

### 文件

- `firmware/`：MT7922 固件三件套（linux-firmware 官方）+ `SHA256SUMS`
- `ssdt/SSDT-ARPT_RP04_Mediatek.aml`：device-id 伪装 SSDT（原帖 Elias64Fr 修正版）
- `patches/`：OpenCore 补丁（Kernel→Add / DeviceProperties / boot-args）
- `kexts/macos26/AppleSunrise.kext`：HAL kext（官方 KDK 26.7.1/25G241 提取，x86_64+arm64e）
- `kexts/macos26/AppleSunrise-26.3.kext`：HAL kext（官方 KDK 26.3/25D5087f 提取，原帖同款，适配 26.3–26.6）
- `kexts/macos15/`：Lilu 1.7.2 + SimpleMtkBtFw 1.0.0 + SimpleMtkWlan 1.2.0（fork 修复版）+ **SimpleMTK-1.0.0.zip 预编译客户端**（上游无 release，本仓库从源码编译，内嵌 Sparkle 2.10.0）
- `INSTALL.md`：macOS 26 原生 + macOS 15 第三方两条安装路径
- `research/`：社区调试记录（不进发布包）

### 实测结论（2026-10-16，macOS 26.7.1/25G241）

- AppleSunriseBluetooth.dext 已含 MT7922 蓝牙固件 → 26.7.1 蓝牙免操作
- AppleSunriseWLAN.dext 仅含 IZUBA(MT7932) Wi-Fi 固件，`*7922*` 文件缺失 → **MT7922 的 Wi-Fi 固件在 26.7.1 仍需本包补齐**（对 MT7932/RZ717 用户则 26.7.1 开箱即用）
- `AppleSunrise.kext` 26.7.1 仍不随系统分发（仅 KDK）
