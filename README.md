# Shizuku Android 17 Patch

> Personal fork notes and a debug-signed APK that restores **in-app Wireless Debugging start** on Android 17, where `Settings.Global.ADB_ENABLED` is no longer readable by third-party apps.

**Base:** [thedjchi/Shizuku](https://github.com/thedjchi/Shizuku) `v13.7.0-thedjchi`  
**License:** [Apache License 2.0](LICENSE) (same as upstream)  
**Release:** see [Releases](https://github.com/pengyuw96/shizuku-android17-patch-notes/releases) for the patched APK and SHA-256.

---

## 中文

### 问题

Google 在近期安全更新中收紧了设备本机对 ADB 相关设置的可见性。第三方应用（含官方 Shizuku）读取 `Settings.Global.ADB_ENABLED`（`adb_enabled`）时，系统会**强制返回假值 `0`**，即使应用已持有 `WRITE_SECURE_SETTINGS`。

结果是：开发者选项里「USB 调试 / 无线调试」实际已开启，Shizuku 仍提示「USB 调试未启用」，无法在手机端自行完成无线调试启动。

[thedjchi/Shizuku](https://github.com/thedjchi/Shizuku) `v13.7.0` 已修复 Android 17 上的**配对**问题（配对守护进程绑定、本地网络权限等），但**未覆盖**上述假值读取问题。

### 解决思路

在 apktool 反编译产物中，对启动无线调试前的 `ADB_ENABLED` 检测做最小 smali 修改：跳过已失效的本机检测，**不改动**无线调试配对、TLS 认证与用户授权流程。

关键改动位于混淆类 `L2.smali`（对应 `StartWirelessAdbViewHolder` 的启动逻辑）：在 `Settings.Global.getInt(...)` 之后无条件将结果视为已开启。另有两处同类默认值/检测调整（`i2$1.smali`、`S1.smali`），详见 [docs/TECHNICAL.md](docs/TECHNICAL.md)。

### 测试矩阵

| 项目 | 值 |
|---|---|
| 设备 | Pixel 10 Pro（`blazer`） |
| 系统 | Android 17（SDK 37） |
| Build | `CP3A.260905.009` |
| 安全补丁级别 | 2026-09-05 |

其他机型 / 版本**未验证**。

### 构建说明

| 项目 | 值 |
|---|---|
| 基础 APK | thedjchi/Shizuku **v13.7.0-thedjchi** |
| 基础 SHA-256 | `6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43` |
| 工具 | apktool 2.12.1、uber-apk-signer 1.3.0 |
| 签名 | 自签名 **debug** 证书（CN=Android Debug） |
| 包名 | `moe.shizuku.privileged.api`（与上游一致） |

补丁版 APK 的校验和见对应 [Release](https://github.com/pengyuw96/shizuku-android17-patch-notes/releases) 说明。

### 使用提示

1. 卸载设备上已有的官方 / 其他签名的 Shizuku（签名不同，无法覆盖安装）。
2. 安装本 Release 中的 debug-signed APK。
3. 按需授予 `WRITE_SECURE_SETTINGS`、`POST_NOTIFICATIONS`（fork 功能所需，manifest 中原本声明）。
4. 在开发者选项中打开「无线调试」，于应用内完成配对并启动。
5. 重启后服务会失效：打开应用点「启动」即可（配对关系通常已持久保存）。

### 免责声明

- 本仓库为**个人适配记录与发布**，非 RikkaApps 或 thedjchi 官方项目。
- APK 使用 **debug 证书**签名，仅供自行研究与自用设备安装；请自行评估风险。
- 补丁**不绕过**系统无线调试的配对、TLS 与用户授权；不含额外网络后门、埋点或未声明权限。
- 修改系统调试相关行为可能违反设备保修条款或本地法规；使用后果自负。
- 上游若发布正式修复，请优先改用官方 / 维护中的发行版。

### 致谢

- [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku) — 原项目
- [thedjchi/Shizuku](https://github.com/thedjchi/Shizuku) — Android 17 配对等相关修复的 fork 基础

### 许可

[Apache License 2.0](LICENSE)

---

## English

### Problem

After recent Android security updates, third-party apps can no longer reliably read developer / ADB switches. Reads of `Settings.Global.ADB_ENABLED` (`adb_enabled`) return a **forced fake `0`**, even when the app holds `WRITE_SECURE_SETTINGS`.

Official Shizuku therefore always believes USB debugging is off and cannot start via Wireless Debugging from the device UI.

[thedjchi/Shizuku](https://github.com/thedjchi/Shizuku) `v13.7.0` fixed Android 17 **pairing** issues (pairing daemon bind address, local-network permission, etc.) but **does not** address this fake-zero read.

### Approach

Minimal smali edits on an apktool decode of thedjchi’s release: skip the broken local `ADB_ENABLED` gate before Wireless Debugging start. **Wireless pairing, TLS authentication, and user consent are unchanged.**

The critical change is in obfuscated `L2.smali` (`StartWirelessAdbViewHolder` start path): after `Settings.Global.getInt(...)`, force the result to “enabled”. Related tweaks exist in `i2$1.smali` and `S1.smali`. Full write-up: [docs/TECHNICAL.md](docs/TECHNICAL.md).

### Test matrix

| Item | Value |
|---|---|
| Device | Pixel 10 Pro (`blazer`) |
| OS | Android 17 (SDK 37) |
| Build | `CP3A.260905.009` |
| Security patch | 2026-09-05 |

Other devices / builds are **unverified**.

### Build notes

| Item | Value |
|---|---|
| Base APK | thedjchi/Shizuku **v13.7.0-thedjchi** |
| Base SHA-256 | `6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43` |
| Tooling | apktool 2.12.1, uber-apk-signer 1.3.0 |
| Signing | Self-signed **debug** certificate (CN=Android Debug) |
| Package | `moe.shizuku.privileged.api` (unchanged) |

Checksums for the patched APK are listed on each [Release](https://github.com/pengyuw96/shizuku-android17-patch-notes/releases).

### Disclaimer

- Personal fork documentation and binary — **not** an official RikkaApps or thedjchi release.
- Debug-signed APK for research / personal devices only; use at your own risk.
- Does **not** bypass Wireless Debugging pairing, TLS, or user authorization; adds no undeclared network backdoors or permissions.
- Prefer upstream when an official fix ships.

### Credits

- [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)
- [thedjchi/Shizuku](https://github.com/thedjchi/Shizuku)

### License

[Apache License 2.0](LICENSE)
