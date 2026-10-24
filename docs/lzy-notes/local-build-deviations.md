# 官方 Elasticsearch 源码树：本地构建改动清单与原因

> 记录范围：`D:\lzycode\lzy4j\lzy-elasticsearch\elasticsearch`，本地分支 `release/v9.5.5`（内容 = tag `v9.5.5`）。
> 对比基准：`git diff v9.5.5 --ignore-cr-at-eol`。
> 最后核对时间：2026-10-24。
> 同目录姊妹篇：[untracked-files.md](./untracked-files.md) —— 工作区里未被 git 跟踪的文件都是什么、怎么生成的。

## 1. 改动总账

| 判据 | 命令 | 结果 |
| --- | --- | --- |
| 改动文件数 | `git diff v9.5.5 --ignore-cr-at-eol --stat` | **13 个文件，+73 / −15** |
| 是否新增/删除文件 | `git diff v9.5.5 --ignore-cr-at-eol --name-status \| awk '{print $1}' \| sort \| uniq -c` | **13 M，0 A，0 D** |
| 是否触碰业务源码 | 过滤 `build-tools*/`、`gradle*`、两个 `build.gradle` 之外的路径 | **无输出（0 个）** |
| build-tools 之外的 `.java` 改动 | 同上再 `grep '\.java$'` | **0** |
| 是否联网下载过 JDK | 全部 11 轮构建日志 `grep -i jdk \| grep -ci download` | **0** |

全部改动只落在这几类文件里：

- Gradle 构建逻辑（`build-tools-internal/`、`build-tools/` 下的 java/groovy）：7 个
- 构建配置（`gradle.properties`、`gradle/gradle-daemon-jvm.properties`、`gradle/verification-metadata.xml`）：3 个
- 两个子项目的构建脚本（`qa/vector/build.gradle`、`test/immutable-collections-patch/build.gradle`）：2 个
- 一个被构建读取的属性文件（`gradle.properties` 已计入上面）

**没有**任何 `server/`、`modules/`、`libs/`、`x-pack/` 的源码、资源、REST 规格、传输版本清单被修改。

## 2. 为什么要改：三条硬约束

本机的约束决定了改动方向，理解这三条就能理解下面每一处修改：

1. **只允许使用本机已装的 JDK**（17 / 21 / 25 / 27，装在 `D:\01code\kits`），路径由用户级 `org.gradle.java.installations.paths` 提供；任何"联网取 JDK"都不允许，包括 Gradle toolchain 自动装机和 ES 自己的 `JdkDownloadPlugin`。
2. **不新增 java 类、不删除业务代码**；偏离只能改现有构建文件，且**上游代码用注释保留、不删行**。
3. **工程内不写绝对路径**（配置要可移植），且这类配置属于工程而不是用户级 Gradle 目录。

上游构建逻辑里大量写死了 JDK 的**厂商**（`JvmVendorSpec.ORACLE` / `ADOPTIUM`）。Gradle 的 toolchain 匹配对厂商是**严格**的：本机 27 版本是 Azul Zulu、17 版本是 Oracle，所以凡是上游要求 "Oracle 27 / Temurin 17 / Temurin 21" 的地方都会报

```
Cannot find a Java installation on your machine (Windows 11 10.0 amd64) matching:
{languageVersion=27, vendor=Oracle, ...}. Toolchain auto-provisioning is not enabled.
```

而"放开厂商约束"（直接不设 vendor）等于删除上游约束，不符合约束 2，所以每一处都改成**替换成本机实际存在的厂商**，并把上游原行注释保留。

## 3. 全绿配方（可复现命令）

```bash
./gradlew --continue -Dtests.bwc.git_fetch_latest=false assemble \
  -x :distribution:archives:linux-tar:assemble \
  -x :distribution:archives:darwin-tar:assemble \
  -x :distribution:archives:darwin-aarch64-tar:assemble \
  -x :distribution:archives:linux-aarch64-tar:assemble \
  -x :distribution:archives:windows-zip:assemble \
  -x :distribution:packages:deb:assemble \
  -x :distribution:packages:rpm:assemble \
  -x :distribution:packages:aarch64-deb:assemble \
  -x :distribution:packages:aarch64-rpm:assemble \
  -x :distribution:docker:assemble \
  -x :distribution:docker:docker-export:assemble \
  -x :distribution:docker:docker-aarch64-export:assemble \
  -x :distribution:docker:cloud-ess-docker-export:assemble \
  -x :distribution:docker:cloud-ess-docker-aarch64-export:assemble \
  -x :distribution:docker:ironbank-docker-export:assemble \
  -x :distribution:docker:ironbank-docker-aarch64-export:assemble \
  -x :distribution:docker:wolfi-docker-export:assemble \
  -x :distribution:docker:wolfi-docker-aarch64-export:assemble \
  -x :distribution:docker:cloud-ess-fips-docker-export:assemble \
  -x :distribution:docker:cloud-ess-fips-docker-aarch64-export:assemble \
  -x :distribution:bwc:major1:assemble \
  -x :distribution:bwc:major1:checkoutBwcBranch
```

