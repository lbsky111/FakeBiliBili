# FakeBiliBili 项目升级变更记录

## 一、升级目标

将旧版 Android 项目从遗留 Gradle/AGP 配置升级至：
- AGP 7.4.2
- Gradle 7.5
- JDK 17

---

## 二、构建配置变更

### 2.1 根目录 `build.gradle`

| 变更项 | 变更前 | 变更后 |
|--------|--------|--------|
| AGP 版本 | 旧版 | `com.android.tools.build:gradle:7.4.2` |
| compileSdkVersion | 旧值 | `33` |
| buildToolsVersion | 旧值 | `"33.0.1"` |
| targetSdkVersion | 旧值 | `33` |
| repositories | 仅 google/mavenCentral | 增加 `jitpack.io`、`jcenter()`、`flatDir` |

### 2.2 `gradle.properties`

| 变更项 | 变更前 | 变更后 |
|--------|--------|--------|
| AAPT2 禁用 | `android.enableAapt2=false` | **已移除**（AGP 7.x 强制使用 AAPT2） |
| JVM 参数 | `-Xmx1536m` | 增加 10 个 `--add-opens` 参数（解决 ButterKnife JDK 17 兼容性） |

### 2.3 三模块 `build.gradle` 通用变更

| 变更项 | 变更前 | 变更后 |
|--------|--------|--------|
| 依赖声明 | `compile` | `implementation` / `api` |
| 测试依赖 | `testCompile` / `androidTestCompile` | `testImplementation` / `androidTestImplementation` |
| minSdkVersion | 14 / 15 | `16` |
| ProGuard 文件 | `proguard-android.txt` | `proguard-android-optimize.txt` |
| namespace | 无 | 各模块添加 `namespace` 声明 |
| compileOptions | 无 | `sourceCompatibility` / `targetCompatibility` → Java 1.8 |

### 2.4 AndroidManifest.xml 变更

| 变更项 | 变更前 | 变更后 |
|--------|--------|--------|
| package 属性 | 各模块 Manifest 有 `package` | **已移除**（改用 build.gradle 中的 `namespace`） |
| android:exported | 部分 Activity 缺失 | 含 intent-filter 的 Activity 添加 `android:exported="true"` |
| uses-sdk 标签 | 库模块 Manifest 有 `<uses-sdk>` | **已移除**（由 Gradle 管理） |

---

## 三、依赖问题及解决方案

### 3.1 JCenter/Bintray 仓库关闭

**问题**：多个依赖托管在已关闭的 JCenter 上，无法通过远程仓库下载。

**解决方案**：
- 在 `allprojects.repositories` 中添加 `jcenter()` 和 `maven { url 'https://jitpack.io' }`
- 对于仍无法下载的库，从阿里云 Maven 镜像手动下载 AAR 文件到本地 `libs/` 目录

### 3.2 本地 AAR 依赖清单

以下 AAR 从阿里云镜像下载至项目根目录 `libs/` 文件夹：

| 文件名 | 大小 | 来源模块 |
|--------|------|----------|
| `DanmakuFlameMaster-0.9.12.aar` | 200KB | bilibili |
| `fragmentation-1.3.1.aar` | 7KB | common |
| `fragmentation-core-1.3.1.aar` | 101KB | common |
| `ijkplayer-java-0.8.2.aar` | 66KB | ijkplayer |
| `ijkplayer-armv7a-0.8.2.aar` | 1.2MB | ijkplayer |
| `ijkplayer-x86-0.8.2.aar` | 1.6MB | ijkplayer |
| `ijkplayer-exo-0.8.2.aar` | 35KB | ijkplayer |

### 3.3 flatDir 依赖解析失败

**问题**：`flatDir` 在各模块单独声明时，跨模块依赖解析找不到 AAR。当 `bilibili` 模块依赖 `:common` 和 `:ijkplayer` 时，Gradle 无法在 `bilibili/libs/` 中找到这些模块的本地 AAR。

**错误信息**：
```
Could not find :fragmentation-1.3.1:.
Searched in the following locations:
  - file:/.../bilibili/libs/fragmentation-1.3.1.aar
Required by: project :bilibili > project :common
```

**解决方案**：
1. 将所有 AAR 文件集中到项目根目录的 `libs/` 文件夹
2. 在根 `build.gradle` 的 `allprojects.repositories` 中配置 `flatDir { dirs rootProject.file('libs') }`
3. 各模块通过 `implementation(name: 'xxx', ext: 'aar')` 引用

> **注意**：`file('libs')` 在 `allprojects` 中会按各项目目录解析，必须使用 `rootProject.file('libs')` 才能正确指向根目录。

### 3.4 fragmentation AAR 内容不完整

