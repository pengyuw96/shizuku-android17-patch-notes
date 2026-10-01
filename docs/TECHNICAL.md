# Technical notes — Android 17 `ADB_ENABLED` bypass for Shizuku

This document explains **how** the patch was derived and verified. For install / disclaimer / credits, see the [README](../README.md).

---

## 中文

### 1. 现象

在 Pixel 10 Pro（`blazer`）、Android 17、`CP3A.260905.009`（安全补丁 2026-09-05）上：

- 开发者选项中「USB 调试」「无线调试」已开启；
- 官方 Shizuku 13.6.0 以及 thedjchi fork v13.7.0 在点击「启动」时仍提示 USB 调试未启用；
- 无法在**仅手机端**完成无线调试启动（需电脑 adb 的路径不受本补丁目标影响）。

### 2. 调查步骤

1. **对照上游源码**  
   阅读 RikkaApps/Shizuku 与 thedjchi/Shizuku 中无线调试启动路径（`StartWirelessAdbViewHolder` 等），确认启动前会调用 `Settings.Global.getInt(resolver, Settings.Global.ADB_ENABLED, default)`。

2. **确认系统行为（假值，而非默认值）**  
   在实测设备上，持有 `WRITE_SECURE_SETTINGS` 的第三方应用读取 `adb_enabled` 仍得到 `0`。  
   将 `getInt` 的**默认值参数**从 `0` 改为 `1` **无效**——说明系统不是「读失败回退默认值」，而是对第三方调用**注入假 `0`**（与 2026-05 起围绕 adbd / 本机 ADB 可见性收紧的方向一致；公开资料中与 **CVE-2026-0073** 等相关讨论一并参考）。

3. **评估 thedjchi fork 覆盖范围**  
   v13.7.0-thedjchi 已处理 Android 17 **配对**侧问题（例如配对守护进程不再仅绑 `127.0.0.1`、`ACCESS_LOCAL_NETWORK` 等）。其测试环境偏 QPR1 Beta，对正式版上的假值读取未做同等处理。

4. **定位发行包中的检测点**  
   对 `v13.7.0-thedjchi` APK（SHA-256 `6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43`）使用 apktool 2.12.1 反编译，在 smali 中搜索 `ADB_ENABLED` / `adb_enabled` 与 `Settings$Global;->getInt`，锁定启动按钮逻辑所在混淆类。

### 3. 为何改默认值不够

```text
getInt(resolver, "adb_enabled", /* default */ 1)
```

若系统在权限不足时返回默认值，改 `default` 即可绕过。实测返回值**始终为 0**，与 default 无关 → 必须在应用侧**忽略该返回值**，或改用不依赖该 setting 的启动路径。本补丁选择最小侵入：在已确认用户意图（点击启动）且无线调试其余安全边界仍在的前提下，跳过失效检测。

### 4. Smali 改动说明

**包名 / 版本字符串未改**；未新增权限、组件或网络逻辑。

#### 4.1 关键：`smali/rikka/shizuku/L2.smali`

对应源码侧 `StartWirelessAdbViewHolder` 的 Companion `start`（「启动」按钮）。

原逻辑（示意）：

```smali
invoke-static {v0, v4, v2}, Landroid/provider/Settings$Global;->getInt(...)
move-result v0
if-nez v0, :cond_6    # 为 0 → 弹「USB调试未启用」并中止
```

改为：

```smali
invoke-static {v0, v4, v2}, Landroid/provider/Settings$Global;->getInt(...)
const/4 v0, 0x1       # 无条件视为已开启
if-nez v0, :cond_6
```

仍调用 `getInt`（便于对照 / 减少无关 diff），但**丢弃**其返回值。`if-nez` 对常量 `1` 永不跳转错误分支。

同文件内另有一处将 `getInt` 默认参数 `0→1` 的探索改动；在关键改动生效后属冗余，无额外副作用。

#### 4.2 `smali/rikka/shizuku/i2$1.smali`

对应 `AdbStarter.stopTcp` 路径上的同类检测默认值调整（`0→1`），影响 TCP 停止相关分支，非启动主路径。

#### 4.3 `smali/rikka/shizuku/S1.smali`

对应 `SettingsHelper.launchOrHighlightWirelessDebugging`，默认值 `0→1`，主要影响教程 / 高亮跳转，非启动主路径。

### 5. 构建与验证

| 步骤 | 工具 / 方法 |
|---|---|
| 解码 / 回编 | apktool 2.12.1 |
| 对齐与签名 | uber-apk-signer 1.3.0（debug 证书，CN=Android Debug） |
| 静态核验 | 对成品 / 装机包再次 apktool，确认 `L2.smali` 含 `const/4 v0, 0x1` 覆盖 `move-result` |
| 动态核验 | 卸载旧签名包 → 安装补丁 APK → 授予所需权限 → 应用内无线调试配对 → 服务以 shell 权限运行；**全程未接电脑** |

