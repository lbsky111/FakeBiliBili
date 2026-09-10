# Android 老项目迁移与构建问题排查经验文档

## 一、 核心概念与版本关系

在 Android 项目构建中，四个核心组件构成了层层依赖的关系。可以用“厨房做饭”来类比：

- **JDK**：最基础的运行环境
- **Gradle**：负责统筹安排所有构建工作。
- **AGP (Android Gradle Plugin)**：告诉构建工具怎么处理 Android 特有的逻辑。
- **Android SDK**：提供 Android 平台的具体 API、编译工具。

**核心依赖链条：JDK → 运行 Gradle → 加载 AGP → 调用 Android SDK。**

**版本强绑定规则**：

1. **Gradle 与 JDK**：每个 Gradle 版本都有其支持的最低和最高 JDK 版本（如 Gradle 9.x 需要 JDK 17-25）。
2. **AGP 与 Gradle**：AGP 的每个版本都强制要求一个最低的 Gradle 版本（如 AGP 7.x 强制要求 Gradle 最低 7.3.3）。
3. **AGP 与 SDK**：AGP 版本决定了它能支持的最高 `compileSdk` 版本。

------

## 二、 推荐的稳定版本组合

对于导入的老项目，盲目使用最新的 Gradle 或最旧的 Gradle 都会引发大量报错。推荐使用以下“黄金组合”进行过渡：

| 组件                            | 推荐版本        | 备注                                     |
| :------------------------------ | :-------------- | :--------------------------------------- |
| **Android Gradle Plugin (AGP)** | 7.4.2           | 稳定且广泛兼容，能较好地平衡新旧项目需求 |
| **Gradle**                      | 7.5             | 满足 AGP 7.4.2 的最低要求，稳定可靠      |
| **Gradle JDK**                  | 17              | 同时兼容 AGP 7.x 和 Gradle 7.5           |
| **compileSdk**                  | 33 (Android 13) | 兼容性广，避开 Android 14 的严苛适配     |

**关键配置文件修改：**

1. `gradle/wrapper/gradle-wrapper.properties`：修改 `distributionUrl` 为 `gradle-7.5-bin.zip`。
2. 根目录 `build.gradle`：修改 `classpath "com.android.tools.build:gradle:7.4.2"`。
3. `Settings` → `Build Tools` → `Gradle`：将 Gradle JDK 设置为 `jbr-17` 或本地 JDK 17。

------

## 三、 常见报错及解决方案（实战踩坑记录）

### 1. JDK 下载失败 / JVM 版本不兼容

- **报错**：`Unable to Download JDK` 或 `Incompatible Gradle JVM version`。
- **原因**：Android Studio 无法自动下载特定版本的 JDK，或者当前运行 Gradle 的 JDK 版本过低（如 Gradle 9.3.0 需要 JDK 17+，但配置了 JDK 8）。
- **解决**：手动安装 JDK 17，并在 `Settings` → `Build Tools` → `Gradle` 中手动指定 JDK 路径。

### 2. Gradle 与 AGP 版本冲突

- **报错**：`Minimum supported Gradle version is 7.3.3. Current version is 6.7.1`。
- **原因**：降级 Gradle 过于激进，低于了当前 AGP 要求的最低版本。
- **解决**：升级 Gradle 到要求的版本（如 7.5），或者降级 AGP 到兼容的旧版本（如 4.2.2）。

### 3. API 缺失报错

- **报错**：`Could not find method forUseAtConfigurationTime()`。
- **原因**：Gradle 版本低于 6.5，但项目中的插件需要 6.5+ 的 API。
- **解决**：将 Gradle 版本提升到 6.5 或以上（推荐直接升至 7.5）。

### 4. 废弃的仓库配置

- **报错**：`Could not find method jcenter()`。
- **原因**：`jcenter()` 仓库已在 2021 年停止维护，并在 Gradle 7.0+ 中被彻底移除。
- **解决**：在所有 `build.gradle` 文件中，将 `jcenter()` 替换为 `mavenCentral()`。

### 5. 废弃的依赖配置

- **报错**：`Could not find method compile()`。
- **原因**：Gradle 7.0+ 彻底删除了 `compile` 配置，必须使用新语法。
- **解决**：全局替换依赖关键字（详见下表）。

| 旧版语法 (Gradle 6 及以前) | 新版语法 (Gradle 7+)        |
| :------------------------- | :-------------------------- |
| `compile`                  | `implementation`            |
| `testCompile`              | `testImplementation`        |
| `androidTestCompile`       | `androidTestImplementation` |
| `provided`                 | `compileOnly`               |
| `debugCompile`             | `debugImplementation`       |

------

## 四、 老项目迁移标准操作流程 (SOP)

如果你以后再次导入一个老项目，建议严格按照以下顺序操作，避免反复报错：

1. **检查 JDK 环境**：确保 Android Studio 中配置了 JDK 17（`Settings` → `Build Tools` → `Gradle`）。
2. **确定 Gradle 版本**：修改 `gradle-wrapper.properties`，将 Gradle 设为 **7.5**。
3. **确定 AGP 版本**：修改根目录 `build.gradle`，将 AGP 设为 **7.4.2**。
4. **替换废弃仓库**：全局搜索 `jcenter()`，全部替换为 `mavenCentral()`。
5. **替换废弃依赖**：全局搜索 `compile`、`testCompile`、`provided` 等，替换为 `implementation`、`testImplementation`、`compileOnly`。
6. **点击大象图标同步**：耐心等待 Gradle 下载依赖并同步。
7. **解决残留报错**：如果还有报错，根据报错信息（如 AndroidManifest 的 `exported` 属性、依赖版本冲突等）逐一解决。

------

## 五、 运行项目的常规步骤

环境配置完成后，运行项目的标准流程：

1. **同步成功**：确保顶部大象图标无红点，底部 `Build` 窗口显示 `BUILD SUCCESSFUL`。
2. **准备设备**：启动 Android 模拟器，或通过 USB 连接开启了“USB 调试”的真机。
3. **选择配置**：在顶部工具栏选择 `app` 模块，并选择目标设备。
4. **点击运行**：点击绿色三角形 （或 `Shift + F10`）。
5. **排查日志**：如果 App 崩溃，切换到 `Logcat` 窗口筛选 `Error` 级别查看具体原因。

> **核心心得**：处理老项目时，**不要追求最新版本的 Gradle**，也不要**无限度降级**。选择中间稳定的版本（如 Gradle 7.5 + AGP 7.4.2 + JDK 17），并在遇到报错时，耐心地根据日志提示替换废弃的语法和仓库，是最高效的迁移策略。