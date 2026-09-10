# Shizuku-m 魔改说明

基于官方 v13.6.0(commit 2650830)的单机离线自启魔改。所有改动均以 `[shizuku-m]` 注释标记。

## 改动清单

| 文件 | 改动 |
| :--- | :--- |
| `manager/.../starter/StarterActivity.kt` | ADB 启动成功后通过 **adbd 内置 `tcpip:5555` 服务**钉死 TCP 端口（真机实测：SELinux 禁止 shell 写 `service.adb.tcp.port`，上游方案的 `setprop` 在 Android 12+ 全部不可用），探测确认后自动回环连接 127.0.0.1:5555 完成 RSA 预授权（30s 超时 × 3 次重试） |
| `manager/.../adb/AdbClient.kt` | 新增 `tcpipCommand()`（复用 ADB 协议服务通道）与可选 `timeoutMs` 构造参数（默认 0，行为与上游一致） |
| `manager/.../home/StartWirelessAdbViewHolder.kt` | 「启动」按钮优先静默探测固定端口（500ms)，活着直连、死了 Toast + 回退原生向导；新增「离线自连」按钮，只探测直连不弹向导 |
| `manager/.../res/layout/home_start_wireless_adb.xml` | 新增 `button_offline` |
| `manager/.../res/values*/strings.xml` | 新增 offline_start_* 字符串；`app_name` 改为 `Shizuku-m` |
| `manager/.../res/mipmap-*/` | 启动图标叠加红色「M」角标（原图备份在 `build-env/icon-backup/`) |
| `manager/build.gradle` | `aapt2` → Windows 下 `aapt2.exe`（上游 bug，仅影响 release 构建） |

## 使用流程

1. **在线激活（每次开机周期一次）**：连任意 Wi-Fi/热点 → 打开无线调试 → 在 Shizuku-m 点「启动」完成常规启动。此时钩子自动钉死 5555 端口并完成预授权（首次会弹「允许 USB 调试」系统框，**勾选「一律允许」**再确认）。
2. **离线自连**：断网/飞行模式下，点「离线自连」（或点「启动」走静默探测）→ 秒级拉起。

## 已知边界

- **重启手机后 5555 端口失效**，需重新在线激活一次（`service.adb.tcp.port` 不持久）。跨重启持久化已实测**不可行**：GT Pro（Android 16 / SELinux Enforcing）上 shell 域写 `persist.adb.tcp.port` 被拒绝，非 root 无解。
- 5555 监听全接口（adbd 限制，无法只绑回环）；传统 ADB 协议有 RSA 认证兜底，陌生设备进不来，但可触发授权弹窗，勿在不可信网络对陌生电脑点「允许」。
- 包名与官方版相同（`moe.shizuku.privileged.api`),**安装前必须先卸载官方版**;签名密钥为项目内自签（`build-env/shizuku-m.jks`)，后续升级可覆盖安装。

## 构建环境（全便携，项目内）

- JDK 21.0.2、Android SDK 36、NDK 29.0.13113456、CMake 3.22.1、Gradle 8.14(wrapper)
- 环境变量：`JAVA_HOME` / `ANDROID_HOME` / `ANDROID_SDK_ROOT` → `build-env/`,`GRADLE_USER_HOME` → `build-env/gradle-home`
- 构建：`gradlew.bat :manager:assembleRelease`，产物在 `out/apk/` 与 `manager/build/outputs/apk/release/`
