# Android 项目编译环境搭建

## 背景

2026-10-01 在 `reading-share-android` 项目上踩了一次坑：从零搭编译环境。
JDK 11 + 没 SDK + 死掉的代理，折腾一阵子。最后 `BUILD SUCCESSFUL`。
把这套流程记下来，下次换机器/换项目能直接抄。

---

## 三件套（缺一不可）

### 1. JDK（Java Development Kit）

**怎么决定装哪个 JDK？** 看项目根目录的 `build.gradle`：

```groovy
classpath 'com.android.tools.build:gradle:8.1.0'
```

对照表：

| AGP 版本 | 最低 JDK |
|---|---|
| 7.0 - 7.4 | JDK 11 |
| 8.0+ | JDK 17（强制） |

**装法：**
```bash
# JDK 17
apt-get install -y openjdk-17-jdk-headless
# 位置：/usr/lib/jvm/java-17-openjdk-amd64
```

**坑：** 报错信息很友好，会告诉你"You are currently using Java 11"，跟着改就行。

---

### 2. Android SDK（不是装一个就完）

SDK = 一组组件，按项目 `compileSdk` 装对应版本。

| 组件 | 命令 | 说明 |
|---|---|---|
| `cmdline-tools` | `sdkmanager` 本身 | 装 SDK 的工具 |
| `platforms;android-34` | `sdkmanager --install "platforms;android-34"` | 对应 `compileSdk 34`，包含 `android.jar`（编译用的 API 类库） |
| `build-tools;34.0.0` | `sdkmanager --install "build-tools;34.0.0"` | aapt2 / d8 / apksigner 等构建工具 |
| `platform-tools` | `sdkmanager --install "platform-tools"` | adb 等。AGP 默认会检查，缺了卡住 |

**完整命令链：**
```bash
export ANDROID_HOME=/root/android-sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH

# 老版本 cmdline-tools（9477386 兼容 JDK 11，最新版的 11076708 要 JDK 17 才能跑 sdkmanager）
mkdir -p $ANDROID_HOME/cmdline-tools && cd $ANDROID_HOME/cmdline-tools
curl -sSL -o /tmp/ct.zip https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip
unzip -q /tmp/ct.zip && mv cmdline-tools latest

yes | sdkmanager --licenses
sdkmanager --install "platform-tools" "platforms;android-34" "build-tools;34.0.0"
```

**注意：** 项目里如果 AGP 8.1，跑起来时还会自动追加 `build-tools;33.0.1`
（项目里有依赖指定），看到 "Install ... 33.0.1" 别慌，让它跑完。

---

### 3. Gradle Wrapper

**不用自己装。** 项目根目录 `gradlew` + `gradle/wrapper/gradle-wrapper.properties`：
```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
```
第一次跑 `./gradlew ...` 会自动下载发行版。

---

## 三者关系图

```
你写代码 (Java/Kotlin/XML)
       │
       ▼
  ┌─────────┐    你的源码 + android.jar (来自 platforms;android-34)
  │ javac   │◀──────────────────────────────┐
  │         │                                │
  └─────────┘    需要 JDK 17                 │
       │                                    │
       ▼                                    │
  .class 文件                                │
       │                                    │
       ▼                                    │
  ┌─────────┐    build-tools 里的工具：       │
  │   d8    │    aapt2 打包资源               │
  │ aapt2   │    d8 转 dex                   │
  │ sign    │    apksigner 签名              │
  └─────────┘                                │
       │                                    │
       ▼                                    │
   app-debug.apk  ◀── Gradle 8.7 串起这一切 │
                          需要 JDK 17 跑自己 │
```

**Gradle 是调度员**，本身要用 JVM 启动（所以也要 JDK 17）。
它按顺序调 javac → aapt2 → d8 → apksigner。

---

## 三大实战坑

### 坑 1：JDK 版本不对齐

```
> Failed to apply plugin 'com.android.internal.application'.
   > Android Gradle plugin requires Java 17 to run. You are currently using Java 11.
```

**解法：** 看 AGP 版本 → 重装 JDK → `export JAVA_HOME=...` → 重跑。

### 坑 2：`gradle.properties` 里的死代理

项目里有人配过：
```properties
systemProp.http.proxyHost=127.0.0.1
systemProp.http.proxyPort=60000
systemProp.https.proxyHost=127.0.0.1
systemProp.https.proxyPort=60000
```

但代理服务挂了，Gradle 报：
```
> Got socket exception during request. It might be caused by SSL misconfiguration
   > Connect to 127.0.0.1:60000 [/127.0.0.1] failed: Connection refused
```

**解法：** 直接**注释掉**这几行（备份原值，不要删）—— 配错代理比不配更糟。

### 坑 3：wrapper 下载 `services.gradle.org` 失败

wrapper 内部用 `HttpURLConnection`，对网络抽风敏感（IPv6 / 端口被封）。
直连 curl 没问题，wrapper 就是拿不到。

**解法 A：** 手动 curl 跟重定向下载，丢到 wrapper cache：
```bash
curl -sSL -o /tmp/gradle-8.7-bin.zip https://services.gradle.org/distributions/gradle-8.7-bin.zip

# 找 wrapper 期望的 cache hash 目录
HASH_DIR=/root/.gradle/wrapper/dists/gradle-8.7-bin/<hash>
mkdir -p $HASH_DIR
cp /tmp/gradle-8.7-bin.zip $HASH_DIR/gradle-8.7-bin.zip
touch $HASH_DIR/gradle-8.7-bin.zip.ok

# 解压到 cache 期望结构（注意 gradle-<v> 这一层）：
cd $HASH_DIR && unzip -q gradle-8.7-bin.zip
```

`<hash>` 在第一次 wrapper 跑失败的错误信息里能看到（路径 `/root/.gradle/wrapper/dists/...`）。

**解法 B：** 配个能用的 HTTP 代理，让 wrapper 自己下。

---

## 一次成功跑通的最小命令

```bash
cd /root/projects/<android-project>

# 1. 环境变量
export ANDROID_HOME=/root/android-sdk
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH

# 2. SDK 路径
echo "sdk.dir=$ANDROID_HOME" > local.properties

# 3. 编译
bash ./gradlew :app:assembleDebug --no-daemon --console=plain

# 产物：app/build/outputs/apk/debug/app-debug.apk
```

**注意事项：**
- `--no-daemon` 防止后台进程残留
- `--console=plain` 输出不被 ANSI 控制符污染（便于日志分析）
- `tee /tmp/gradle-build.log` 留底（编译输出动辄几千行）

---

## 一句话总结

> **编译 Android 项目 = 对齐 JDK 版本（按 AGP 反查）+ 装 Android SDK 三件套
> （platforms / build-tools / platform-tools）+ 让 Gradle wrapper 跑起来。**
>
> 90% 的卡住都是这三件事有任何一个没装对版本。