实测（2026-10-24，13 文件最终态）：`BUILD SUCCESSFUL in 6m 12s`，1809 可执行任务（115 executed / 1694 up-to-date），`FAILED` 计数 0，`gradle_exit=0`。首次全量（第 5 轮）耗时 14h30m，之后因为绝大多数任务 up-to-date，重跑只要几分钟。

三个注意点：

- **`-x` 只能写日志里确实出现过的任务名**。`-x` 引用不存在的任务会让整个构建在选图阶段就失败（`Selection failed ... task 'assemble' not found in project ':distribution:bwc:major2'`）。`settings.gradle` 里列了 9 个 bwc 子工程，但**只有 `major1` 挂了 `assemble`**，别照着项目清单批量生成 `-x`。
- **归档任务注册在父工程 `:distribution:archives` 上**，是子工程的 `assemble` 通过 `InternalDistributionArchiveSetupPlugin`（`sub.getTasks().named("assemble").dependsOn(archiveTask)`）把它们拉进图的，所以只 `-x :distribution:archives:assemble` 完全无效，必须排到具体子工程；docker 同理，入口是 10 个 `*-docker-export` 子工程。
- **`-Dtests.bwc.git_fetch_latest=false` 是上游自带的开关**（`InternalBwcGitPlugin` 读该系统属性）。`fetchLatest` 会 `git fetch --all --tags --force` 官方 `elastic` remote，也就是拉官方 git 历史，属禁止项；用开关跳过优于改代码。

### 被排除目标各自的不可行原因

| 排除对象 | 原因 |
| --- | --- |
| `distribution/archives/*`、`distribution/packages/*` | 走 `distribution/build.gradle` 的 `jdkFiles` → `project.jdks."bundled_${os}_${arch}"`，即 `JdkDownloadPlugin` 联网拉 **OpenJDK 27+35 的 5 份外部平台归档**。本机只有 Windows 版，darwin/linux 物理上无法本地替代 |
| `distribution/docker*` | 依赖 `:distribution:archives:linux-tar`（`distribution/docker/build.gradle:117-120`），且需要 Docker 守护进程；本机 Docker Desktop 服务未运行（`failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`） |
| `distribution/bwc/major1` | `createClone` 是把**本地工作区根** clone 到 `build/bwc/checkout-8.19`（不联网），但 `checkoutBwcBranch` 检的是 `elastic/8.19` **远端跟踪 ref**，只有真去 fetch 官方远端才存在 → 与"不拉官方 git 历史"冲突。它是 BWC 测试设施，不是本工程代码的构建 |

`integ-test-zip` **不需要**排除：test distro 的 copySpec 不打包 `jdk/`。

## 4. 逐条改动：内容、原因、证据

### A 类：全工程 `assemble` 必须（回退就红）

**A1. `build-tools-internal/.../MrjarPlugin.java`（+26 / −1，2 个 hunk）**
新增 `private boolean hasToolchainFor(int)`，在 `apply()` 的调用点过滤掉本机没有对应 JDK 的多-release 源集。

- **为什么**：`findSourceVersions(project, minJavaVersion)` 扫到 `src/mainNN` 后，若 `NN` 低于最小版本会抛 `IllegalArgumentException`；而更根本的是本机只有 17/21/25/27，`main22`/`main26` 这两层无法编译。
- **写法要点**：`javaToolchains.compilerFor(spec).isPresent()` 在"无匹配且禁止自动装机"时**不是返回空而是抛 RuntimeException**，所以 `hasToolchainFor` 要 `try/catch`。保持 `findSourceVersions` 仍是 `private static`（不改上游签名），偏离落在调用点。
- **实测**：全绿日志恰好 4 条 warn——`:libs:cli-terminal`/`:libs:foreign-adapter` 的 `main22`、`:libs:entitlement`/`:libs:entitlement:qa:entitlement-test-plugin` 的 `main26`；`main25`/`main27` 照编。
- **本树事实**：版本层目录只有 8 个，全在 `libs/` 下；`x-pack` 没有任何 `mainNN` 目录。判据是 `find . -type d -name 'main[0-9][0-9]' -not -path '*/build/*'` 加逐模块 `ls <模块>/src` 复核。