**未改变的安全边界：**

- 无线调试配对码 / 密钥协商；
- adbd TLS / 用户授权；
- Shizuku 对客户端应用的单独授权列表。

### 6. 限制与安全考虑

| 项 | 说明 |
|---|---|
| 签名 | Debug 自签名；无法与 Play / 官方发行版覆盖安装；重装会清空应用数据与授权。 |
| 威胁模型 | 补丁假设：**用户已手动打开无线调试**，且接受在本机跳过失效的 setting 探测。它不提升对未配对设备的远程攻击面。 |
| 误用风险 | 若系统未来改变无线调试策略，跳过检测可能导致更难理解的错误；应优先跟进上游。 |
| 合规 | 修改调试相关客户端逻辑仅供研究与自用；勿用于未授权设备。 |
| 维护 | 上游大版本更新需重新定位混淆类名与偏移；本仓库记录的是针对 **v13.7.0-thedjchi** 的定点补丁。 |

### 7. 基础与发布校验

- 基础 APK SHA-256：`6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43`
- 补丁 APK SHA-256：见对应 GitHub Release 说明（勿依赖仓库历史中的本地路径）。

---

## English

### 1. Symptom

On Pixel 10 Pro (`blazer`), Android 17, build `CP3A.260905.009` (security patch 2026-09-05):

- USB / Wireless Debugging are on in Developer options;
- Official Shizuku 13.6.0 and thedjchi v13.7.0 still report USB debugging disabled when tapping Start;
- Phone-only Wireless Debugging start fails (PC-side `adb` workflows are out of scope for this patch).

### 2. Investigation

1. **Upstream source** — Confirmed the Wireless Debugging start path calls `Settings.Global.getInt(..., ADB_ENABLED, default)` before proceeding.
2. **Fake zero, not default fallback** — With `WRITE_SECURE_SETTINGS`, reads still return `0`. Changing the **default argument** of `getInt` from `0` to `1` had **no effect**, proving the platform injects a fake `0` rather than returning the default on failure. This aligns with tightened on-device ADB visibility after mid-2026 security work (discussions around **CVE-2026-0073** and related adbd hardening are context, not a claim that this APK “fixes” that CVE).
3. **Fork coverage** — thedjchi `v13.7.0` fixed Android 17 **pairing** (daemon bind address, `ACCESS_LOCAL_NETWORK`, etc.) but not the fake-zero gate on release builds exercised here.
4. **Binary localization** — apktool 2.12.1 decode of `v13.7.0-thedjchi` (SHA-256 `6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43`); locate `Settings$Global;->getInt` / `adb_enabled` in the Start button smali.

### 3. Why changing the default failed

If the API returned the default on failure, bumping `default` would suffice. Observed returns are always `0` regardless of default → the client must **ignore** the value (or use another start path). This patch takes the smallest change: after the user explicitly taps Start, skip the dead check while leaving pairing / TLS / consent intact.

### 4. Smali rationale

No package rename, no new permissions, components, or network code.

#### 4.1 Critical: `L2.smali` (`StartWirelessAdbViewHolder` start)

Replace `move-result` consumption with `const/4 v0, 0x1` so the subsequent `if-nez` never takes the “debugging disabled” branch. `getInt` remains for a small diff and easier auditing.

A leftover default `0→1` in the same method is redundant once the force-enabled constant is present.

#### 4.2 `i2$1.smali` / `S1.smali`

Same-class default tweaks on `AdbStarter.stopTcp` and `SettingsHelper.launchOrHighlightWirelessDebugging` paths — not the primary Start gate.

### 5. Verification

Rebuild with apktool → sign with uber-apk-signer (debug cert) → re-decode the shipped APK and confirm the `L2.smali` pattern → install on device, grant permissions, complete in-app Wireless Debugging pairing, confirm `shizuku_server` runs with shell uid **without a PC attached**.

**Unchanged:** pairing ceremony, adbd TLS / user auth, per-client Shizuku grants.

### 6. Limitations & security

| Topic | Note |
|---|---|
| Signing | Debug self-signed; cannot update-install over official builds; reinstall clears data. |
| Threat model | Assumes the user already enabled Wireless Debugging and accepts skipping a broken setting probe. Does not expand unauthenticated remote attack surface. |
| Maintenance | R8 names / offsets are tied to **v13.7.0-thedjchi**; newer upstream builds need re-discovery. |
| Prefer upstream | Switch back when an official fix lands. |

### 7. Checksums

- Base APK SHA-256: `6ea6dee65d5ddc626b6b75b2c2f67f8cc547fa47d7b437e6892639c37eaffe43`
- Patched APK SHA-256: published on the matching GitHub Release (not as local workspace paths).
