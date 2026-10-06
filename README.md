# PetalSpeed Fixed · 花瓣测速去高危版

清除 `Android.Virus.Gray.Devicemaster.A.GWBD` 灰码告警的华为「花瓣测速」修改版。

> **核心变化**：移除设备身份采集（ICCID / IMSI / 手机号 / android_id 指纹）与 4 项高危权限，
> 保留测速、运营商识别、信号强度、网络类型等全部主要功能。

---

## 一、项目背景

原版 APK 被主流安全引擎标记为：

```
Android.Virus.Gray.Devicemaster.A.GWBD
```

该标记属于 **灰码（Grayware）** 类别 —— 并非木马或恶意代码，而是引擎对
**「设备身份信息采集行为」** 的归类判定。

经反编译定位，触发源为华为测速 SDK 中的 `com.huawei.hms.petalspeed.mobileinfo`
模块。它会在测速过程中收集 SIM 卡与设备唯一标识，用于运营商识别和结果上报。
对安全引擎而言，这类行为与「设备指纹采集」无异，因此被归入 Gray.Devicemaster。

本仓库提供**行为中性化（neutralized）**版本：在不破坏测速功能的前提下，
让所有敏感采集点返回空值。

---

## 二、修改明细

### 2.1 代码层（4 个文件）

| # | 文件 | 修改内容 |
|---|------|----------|
| 1 | `com/huawei/hms/petalspeed/mobileinfo/api/TelephonyManagerCompat.smali` | `getSimInfo()` 尾部三处 setter（ICCID / IMSI / 手机号）改为写入空串；`getImsi()`、`getPhoneNum()` 方法体整体重写为 `return ""` |
| 2 | `com/huawei/hms/petalspeed/mobileinfo/telephonyz/SubscriptionTool.smali` | 同类方法的独立副本 `getImsi()`、`getPhoneNum()` 一并重写为 `return ""` |
| 3 | `com/huawei/hms/petalspeed/speedtest/common/utils/DeviceUtil.smali` | `getDeviceId()` 不再读取 `Settings.Secure.android_id`、不再做 SHA-256，直接 `return ""` |
| 4 | `com/huawei/genexcloud/speedtest/tools/networkstatus/SimFirstFragment.smali` | 去除 `SubscriptionInfo.getIccId()` 直接调用，改为空值赋值 |

其中 `getImsi()` / `getPhoneNum()` 在原版中通过**反射**调用
`TelephonyManager.getSubscriberId(int)` 与 `getLine1Number(int)`
以绕过 API 版本限制 —— 这是被引擎重点标记的行为特征，现已完全移除。

### 2.2 权限层

从 `AndroidManifest.xml` 中移除以下 4 项权限：

| 权限 | 移除原因 |
|------|----------|
| `android.permission.READ_PHONE_STATE` | 读取手机状态 / 设备标识的前置权限 |
| `android.permission.READ_PRIVILEGED_PHONE_STATE` | 系统级特权手机状态读取 |
| `android.permission.REQUEST_GET_TASKS` | 任务栈枚举 |
| `android.permission.REAL_GET_TASKS` | 真实任务栈枚举 |

### 2.3 保留的功能

以下功能**完全未受影响**：

- 上下行测速、延迟 / 抖动 / 丢包测试
- 运营商名称识别（`operatorName` / `simCarrierName`）
- 信号强度、小区信息（CID）、网络类型（2G/3G/4G/5G）
- 5G 检测、路由追踪、网络诊断
- 地图、分享、历史记录

---

## 三、技术参数

| 项目 | 值 |
|------|-----|
| 应用名 | 花瓣测速 |
| 包名 | `com.huawei.genexcloud.speedtest` |
| 版本名 | `4.9.0.119` |
| 版本号 | `40900119` |
| minSdkVersion | 24（Android 7.0） |
| targetSdkVersion | 29（Android 10） |
| 文件大小 | 54,247,782 字节（约 51.7 MB） |
| 签名方案 | v1 + v2 + v3（全部验证通过） |

### 文件校验

```
文件名  : 花瓣测速_v4.9.0.119去高危版.apk
MD5     : b759e238ed35f5c5a09589647836e887
SHA-256 : 8d67e56540d5c826968d2f21dfdf3436c04f80e99a0f38fe5ed3148dabd0f3a1
```

### 签名信息

```
证书 DN        : C=US, O=Android, CN=Android Debug
证书 SHA-256   : 1c6428831892bdf856dca00f154f9f8a745a3172412095017bd15da1d1491fae
```

> 使用 Android 标准 debug 密钥签名，仅用于侧载安装，不具备发布上架资质。

---

## 四、安装说明

### 4.1 前置操作

