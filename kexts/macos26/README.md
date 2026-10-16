# kexts/macos26/ — AppleSunrise.kext（macOS 26 原生路线 HAL）

`AppleSunrise.kext` 是 Apple Sunrise Wi-Fi 驱动的 HAL（硬件抽象层），**只随 KDK 发布、不随系统分发**，26.0–26.6 的原生路线必须由用户自行注入。本文件从官方 KDK 原样提取，未做任何改动。

| 文件 | 来源 | 适配系统 | SHA256（二进制） |
|---|---|---|---|
| `AppleSunrise.kext` | KDK 26.7.1 build 25G241 | **macOS 26.7.x**（LSMinimumSystemVersion = 26.7） | `e9b08b41…b9d31c5c` |

- 架构：**x86_64 + arm64e** 通用二进制（`lipo -archs` 实测）
- Bundle ID：`com.apple.driver.AppleSunrise`
- 来源：<https://developer.apple.com/download/all/>（官方 KDK）；本文件取自 Dortania KDK 镜像 <https://github.com/dortania/KdkSupportPkg/releases/tag/25G241>（Kernel_Debug_Kit_26.7.1_build_25G241.dmg，解包 `KernelDebugKit.pkg` 获得）
- 许可：**Apple 专有**。本仓库仅为黑苹果用户便利提供，官方获取渠道见上；不得用于商业分发。

## 版本匹配规则

kext 的 `LSMinimumSystemVersion` 决定它能加载的最低系统版本：

- 系统 **26.7.x** → 用本目录的 `AppleSunrise.kext`（KDK 25G241）
- 系统 **26.0–26.6** → 必须下载对应自己系统 build 的 KDK（推荐用镜像 <https://github.com/dortania/KdkSupportPkg/releases>，同 build 号），解包后取 `System/Library/Extensions/AppleSunrise.kext`
  （原帖实测用的是 `KDK_26.3_25D5087f`，镜像内有）
- 26.7.1 用户注意：即使你的系统是 26.7.1，本 kext 也只在「系统 DEXT 不认卡」时才需要注入

## 提取方法（自行操作时）

```bash
hdiutil attach -nobrowse Kernel_Debug_Kit_26.x.dmg
pkgutil --expand-full "/Volumes/Kernel Debug Kit/KernelDebugKit.pkg" ~/kdk-pkg
cp -R ~/kdk-pkg/KDK.pkg/Payload/System/Library/Extensions/AppleSunrise.kext .
```

放入 `EFI/OC/Kexts/`，`config.plist → Kernel → Add` 按 `../patches/Kernel-Add-AppleSunrise.xml` 增加条目（ExecutablePath 以 kext 内实际值为准）。