**问题**：从阿里云镜像下载的 `me.yokeyword:fragmentation:1.3.1` AAR 仅包含 3 个类（`SupportActivity`、`SupportFragment`、`BuildConfig`），缺少核心接口（`ISupportFragment`、`SupportActivityDelegate`、`SupportHelper`、`ExtraTransaction`、`FragmentAnimator` 等）。

**原因**：Maven Central 上的 `fragmentation` 仅有包装类，完整实现位于 `fragmentation-core` 模块（原 JCenter 独有）。

**解决方案**：
- 额外下载 `fragmentation-core-1.3.1.aar`（包含 77 个类）
- 在 `common/build.gradle` 中同时依赖两个 AAR：
  ```groovy
  api(name: 'fragmentation-1.3.1', ext: 'aar')
  api(name: 'fragmentation-core-1.3.1', ext: 'aar')
  ```

### 3.5 ButterKnife 8.8.1 与 JDK 17 不兼容

**问题**：ButterKnife 注释处理器 `ButterKnifeProcessor$RClassScanner` 访问 `com.sun.tools.javac.tree.TreeScanner` 时被 JDK 17 模块系统拦截。

**错误信息**：
```
java.lang.NoClassDefFoundError: javax/annotation/Generated
```

**解决方案**：
1. 在 `gradle.properties` 中添加 JVM `--add-opens` 参数：
   ```
   --add-opens=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.processing=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.jvm=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.model=ALL-UNNAMED
   --add-opens=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED
   ```
2. 在 `common` 和 `bilibili` 模块添加 `javax.annotation-api` 注释处理器依赖：
   ```groovy
   annotationProcessor 'javax.annotation:javax.annotation-api:1.3.2'
   ```

### 3.6 传递依赖不可见

**问题**：`implementation` 声明的依赖不具有传递性，`bilibili` 模块无法访问 `common` 和 `ijkplayer` 模块的 `implementation` 依赖（如 support 库、RxJava、Retrofit、Gson、ijkplayer 等）。

**解决方案**：将 `common` 和 `ijkplayer` 模块中被外部使用的依赖从 `implementation` 改为 `api`：

**common 模块**（改为 `api`）：
- `com.android.support:appcompat-v7`
- `com.android.support:design`
- `com.android.support:cardview-v7`
- `com.android.support.constraint:constraint-layout`
- `io.reactivex.rxjava2:*`
- `com.trello.rxlifecycle2:*`
- `org.greenrobot:eventbus`
- `com.google.code.gson:gson`
- `com.squareup.retrofit2:*`
- `com.squareup.okhttp3:*`
- `com.jakewharton:butterknife`
- `com.google.dagger:dagger`
- `de.hdodenhof:circleimageview`
- `com.facebook.fresco:fresco`
- `me.drakeet.multitype:multitype`
- `fragmentation-1.3.1`、`fragmentation-core-1.3.1`

**ijkplayer 模块**（改为 `api`）：
- `ijkplayer-java-0.8.2`

---

## 四、ndk 配置变更

**问题**：`armeabi` ABI 在新版 NDK 中已弃用。

**解决方案**：从 `bilibili/build.gradle` 的 `abiFilters` 中移除 `armeabi`，保留 `armeabi-v7a` 和 `x86`。

---

## 五、最终构建结果

```
BUILD SUCCESSFUL in 1m 23s
85 actionable tasks: 32 executed, 53 up-to-date
```

三个模块（`bilibili`、`common`、`ijkplayer`）均编译通过，生成 debug APK。

---

## 六、文件变更清单

| 文件 | 变更类型 |
|------|----------|
| `build.gradle`（根目录） | 修改 — AGP 版本、ext 版本、allprojects 仓库、flatDir |
| `gradle.properties` | 修改 — 移除 AAPT2 禁用、增加 JVM --add-opens 参数 |
| `bilibili/build.gradle` | 修改 — namespace、依赖声明、compileOptions、ndk abiFilters、proguard 文件名 |
| `common/build.gradle` | 修改 — namespace、依赖声明（implementation→api）、compileOptions |
| `ijkplayer/build.gradle` | 修改 — namespace、依赖声明（implementation→api）、compileOptions |
| `bilibili/src/main/AndroidManifest.xml` | 修改 — 移除 package、添加 android:exported |
| `common/src/main/AndroidManifest.xml` | 修改 — 简化，移除 package/uses-sdk |
| `ijkplayer/src/main/AndroidManifest.xml` | 修改 — 添加 android:exported |
| `libs/DanmakuFlameMaster-0.9.12.aar` | 新增 |
| `libs/fragmentation-1.3.1.aar` | 新增 |
| `libs/fragmentation-core-1.3.1.aar` | 新增 |
| `libs/ijkplayer-java-0.8.2.aar` | 新增 |
| `libs/ijkplayer-armv7a-0.8.2.aar` | 新增 |
| `libs/ijkplayer-x86-0.8.2.aar` | 新增 |
| `libs/ijkplayer-exo-0.8.2.aar` | 新增 |
