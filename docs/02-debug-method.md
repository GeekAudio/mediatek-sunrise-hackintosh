# 02 · 调试五步法（2026-01 实测完整版）

> 这是全帖最有价值的资产：**不猜，让 Apple 自己的驱动把"它想要什么、卡在哪一步"全写进日志**，再按图索骥。

## Step 1 · 让 DEXT 加载

1. 安装 M.2 MT7922 模块（或 M.2→PCIe 转接卡）。
2. 从对应版本 KDK 取 `AppleSunrise.kext`（实测：`KDK_26.3_25D5087f`），OpenCore `Kernel → Add` 注入 → Tahoe 可启动。
3. **device-id 伪装**：DEXT 只匹配 `0x7922`，而 RZ616 原生是 `0x616`：
   - 用帖内附件 `SSDT-ARPT_RP04_Mediatek.aml` 把 device-id 伪造成 `0x7922`（其他 root port 改其中 4 行即可）；**MT7925 伪造为 `22790000`**。
   - 同时给 PXSX 补 **DEXT 相关属性**，否则 Info.plist 阶段就不匹配（Elias64Fr 指出的关键缺失项）。
   - 注：原生 `0x616` 即使伪装成 `0x7922` 也展不开设备树（反汇编显示 0x616 被同等配置，实际行为不同）。

## Step 2 · 打开详细日志

NVRAM 加 boot-arg `wlan.debug.enable=0x1`。仅注入 kext 时日志不会自动出现（CaseySJ 实测：先加参数时日志为空）。

## Step 3 · 收集日志

```bash
log show --style compact --debug --last boot --predicate 'drv.'
# info 级消息要加 --info；纯 drv. 谓词可能为空，换 --info 再试
```

## Step 4 · 读日志定位卡点 → 补缺失件

驱动请求的文件（`drv.I` 前缀即日志已解锁）：

- `wifi.cfg`
- `WIFI_MT7922_patch_mcu_1_1_hdr.bin`
- 另需 `WIFI_RAM_CODE_MT7922_1.bin`，**按驱动请求改名为 `W7922_1.bin`**

向系统卷写入（Recovery 模式，Elias64Fr 实测可行）：

```bash
csrutil disable
csrutil authenticated-root disable
mkdir ~/mount && sudo mount -o nobrowse /dev/diskXsY ~/mount   # 主 Tahoe 卷，非 data 卷
# 拷入固件，权限 root:wheel、644：
#   WIFI_MT7922_patch_mcu_1_1_hdr.bin
#   W7922_1.bin（即 WIFI_RAM_CODE_MT7922_1.bin 改名）
#   wifi.cfg（可选，非默认参数时才需要）
sudo bless --folder ~/mount/System/Library/CoreServices --bootefi --create-snapshot
```

安全性：macOS 本就工作在快照上；创建新快照后即从新快照启动；忘记改动时用 SIP + authenticated-root 启动会自动清理。

## Step 5 · 排除法兜底

- 0x7922 伪装会让 DEXT 每 ~10s 重试、崩溃循环并拖垮系统：
  - `mtk.wlan.startdelay=30`（Fenvi AX5400 实测可进系统）；华硕 PCE-AXE59BT 需 `=45`
  - `mtk.wlan.hangonstart=1`（锁死驱动保证进系统）
- 与成功样本对照：f10（macOS 26.3 beta + KDK_26.3 kext）是唯一见到**完整 AWDL 支持**的配置，需逐一对照差异（机型、RestrictEvents.kext、HDMI 属性、USB 映射方式）。

## 日志判读表

| 日志行 | 含义 | 对策 |
|---|---|---|
| `DK: AppleSunriseWLAN::start ok` | DEXT 已 attach，HAL 注入成功 | — |
| `Failed to open .../wifi.cfg` | 缺配置文件 | Dext 内补入 |
| `Failed to open .../WIFI_MT7922_patch_mcu_1_1_hdr.bin` | 缺 MCU 补丁固件 | 系统卷快照注入 |
| `request_firmware: Failed to find firmware file` | 同上（另一代码路径） | 同上 |
| `wlanCfgSet:(INIT ERROR) wifi config there is no empty entry`（刷屏，键如 `LlwLatencyDbg`/`NanWarmUpTime`/`AwdlGenCap`） | wifi.cfg 缺失，配置表仅 256 项 | Elias64Fr：**不致命**，但放入固件后会出现更严重错误 |
| `wlanDownloadPatch:(INIT WARN) FW[2] load error!` | 固件加载失败 | 补固件 |
| `wlanRemove` + `Disabling PCI device` | 驱动放弃并移除设备 | 调试失败的终点信号 |
| `IOUserServer(com.apple.AppleSunriseWLAN) has crashed too many times` panic | DEXT 崩溃循环（Sn0wMan 0x7922 卡实录） | startdelay / hangonstart 缓解 |
| `corecapture: Missing ivars in CCLogPipe` | CoreCapture 日志管道创建问题 | 观察项，非阻塞错误 |
| `wifiFirmwareLoader: Waiting for firmware to load`（再无后续） | 固件加载器卡死（BCMWLAN dext 上有完整流程可对比） | 佐证固件缺失是根因 |

## 2026-01 调试结论

1. **根因**：26.0–26.2 的 DEXT 里 Wi-Fi 固件和 `wifi.cfg` 全部缺失，驱动在启动早期即失败。
2. **只补固件不够**：缺 wifi.cfg 先刷屏 `wlanCfgSet`；补了固件后驱动仍以 `wlanRemove` 收场。
3. 原生 0x616 卡片（RZ616）即使伪装也展不开设备树。
4. 最终策略：等 Apple 在后续 Tahoe 更新中随附固件（蓝牙固件已随 26.2 的 dext 先行落地，Wi-Fi 固件顺理成章）。