**A2. `build-tools-internal/.../dependencies/rules/ComponentMetadataRulesPlugin.java`（+15 / −3）**
Apache Arrow 三个模块（`arrow-format`、`arrow-memory-core`、`arrow-vector`）的"排除全部传递依赖"从可缓存规则类 `ExcludeAllTransitivesRule` 改成等价的非缓存 `Action<ComponentMetadataDetails>`。

- **为什么**：用规则类注册时，解析 `:x-pack:plugin:esql:moduleCompileClasspath` 会失败：`Unexpected state for resolution: Unknown`（Gradle 9.7.1 下的缓存型规则与 module 变体解析冲突）。改成 action 后依赖图完全一致且能正常解析。
- **实测**：`D:\lzycode\client-esql-build.log:44` 保留了改前的报错原文。
- **可再收紧的候选（未验证）**：`components.withModule(id, RuleClass.class, details -> {})` 三参形式理论上也可能绕开缓存路径，但没实测，现状保持已验证的写法。

**A3. `build-tools-internal/.../info/GlobalBuildInfoPlugin.java`（+6）+ `gradle.properties` 的 branches 属性（+2）**
`getDevelopmentBranches()` 里把**相对路径**的 `branches-file-location` 锚定到工作区根；`gradle.properties` 新增 `org.elasticsearch.build.branches-file-location=branches.json`。

- **为什么**：默认值是一个 `raw.githubusercontent.com` 的 URL，本机直连该域名超时（`curl` 15s 无响应），配置期直接抛 `Failed to download branches.json from: ...`，整个构建在任何任务跑之前就死。上游自带的兜底只在 `--offline` 且"配置值仍是 http URL"时生效，联网跑就绕不过去。
- **为什么必须加锚定代码**：用户要求工程内写相对路径而不是绝对路径，但上游把相对路径按 **Gradle daemon 的工作目录**解析，于是红：`A problem occurred evaluating project ':docs' > Failed to read branches.json from: branches.json`（实测：只回退这 6 行代码，`./gradlew help` 立刻失败）。
- **两条并存不重复**：设了属性之后，上游第 227-247 行的 offline 分支不再触发，但它仍在保护"没设属性 + offline"的场景。

**A4. `build-tools-internal/.../ElasticsearchJavaPlugin.java`（+4 / −1）**
javadoc toolchain 的 `JvmVendorSpec.ORACLE` → `AZUL`（原行注释保留）。

- **为什么**：javadoc 任务用最小运行时版本（21）的 JDK，上游钉 Oracle，本机 21 是 Azul Zulu。
- **实测**：回退后 `:libs:logging:javadoc` 报 `Error while evaluating property 'javadocTool' ... {languageVersion=21, vendor=Oracle}`。javadoc 确实在 assemble 图里：全量日志中 `:javadoc` 出现 108 次、`Task :libs:logging:javadoc` 出现 2 次。

**A5. `gradle/gradle-daemon-jvm.properties`（改 1 行）**
`toolchainVendor=ADOPTIUM` → `AZUL`；6 条 `toolchainUrl.*`（adoptium 二进制地址）**原样保留**。

- **为什么必须改这个文件**：它是 Gradle 9 的 daemon JVM 条件文件，上游原文要求 "Temurin 21"；本机没有 Temurin，联网装机又被禁，于是**任何任务都起不来**：`Cannot find a Java installation ... {languageVersion=21, vendor=Eclipse Temurin, ...}. Toolchain auto-provisioning is not enabled.`
- **为什么这是最小形式**：`toolchainUrl.*` 只在需要 provisioning 时才用到，`org.gradle.java.installations.auto-download=false` 下永远不会被读取，所以保留它们不改变行为。之前"注释掉 6 条 URL + 注释掉 vendor"（共 7 行）是过度改动。
- **实测**：改成 AZUL 后 `./gradlew --offline help` = `BUILD SUCCESSFUL in 14s`；IDEA sync 的 Hamster 横幅显示 `JDK Version : 21.0.12.1+1-LTS (Azul Zulu)`。

**A6. `gradle.properties` 的 `org.gradle.java.installations.auto-download=false`（+3）**
- **为什么**：这是约束 1 的实现手段。它和已有的 `auto-detect=false` **不重复**——`auto-detect` 管"要不要扫描本机 JDK"，`auto-download` 管"匹配不到时要不要联网装"。少了这条，任何 toolchain 失配都会触发联网取 JDK。

### B 类：不在 assemble 图里，但对应入口实测必须

