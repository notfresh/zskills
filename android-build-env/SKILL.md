---
name: android-build-env
description: Use when compiling an Android project from scratch or troubleshooting build errors. Three-step bootstrap (JDK + SDK + Gradle), proxy / sandbox-detour pitfalls.
---

# Android 项目编译环境搭建

Use when the user wants to build / compile an Android project and the local
environment isn't ready, or hits AGP / SDK / proxy errors.

## 三件套

1. **JDK** — 按 `build.gradle` 里的 AGP 版本反查：
   - AGP 7.x → JDK 11
   - AGP 8.x → JDK 17 必须
2. **Android SDK** — 三组件，分开装：
   - `cmdline-tools`（装 SDK 的工具；JDK 11 兼容版 = 9477386）
   - `platforms;android-<compileSdk>`
   - `build-tools;<version>`（AGP 8.1 → 34.0.0）
   - `platform-tools`（adb 等；AGP 默认会检查）
3. **Gradle wrapper** — `gradle/wrapper/gradle-wrapper.properties` 里写死版本，**不用自己装**。

## 一句话命令链（root 装好 JDK 17 之后）

```bash
export ANDROID_HOME=/root/android-sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

# 老版 cmdline-tools（JDK 11 兼容）
curl -sSL -o /tmp/ct.zip https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip
mkdir -p $ANDROID_HOME/cmdline-tools && cd $ANDROID_HOME/cmdline-tools
unzip -q /tmp/ct.zip && mv cmdline-tools latest

yes | sdkmanager --licenses
sdkmanager --install "platform-tools" "platforms;android-34" "build-tools;34.0.0"

echo "sdk.dir=$ANDROID_HOME" > <project>/local.properties
cd <project> && bash ./gradlew :app:assembleDebug --no-daemon --console=plain
```

## 三大坑

1. **JDK 版本不对齐** — 报错 `AGP requires Java 17`，按 AGP 版本表重装。
2. **gradle.properties 里死了的代理** — `127.0.0.1:<port>` 配了但代理服务挂了，
   报 `SSL misconfiguration / Connection refused`。**直接注释掉**那几行（备份原值）。
3. **wrapper 下载 services.gradle.org 失败** — IPv6 / 网络敏感。
   手动 curl 跟重定向下载 zip，丢到
   `~/.gradle/wrapper/dists/gradle-<v>-bin/<hash>/gradle-<v>-bin.zip`
   并 `touch .../gradle-<v>-bin.zip.ok`，wrapper 就不重下了。

## 详细参考

- 实战案例 + 关系图：见本目录下 `android-build.md`