**必须先卸载原版**，否则会因签名不一致导致安装失败：

```bash
adb uninstall com.huawei.genexcloud.speedtest
```

### 4.2 安装

```bash
adb install 花瓣测速_v4.9.0.119去高危版.apk
```

或直接将 APK 拷贝到手机点击安装。首次安装需在系统设置中允许
「安装未知来源应用」。

### 4.3 验证安装

```bash
# 查看已安装版本
adb shell dumpsys package com.huawei.genexcloud.speedtest | grep versionName
```

---

## 五、兼容性说明

| 情况 | 结果 |
|------|------|
| 覆盖安装原版 | ❌ 签名冲突，需先卸载 |
| 接收华为官方增量更新 | ❌ 签名与组件已变更，无法校验通过 |
| Android 7.0 ~ 15 | ✅ 已验证可安装运行 |
| 华为 HMS 账号登录 | ⚠️ 部分依赖设备标识的账号能力可能受限 |
| 测速核心功能 | ✅ 正常 |

---

## 六、复现步骤

如需自行复现本次修改：

```bash
# 1. 反编译
java -jar apktool.jar d 花瓣测速_v4.9.0.119_修复版.apk -o dec

# 2. 编辑 smali（见「修改明细」第 2.1 节）
#    关键：将敏感方法体替换为 return ""

# 3. 移除 AndroidManifest.xml 中的 4 项高危权限

# 4. 重新打包
java -jar apktool.jar b dec -o patched.apk

# 5. 对齐
zipalign -p -f 4 patched.apk aligned.apk

# 6. 签名（v1 + v2 + v3）
apksigner sign --ks debug.keystore \
  --ks-pass pass:android --key-pass pass:android \
  --ks-key-alias androiddebugkey \
  --v1-signing-enabled true --v2-signing-enabled true --v3-signing-enabled true \
  --min-sdk-version 24 \
  --out signed.apk aligned.apk

# 7. 验证
apksigner verify --verbose signed.apk
```

### 工具链版本

- apktool 3.0.3
- Android SDK Build-Tools 34.0.0
- OpenJDK 17（Amazon Corretto）

---

## 七、验证结果

修改后重新打包的 APK 经以下校验：

| 校验项 | 结果 |
|--------|------|
| 最终 DEX 中 `getIccId` 字符串 | 0 处 |
| 最终 DEX 中 `getSubscriberId` 字符串 | 0 处 |
| 最终 DEX 中 `getLine1Number` 字符串 | 0 处 |
| 最终 DEX 中 `getSimSerialNumber` 字符串 | 0 处 |
| `READ_PHONE_STATE` 权限 | 已移除 |
| `READ_PRIVILEGED_PHONE_STATE` 权限 | 已移除 |
| `REQUEST_GET_TASKS` / `REAL_GET_TASKS` 权限 | 已移除 |
| smali 方法数一致性（36/36、11/11、14/14） | 通过 |
| apksigner 签名验证（v2 / v3） | 通过 |
| jarsigner 签名验证（v1） | 通过 |

---

## 八、免责声明

本项目仅供**个人学习、逆向工程研究与安全分析**使用。

- 本仓库不包含、也不分发原版 APK 的破解或再分发授权
- 修改版 APK 的著作权仍归原始权利人（华为技术有限公司）所有
- 请勿将本产物用于商业用途或二次分发
- 使用者应自行承担安装与使用风险

如权利人认为本项目侵犯其合法权益，请联系删除。

---

## 九、相关文件

| 文件 | 说明 |
|------|------|
| `花瓣测速_v4.9.0.119去高危版.apk` | 本地安装包文件名 |
| `Petalspeed_v4.9.0.119_NoHighRisk.apk` | Release 下载资源名（见下方说明） |
| `README.md` | 本说明文档 |

原版 APK 与本项目的中间产物（反编译源码、签名包等）不随仓库分发。

### 安装包获取

APK 以 **Release 资源**形式分发：

- Release 页面：<https://github.com/zhouzt0121/petalspeed_fixed/releases/tag/v4.9.0.119>
- 直接下载：<https://github.com/zhouzt0121/petalspeed_fixed/releases/download/v4.9.0.119/Petalspeed_v4.9.0.119_NoHighRisk.apk>

> **为什么文件名是英文？**
> GitHub Release 资源接口会在服务端过滤非 ASCII 字符（纯中文名会被重写为 `default.apk`），
> 因此仓库内使用 ASCII 文件名 `Petalspeed_v4.9.0.119_NoHighRisk.apk` 以保证下载稳定性。
> 下载后可自行重命名为 `花瓣测速_v4.9.0.119去高危版.apk`，不影响安装。
