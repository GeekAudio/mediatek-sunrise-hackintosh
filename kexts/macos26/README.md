# kexts/macos26/ — AppleSunrise.kext（macOS 26 原生路线 HAL）

`AppleSunrise.kext` 是 Apple Sunrise Wi-Fi 驱动的 HAL（硬件抽象层），**只随 KDK 发布、不随系统分发**，26.0–26.6 的原生路线必须由用户自行注入。本文件从官方 KDK 原样提取，未做任何改动。

| 文件 | 来源 | 适配系统 | SHA256（二进制） |
|---|---|---|---|
| `AppleSunrise.kext` | KDK 26.7.1 build 25G241 | **macOS 26.7.x+**（LSMinimumSystemVersion = 26.7） | `e9b08b41…b9d31c5c` |
| `AppleSunrise-26.3.kext` | KDK 26.3 build 25D5087f（原帖实测同款） | **macOS 26.3 – 26.6**（LSMinimumSystemVersion = 26.3） | `d2f5893f…f277b594` |

两个 kext 均为 **x86_64 + arm64e** 通用二进制，Bundle ID `com.apple.driver.AppleSunrise`，均从官方 KDK 原样提取未做改动。
26.0–26.2 用户请自行下载对应 build 的 KDK 提取（26.2 为该驱动首个含此 kext 的 KDK 版本）。

- 架构：**x86_64 + arm64e** 通用二进制（`lipo -archs` 实测）
- Bundle ID：`com.apple.driver.AppleSunrise`
- 来源：<https://developer.apple.com/download/all/>（官方 KDK）；本文件取自 Dortania KDK 镜像 <https://github.com/dortania/KdkSupportPkg/releases/tag/25G241>（Kernel_Debug_Kit_26.7.1_build_25G241.dmg，解包 `KernelDebugKit.pkg` 获得）
- 许可：**Apple 专有**。本仓库仅为黑苹果用户便利提供，官方获取渠道见上；不得用于商业分发。

## 版本匹配规则

kext 的 `LSMinimumSystemVersion` 决定它能加载的最低系统版本：

- 系统 **26.7.x+** → `AppleSunrise.kext`（KDK 25G241，本仓库已备）
- 系统 **26.3 – 26.6** → `AppleSunrise-26.3.kext`（KDK 25D5087f，原帖同款，本仓库已备）
- 系统 **26.0 – 26.2** → 自行下载对应 build 的 KDK 提取（镜像 <https://github.com/dortania/KdkSupportPkg/releases>，选同 build 号）
- 26.7.1 用户注意：即使系统是 26.7.1，也只在「系统 DEXT 不认卡」时才需要注入

## 提取方法（自行操作时）

```bash
hdiutil attach -nobrowse Kernel_Debug_Kit_26.x.dmg
pkgutil --expand-full "/Volumes/Kernel Debug Kit/KernelDebugKit.pkg" ~/kdk-pkg
cp -R ~/kdk-pkg/KDK.pkg/Payload/System/Library/Extensions/AppleSunrise.kext .
```

放入 `EFI/OC/Kexts/`，`config.plist → Kernel → Add` 按 `../patches/Kernel-Add-AppleSunrise.xml` 增加条目（ExecutablePath 以 kext 内实际值为准）。
