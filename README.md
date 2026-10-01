# Shizuku 补丁版 APK 声明与工作记录

**日期**：2026-10-01
**设备**：Pixel 10 Pro（blazer），Android 17（SDK 37）
**问题**：系统更新后，Shizuku 等应用检测不到"开发者选项/USB调试"开关，导致无法通过无线调试在手机端自行启动。

---

## 一、问题根因（调查结论）

1. 2026-05 Android 安全补丁修复了 adbd 认证绕过漏洞 **CVE-2026-0073**，Google 借此收紧了设备本机 ADB 权限。
2. 实测确认（本机，Android 17 正式版）：第三方应用读取 `Settings.Global.ADB_ENABLED`（"adb_enabled"）时，**系统强制返回假值 0**，即使应用已持有 `WRITE_SECURE_SETTINGS` 权限；修改"读取失败时的默认值"无效（说明不是返回默认值，而是注入假 0）。
3. 同类限制导致官方 Shizuku（13.6.0，2025-05 后停更）的"无线调试启动"流程永远弹"USB调试未启用"。
4. 社区方案：thedjchi/Shizuku fork v13.7.0 已修复 Android 17 的**配对**问题（配对守护进程不再绑定 127.0.0.1、新增 ACCESS_LOCAL_NETWORK 运行时权限申请），但**未覆盖上述读取假 0 问题**（其测试环境为 QPR1 Beta，限制更宽松）。

## 二、我所做的代码修改（全部改动清单）

基础包：`thedjchi/Shizuku` 官方 Release **v13.7.0-thedjchi**（APK SHA256：
`6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43`）。

对反编译产物（apktool 2.12.1）做了 **3 处 smali 修改**，其中第 3 处是使检测绕过生效的关键改动。除此之外**没有**新增、删除或修改任何其他代码、资源、权限、组件：

### 改动 1：`smali/rikka/shizuku/L2.smali`（R8 混淆名，对应源码 `StartWirelessAdbViewHolder$Companion.start`，即"启动"按钮逻辑）★关键改动

原代码（约 1233 行）：
```smali
invoke-static {v0, v4, v2}, Landroid/provider/Settings$Global;->getInt(Landroid/content/ContentResolver;Ljava/lang/String;I)I
move-result v0            # 取读取结果（Android 17 下被系统注入假 0）
if-nez v0, :cond_6        # 为 0 则弹"USB调试未启用"对话框并中止
```
改为：
```smali
invoke-static {v0, v4, v2}, Landroid/provider/Settings$Global;->getInt(Landroid/content/ContentResolver;Ljava/lang/String;I)I
const/4 v0, 0x1           # 无条件视为已开启，跳过失效的检测
if-nez v0, :cond_6
```

### 改动 2：`smali/rikka/shizuku/L2.smali`（同文件约 1128 行，同一方法内）
`const/4 v2, 0x0` → `const/4 v2, 0x1`（getInt 的默认值参数，0 改 1；此改动被改动 3 覆盖，属探索过程遗留，无副作用）。

### 改动 3（探索过程，已包含在成品中）：
- `smali/rikka/shizuku/i2.1.smali`（对应 `AdbStarter.stopTcp`）约 799 行：`const/4 v1, 0x0` → `const/4 v1, 0x1`（仅影响 TCP 停止模式的同类检测）。
- `smali/rikka/shizuku/S1.smali`（对应 `SettingsHelper.launchOrHighlightWirelessDebugging`）约 81 行：`const/4 v2, 0x0` → `const/4 v2, 0x1`（仅影响教程页跳转行为）。

**声明**：以上改动仅使 Shizuku 在 Android 17 上跳过自身已失效的"开发者选项检测"，不绕过任何系统安全机制（无线调试的配对、TLS 认证、用户授权流程全部原样保留），不含任何网络后门、数据收集或额外权限。APK 使用 uber-apk-signer 生成的**自签名 debug 证书**重新签名（证书 CN=Android Debug，SHA256 `1e08a903aef9c3a721510b64ec764d01d3d094eb954161b62544ea8f187b5953`，有效期至 2044-03-11），仅供本人设备使用。Shizuku 原项目为 Apache-2.0 许可，允许修改。

## 三、手机上的变更

1. 卸载官方 Shizuku 13.6.0，安装上述补丁版（包名不变 `moe.shizuku.privileged.api`，版本号 13.7.0-thedjchi）。
2. `pm grant` 授予该应用：`android.permission.WRITE_SECURE_SETTINGS`、`android.permission.POST_NOTIFICATIONS`（二者为 fork 功能所需，manifest 中原本声明）。
3. 通过补丁版自身的配对流程完成无线调试配对（应用私钥存储于应用私有目录），服务 `shizuku_server` 现以 shell 权限运行——**全程未连接电脑**。
4. 注意：重装清空了原官方版数据，之前授权的 4 个应用需在 Shizuku 中重新授权。

## 四、工作产物说明（本地保留，不上传仓库）

本仓库仅发布声明与改动说明；APK、反编译目录、工具链与过程截图留在本机，不纳入 Git。

| 类别 | 说明 |
|---|---|
| 补丁成品 APK | 基于 thedjchi/Shizuku v13.7.0，apktool 改 smali 后自签名（debug 证书） |
| 基础包 | thedjchi fork v13.7.0，SHA256 `6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43` |
| 补丁位置 | 反编译后 `smali/rikka/shizuku/L2.smali`、`i2.1.smali`、`S1.smali`（见第二节） |
| 工具 | apktool 2.12.1、uber-apk-signer 1.3.0、OpenJDK 17 |

**不上传**：成品/中间 APK、装机核验包、`fork_decoded` / `verify_*`、上游源码克隆、JDK/工具 jar、过程截图与 API 缓存。

## 五、后续使用说明

- 手机重启后 Shizuku 服务失效：打开 Shizuku 点"启动"即可（配对关系已持久保存，无需电脑）。若提示无线调试未开，先到开发者选项打开"无线调试"。
- fork 自带"开机自启 + 看门狗"（设置内开启后，启动成功会自动关闭无线调试以省电，属正常行为）。
- 若日后重新刷机或卸载重装，需重新授予两项权限并重新配对；APK 请使用 `out/` 中的成品，勿混用官方版/fork 原版（签名不同）。

---

## 许可与致谢

- 上游 Shizuku（RikkaApps）与 fork（thedjchi/Shizuku）均为 **Apache-2.0**。
- 本文档描述的修改仅供个人设备适配记录；重新分发修改后的 APK 时请遵守原许可并保留版权声明。
- 引用： [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)、[thedjchi/Shizuku](https://github.com/thedjchi/Shizuku)。