**B1. `test/immutable-collections-patch/build.gradle`（改 1 行）**、**B2. `qa/vector/build.gradle`（改 1 行）**
`generatePatch` / `checkVec` 的 toolchain 里 bundled JDK（27）厂商 `ORACLE` → `AZUL`。

- **实测**：分别回退后
  - `:test:immutable-collections-patch:generatePatch` → `Failed to calculate the value of task ... property 'javaLauncher' > Cannot find a Java installation ... {languageVersion=27, vendor=Oracle}`
  - `:qa:vector:checkVec` → 同样报错
- **定位更正**：这两个任务**不在全量 assemble 日志里**（`grep -o "Task :qa:vector:[A-Za-z]*"` 只有 `assemble/classes/compileJava/jar/processResources`），所以它们是"入口所需"而不是"全绿所需"。

### C 类：只在测试 / IDE 入口生效

**C1. `build-tools-internal/.../test/rest/RestTestBasePlugin.java`（+4 / −1）**、**C2. `build-tools/.../testclusters/TestClustersPlugin.java`（+4 / −1）**
JDK 17 兜底 launcher 的 `ADOPTIUM` → `ORACLE`（本机 17 是 Oracle JDK）。

- **可达性**：两处最终都由 `ElasticsearchNode.java:819` / `RestTestBasePlugin` 第 306 行的 `jdkIsIncompatibleWithOS(version)` 触发，条件注释写明是"8.10.4 及更早、出厂带 <21 JDK 的发行版"。
- **实测**：两个文件**同时**回退成上游后 `./gradlew --offline help` = `BUILD SUCCESSFUL in 3m 56s` → 配置期完全不需要；本机也拿不到那些老版本发行版（BWC 被排除），所以这两条是当前**最接近冗余**的改动。**未端到端验证**。

**C3. `build-tools-internal/src/main/groovy/elasticsearch.runtime-jdk-provision.gradle`（2 处，bundled 27 厂商 `ORACLE` → `AZUL`）**
- 两处都包在 `if (buildParams.getIsRuntimeJavaHomeSet() == false)` 的 else 分支里（第 24、41 行），即**只有不带 `-Druntime.java=` 跑测试时**才会解析。日常用法带该参数，所以这两处实际不触发；但它们是默认路径，回退会让"不带参数跑 test"变红。机制与 B 类相同（同类报错已实测两次），**本条未单独回退实测**。

**C4. `build-tools-internal/.../ResolveAllDependencies.java`（+8 / −3）**
`resolveDefaultJavaToolChain()` 里 bundled JDK 厂商 → `AZUL`。
- **可达性**：只在 `resolveAllDependencies` 任务（以及 IDE 的"下载全部依赖"动作）中 `.get()`，不在任何编译/打包路径上。
- **形式说明**：因为上游那两行按规则被注释保留，局部变量 `bundledVendor` 变为未使用（编译只告警，不报错）。

**C5. `gradle/verification-metadata.xml`（+5）——只修 IDEA sync**
给 `org.jetbrains.kotlin:kotlin-stdlib-common:1.9.10` 补一条 sha256。

- **症状**：命令行 `assemble` 全绿，但 IDEA `sync Gradle project` 红：`Dependency verification failed for configuration ':build-conventions:compileClasspath' ... kotlin-stdlib-common-1.9.10.jar ... from repository tencentyun`；HTML 报告给出的确切原因是 **`Checksums are missing from verification metadata`**（缺条目，不是校验和不匹配）。
- **机制**：本文件的策略是 `verify-metadata=false` + `verify-signatures=false`，只靠 `<trusted-artifacts>` 的 regex 放行 `.*-sources[.]jar` / `.*-javadoc[.]jar`，所以任何"没有 sha256 条目、又不匹配 trust regex"的构件都会红。而 `kotlin-stdlib-common` 只出现在 included build `build-conventions` 的 `runtimeClasspath` / `testRuntimeClasspath`（链：`spotless-lib-extra` → `dev.equo.ide:solstice` → `okhttp:4.12.0` → `okio:3.6.0` → `kotlin-stdlib-common:1.9.10`）。命令行编 included build 只解析 `compileClasspath`，不触发；IDEA sync 会把 runtime/test 视图整个解析一遍，于是触发。
- **该构件的性质**：实测 jar 里 **0 个 `.class`、358 个 `.kotlin_metadata`**——Kotlin 多平台的 common 源集声明库，对 JVM 产物零贡献，纯连带依赖。
- **为什么可信**：值 `cde3341ba18a2ba262b0b7cf6c55b20c90e8d434e42c9a13e6a3f770db965a88` 与 tencentyun 公布的 `.sha256` 一致；另外 Maven Central 的 `.jar.sha1` = `dafaf2c27f27c09220cee312df10917d9a5d97ce`，与 Gradle 缓存的内容目录名（目录名就是文件内容 sha1）逐字一致 → 与官方仓库字节相同。
- **闭环验证**：补条目后 IDEA sync `BUILD SUCCESSFUL in 1m 56s`（48 任务，`configureIdeCheckstyle / configureIdeaGradleJvm / buildDependencyArtifacts / enablePreviewFeatures` 全过）。

