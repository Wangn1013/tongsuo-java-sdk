# Tongsuo Java SDK ARM64 编译记录

本文记录了 `Tongsuo-Project/tongsuo-java-sdk` 在 `Linux ARM64 (aarch64)` 环境下的构建全过程，包括准备工作、离线材料打包、实际编译步骤、遇到的问题、临时修复方式，以及最终产物说明。

适用场景：

- 目标平台是 `Linux aarch64`
- 业务运行环境仍可能是 `JDK 8`
- 构建环境使用 `JDK 17`
- 希望尽量减少 ARM 机器联网下载，甚至完全离线构建

最终成功产物：

```text
tongsuo-openjdk-1.1.0-linux-aarch_64.jar
```

## 1. 结论

这次最终证明：

- `tongsuo-java-sdk` 可以在 `Linux ARM64` 上成功编译
- 直接原仓库构建通常不够，需要补充 `linux aarch64` 的 Gradle 支持
- 构建时用 `JDK 17` 是可行的
- 产物仍然可以保持 Java 8 目标版本，因为项目里使用了 `options.release = 8`
- 最终得到的 `tongsuo-openjdk-1.1.0-linux-aarch_64.jar` 可以作为 ARM64 平台包使用，角色等同于官方示例中的 `linux-x86_64` 包

## 2. 环境信息

ARM64 目标机器基础环境：

```text
uname -m = aarch64
gcc = 8.3.0
perl = 5.28.1
```

Java 情况：

- 机器原本默认有 `JDK 8`
- 但该项目构建阶段不能使用 `JDK 8`
- 最终构建使用了 `JDK 17`

构建时的关键结论：

- `JDK 8` 不能用于编译本项目
- `JDK 11` 理论上也可以，但本次离线环境最后实际使用的是 `JDK 17`
- 业务运行后续仍可继续验证 `JDK 8`

## 3. 为什么不能直接用 JDK 8 编译

在最开始尝试 `./gradlew help --refresh-dependencies` 时，出现了这类错误：

```text
No matching variant ... component compatible with Java 11 and the consumer needed ... Java 8
```

根因：

- 仓库里的 Gradle 插件和构建逻辑要求构建机至少使用 `Java 11+`

后续核实到：

- 项目根 `build.gradle` 使用了 Java toolchain
- 编译目标仍通过 `options.release = 8` 保持 Java 8 兼容

所以应当这样理解：

- 构建机：`JDK 11` 或 `JDK 17`
- 目标 class 版本：Java 8

## 4. Windows 侧准备过程

因为 ARM 机器不方便直接走完整联网构建，所以先在 Windows 上准备材料。

### 4.1 下载 Gradle 本体

项目实际需要：

```text
gradle-7.5-bin.zip
```

这个文件后来被带到 ARM 机器，通过 `gradle-wrapper.properties` 里的本地 `file:///` 地址引用，避免 ARM 机器下载 Gradle 本体。

### 4.2 下载源码

在 Windows 上下载了两个仓库：

```text
E:\tongsuo\Tongsuo
E:\tongsuo\tongsuo-java-sdk
```

### 4.3 Windows 上用 JDK 17 预热 Gradle 依赖

一开始用旧版 `JDK 11.0.2` 时，访问 Maven Central 出现：

```text
peer not authenticated
```

原因是：

- JDK 太老
- 内置证书链过旧

切换到 `JDK 17` 后，以下命令成功：

```powershell
.\gradlew.bat help --refresh-dependencies -PtongsuoHome=E:/tongsuo/tongsuo-home
```

### 4.4 Windows 上遇到的额外问题

#### 4.4.1 PowerShell 不能直接执行当前目录脚本

需要这样执行：

```powershell
.\gradlew.bat --version
```

而不是：

```powershell
gradlew.bat --version
```

#### 4.4.2 项目要求 `tongsuoHome/include/lib` 都存在

只是为了通过配置检查，Windows 上先准备了占位目录：

```text
E:\tongsuo\tongsuo-home
  include
  lib
```

并把 `Tongsuo` 源码里的 `include` 拷贝进去。

#### 4.4.3 `Copy-Item` 复制 `.gradle\caches` 失败

原因是：

- 目录过深
- Windows PowerShell 在 Gradle 缓存上容易翻车

改用：

```powershell
robocopy C:\Users\wang\.gradle\wrapper E:\tongsuo\offline-bundle\gradle-home\wrapper /E /R:1 /W:1
robocopy C:\Users\wang\.gradle\caches E:\tongsuo\offline-bundle\gradle-home\caches /E /R:1 /W:1
```

### 4.5 Windows 上打离线包

最终将这些内容一起打包：

- `Tongsuo`
- `tongsuo-java-sdk`
- `gradle-7.5-bin.zip`
- `.gradle/wrapper`
- `.gradle/caches`

打成：

