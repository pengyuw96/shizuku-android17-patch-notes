# Shizuku Android 17 适配说明

Google 系统更新后，第三方应用读取「开发者选项 / USB 调试」相关设置（`Settings.Global.ADB_ENABLED`）会得到无效结果，导致官方 Shizuku 等应用无法在手机端自行完成无线调试启动。

本仓库记录在此条件下，基于 [thedjchi/Shizuku](https://github.com/thedjchi/Shizuku)（v13.7.0）构建的一份个人适配 fork 说明（Apache-2.0，沿用上游许可证，见 [LICENSE](LICENSE)）。

## 测试条件

| 项目 | 值 |
|---|---|
| 设备 | Pixel 10 Pro（`blazer`） |
| 系统 | Android 17 |
| Build | `CP3A.260905.009` |
| 安全补丁级别 | 2026-09-05 |

以上为实测环境；其他机型 / 版本未验证。

## 背景

- 上游官方 Shizuku（RikkaApps）在部分新系统上无法正确感知调试开关状态。
- [thedjchi/Shizuku](https://github.com/thedjchi/Shizuku) 已处理 Android 17 的无线调试配对等问题，但仍可能受上述设置读取限制影响。
- 本适配在 fork 基础上跳过已失效的本地检测，**不改变**无线调试配对、TLS 认证与用户授权流程。

## 改动摘要

基础包：thedjchi/Shizuku **v13.7.0-thedjchi**  
APK SHA256：`6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43`

对 apktool 反编译产物做少量 smali 调整（主要在启动无线调试前的 `ADB_ENABLED` 检测逻辑），重新以自签名 debug 证书打包，仅供本人设备使用。本仓库**不提供** APK 下载。

## 致谢

- [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)
- [thedjchi/Shizuku](https://github.com/thedjchi/Shizuku)

## 许可

与上游一致，采用 [Apache License 2.0](LICENSE)。