### 曾经存在、现已回退的一条

**`build-tools-internal/.../docker/DockerSupportService.java`**：Windows 上 Docker Desktop 在 `C:\Program Files\Docker\Docker\resources\bin` 同时放了无扩展名的 bash shim `docker` 和 `docker.exe`，上游按字面名 `docker` 扫 PATH 会命中那个脚本，启动即 `CreateProcess error=193`。当时改成按平台取 `docker.exe`。

- **2026-10-24 实测推翻"它在全绿路径上"**：整文件回退成上游版后跑完整配方 → `BUILD SUCCESSFUL in 17m 17s`，1809 可执行任务（1531 executed / 278 up-to-date），日志 `error=193` 计数 0、`gradle_exit=0`。原因是当前配方已经把 docker 任务全部 `-x` 掉，那个 shim 根本不会被启动。
- 将来要构建 `*DockerImage` / `*-docker-export` 时再按"Windows 取 `docker.exe`"加回。

## 5. 代码之外的本机改动（不属于 git 差异，但会影响构建）

| 动作 | 位置 | 原因 |
| --- | --- | --- |
| 把 `com.google.protobuf:protoc:4.32.0` 的 11 个文件（10 个平台 classifier 的 `.exe` + pom）从 Gradle 缓存平铺拷进 mavenLocal | `D:/01code/apache/maven/repository/com/google/protobuf/protoc/4.32.0/` | `:x-pack:plugin:otel-data:generateProto`、`:x-pack:plugin:prometheus:generateProto` 报 `Could not resolve com.google.protobuf:protoc:4.32.0`，原文是 `Repository tencentyun is disabled due to earlier error ... Connection timed out`（`198.18.x` 是 Clash fake-IP）；**一个仓库被禁用后整个解析就失败，不再尝试后面的仓库**。构件其实早已在 Gradle 缓存里 → 拷进 mavenLocal（它是仓库链第一环）即可。普通 Maven 工程也能直接用这些文件 |
| `git config --global core.longpaths true` | 全局 git 配置 | `:distribution:bwc:major1:createClone` 报 `fatal: cannot create directory ...: Filename too long`。**仓库级 `core.longpaths` 不会被 `git clone` 出的新仓库继承**，所以必须设 global |
| 手工放置仓库根 `branches.json`，并用 `.git/info/exclude` 本地排除 | 仓库根；`.git/info/exclude:7` | 给 A3 的属性提供本地数据源；不是构建产物，也不该提交 |
| 用户级 `org.gradle.java.installations.paths` 指向本机 JDK | `D:/01code/gradle`（`GRADLE_USER_HOME`）下的 `gradle.properties` | 约束 1 的另一半：JDK 只能来自这里 |

## 6. 维护提示

- 编译 esql 之后工作树会出现上千个 `x-pack/plugin/esql/{,compute/}src/main/generated/**` 的 `M`，那是注解处理器写 CRLF 与 `.gitattributes:5`（`*.java text eol=lf`）冲突造成的**行尾 churn**，不是真改动。判据：`git diff --ignore-cr-at-eol --name-only <这些目录>` 输出为空；恢复：`git checkout -- x-pack/plugin/esql/src/main/generated x-pack/plugin/esql/compute/src/main/generated`。
- 报改动规模时必须把"行尾 churn"和"真改动"分开给，否则数字会吓人（1098 vs 13）。
- 交付前自检：`git status --porcelain -uall -- '*.java' | grep '^??'` 必须为空（不允许新增 java 类）。
- 排查"IDEA 构建红但命令行绿"这类问题，入口是 `$GRADLE_USER_HOME/daemon/<version>/daemon-<pid>.out.log`（本机 `D:/01code/gradle/daemon/9.7.1/`）。IDE sync 和命令行构建共用同一个 daemon 日志，要用 `BUILD (SUCCESSFUL|FAILED)` 分段找最近一段；本机的 `%LOCALAPPDATA%/JetBrains/IntelliJIdea*` 下没有 `idea.log`。