```text
tongsuo-arm64-offline.tar.gz
```

## 5. Linux ARM64 侧解包后遇到的问题

### 5.1 tar 解压提示 `SCHILY.fflags`

解压时出现：

```text
tar: 忽略未知的扩展头关键字‘SCHILY.fflags’
```

这不是致命错误，只是 Windows 打包工具写入的扩展元数据被 Linux `tar` 忽略，不影响实际文件内容。

### 5.2 脚本执行权限丢失

从 Windows 打包再到 Linux，很多脚本失去了执行权限，例如：

- `config`
- `Configure`
- `gradlew`

需要补权限：

```bash
chmod +x config Configure
chmod +x gradlew
```

### 5.3 Windows 换行符 CRLF 导致 `/bin/sh^M`

报错类似：

```text
/bin/sh^M: interpreter error
```

原因是：

- 文件带有 Windows CRLF 换行

修复方式：

```bash
sed -i 's/\r$//' config Configure gradlew
```

必要时对更多脚本批量处理：

```bash
find . -type f \( -name "config" -o -name "Configure" -o -name "*.sh" -o -name "*.pl" \) -exec sed -i 's/\r$//' {} \;
```

## 6. ARM64 实际构建步骤

### 6.1 设置构建用 JDK 17

确认 `java -version` 为 17。

### 6.2 修改 wrapper 使用本地 Gradle zip

修改文件：

```text
tongsuo-java-sdk/gradle/wrapper/gradle-wrapper.properties
```

将：

```properties
distributionUrl=file:///E:/tongsuo/gradle-7.5-bin.zip
```

改为：

```properties
distributionUrl=file:///home/wangning/build/pkg/gradle-7.5-bin.zip
```

### 6.3 编译安装 Tongsuo

执行：

```bash
cd /home/wangning/build/Tongsuo
./config no-shared enable-ntls enable-weak-ssl-ciphers --release --prefix=$HOME/tongsuo --libdir=$HOME/tongsuo/lib
make -j$(nproc)
make install
```

这一步成功时会看到：

```text
Configuring Tongsuo version 8.5.0-pre1 for target linux-aarch64
```

说明 Tongsuo 本身已经正确识别为 `linux-aarch64`。

### 6.4 设置离线 Gradle 缓存

```bash
export GRADLE_USER_HOME=/home/wangning/build/gradle-home
export TONGSUO_HOME=$HOME/tongsuo
```

## 7. 编译 tongsuo-java-sdk 时遇到的关键坑

### 7.1 `No native builds selected.`

报错位置：

```text
openjdk/build.gradle
```

原因：

- `NativeBuildInfo` 里只定义了 `LINUX_X86_64`
- 没有 `LINUX_AARCH64`

修复：

在 `openjdk/build.gradle` 中补充：

```gradle
LINUX_AARCH64("linux", "aarch_64"),
```

### 7.2 Gradle 离线状态下尝试下载 JDK 11 toolchain

报错类似：

```text
Unable to download toolchain matching these requirements: languageVersion=11
No cached resource ... for offline mode
```

原因：

- 项目把 Java toolchain 固定成了 11
- 但本地离线机器只有 JDK 17

修复：

将根 `build.gradle` 里的：

```gradle
languageVersion = JavaLanguageVersion.of(11)
```

改为：

```gradle
languageVersion = JavaLanguageVersion.of(17)
```

同时保留：

```gradle
options.release = 8
```

这样：

- 构建用 JDK 17
- 目标字节码仍保持 Java 8

### 7.3 `Don't know how to build for platform 'linux_aarch64'`

报错来自：

```text
:conscrypt-constants:compileGenExecutableGenCpp
```

原因：

- 根 `build.gradle` 的 `toolChains` 只对 macOS ARM 做了 target 配置
- 没有告诉 `clang/gcc` 如何处理 `linux_aarch64`

修复：

在根 `build.gradle` 的 `toolChains` 中增加：

```gradle
clang(Clang) {
    target("linux_aarch64")
    ...
}
gcc(Gcc) {
    target("linux_aarch64")
}
```

### 7.4 离线缓存缺 `errorprone` 和测试依赖

报错包括：

- `com.google.errorprone:javac:9+181-r4173-1`
- `org.bouncycastle:bcpkix-jdk15on:1.63`
- `org.bouncycastle:bcprov-jdk15on:1.63`
- `junit:junit:4.13.2`

原因：

- Windows 侧只做了 `help --refresh-dependencies`
- 没有把完整构建链路所有依赖都拉全

处理策略：

- 不再跑根项目 `build`
- 改为只构建真正需要的 `:tongsuo-openjdk:assemble`
- 临时关闭 `errorprone`

### 7.5 `X25519_PUBLIC_VALUE_LEN` / `X25519_PRIVATE_KEY_LEN` 未定义

在 JNI C++ 编译日志中定位到错误：

```text
use of undeclared identifier 'X25519_PUBLIC_VALUE_LEN'
use of undeclared identifier 'X25519_PRIVATE_KEY_LEN'
```

报错位置：

```text
common/src/jni/main/cpp/conscrypt/native_crypto.cc:2242
```

原因：

- 当前 Tongsuo/OpenSSL 头文件里没有这两个宏

观察到这段代码本身就是在校验数组长度是否为 `32`，并且异常文案也写死为 `32`，所以最小修复是直接替换成常量。

将：

```cpp
if (outPublic.size() != X25519_PUBLIC_VALUE_LEN || outPrivate.size() != X25519_PRIVATE_KEY_LEN) {
```

改为：

```cpp
if (outPublic.size() != 32 || outPrivate.size() != 32) {
```

## 8. 最终保留下来的源码修改

以下为这次成功编译 ARM64 时实际需要的关键修改。

### 8.1 `openjdk/build.gradle`

增加：

```gradle
LINUX_AARCH64("linux", "aarch_64"),
```

### 8.2 根 `build.gradle`

在 `toolChains` 中给 `clang/gcc` 增加：

```gradle
target("linux_aarch64")
```

将 Java toolchain 调整为：

```gradle
languageVersion = JavaLanguageVersion.of(17)
```

### 8.3 `common/src/jni/main/cpp/conscrypt/native_crypto.cc`

将 X25519 长度判断改为直接比较 `32`。

### 8.4 临时关闭 `errorprone`

本次为了离线构建顺利通过，临时做了这几项处理：

- 注释 `plugins {}` 中的 `net.ltgt.errorprone`
- 注释 `apply plugin: "net.ltgt.errorprone"`
- 注释 `errorprone(...)`
- 注释 `errorproneJavac(...)`

这部分属于“为了离线构建简化依赖”的处理，是否长期保留，要看后续是否愿意补齐离线缓存或调整构建流程。

## 9. 最终成功使用的构建命令

在 ARM64 机器上，最终成功的是：

```bash
export GRADLE_USER_HOME=/home/wangning/build/gradle-home
export TONGSUO_HOME=$HOME/tongsuo

cd /home/wangning/build/tongsuo-java-sdk
./gradlew --offline :tongsuo-openjdk:assemble -x test -PtongsuoHome=$TONGSUO_HOME
```

最终结果：

```text
BUILD SUCCESSFUL
```

## 10. 最终产物

产物位于：

```text
/home/wangning/build/tongsuo-java-sdk/openjdk/build/libs
```

关键产物：

```text
tongsuo-openjdk-1.1.0-linux-aarch_64.jar
```

这个 jar 的定位是：

- 作为 `Linux ARM64` 平台包使用
- 角色等同于 `linux-x86_64` 平台包
- 后续业务项目中应使用 `linux-aarch_64` classifier，而不是继续使用 `linux-x86_64`

## 11. 业务使用说明

如果业务原来在 x86 上使用的是：

```text
tongsuo-openjdk-1.1.0-linux-x86_64.jar
```

那么在 ARM64 上，对应应改为：

```text
tongsuo-openjdk-1.1.0-linux-aarch_64.jar
```

Java 代码使用方式原则上不变，主要变化在于：

- 运行机器必须是 `Linux ARM64`
- 依赖 classifier 要替换成 `linux-aarch_64`

## 12. 本次最重要的经验

### 12.1 原生 ARM64 编译比交叉编译省事

如果目标就是 `Linux aarch64`，最稳妥的路线仍然是：

- 在 ARM64 机器上原生编译

### 12.2 离线构建真正难的是依赖缓存，不是源码本身

真正消耗时间的不是 C/C++ 编译，而是：

- Gradle 本体
- plugin portal
- Maven 依赖
- errorprone
- test 依赖

### 12.3 Windows 打包到 Linux 会带来两个固定坑

- 执行权限丢失
- CRLF 换行符问题

### 12.4 `linux aarch64` 支持并不是仓库当前默认完整具备

至少本次验证下来，需要补：

- `NativeBuildInfo`
- `toolChains`
- 部分 JNI 兼容性修正

### 12.5 构建用 JDK 与运行用 JDK 要分开看

本项目可以采用：

- 构建：`JDK 17`
- 产物目标：`Java 8`
- 业务运行：后续继续验证 `JDK 8`

## 13. 建议后续动作

如果后续希望把 ARM64 支持长期固化，建议：

1. 将本次 ARM64 相关 Gradle 修改整理成正式补丁
2. 确认是否参考上游 PR 方案统一合并
3. 评估是否保留 `errorprone`，或者改成在联网 CI 上启用
4. 增加 ARM64 构建验证流程
5. 补一份最小运行验证，确认 `JDK 8 + Linux ARM64` 的实际加载情况

