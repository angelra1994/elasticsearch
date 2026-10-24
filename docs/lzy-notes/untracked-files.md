# 本工程中未被 git 跟踪的文件：是什么、谁生成的、能不能删

> 记录时间：2026-10-24。基线：本地分支 `release/v9.5.5`（内容等同 tag `v9.5.5`）。
> 所有数字都是当天下钻实测得到的，命令附在每节里，可复现。

## 1. 一句话结论

工作区里"git 看不到"的东西只有两类：**被 `.gitignore` / `.git/info/exclude` 忽略的**（346 条目录级条目，展开成单个文件是 121947 条）和**未跟踪也未忽略的**（只有 2 个目录、展开 5 个文件：`.workbuddy-ai/` 和 `docs/lzy-notes/`）。
前者的 99.998% 是 Gradle 构建产物和项目级缓存，随时可删；只有 2 个忽略文件是"人放的"（`branches.json`、`checkstyle_ide.xml`），其中 `branches.json` 删了会导致构建失败。

## 2. 复现命令与实测输出

```bash
# 目录级视角（git 把整个被忽略的目录折叠成一条）
git status --porcelain --ignored | grep -c '^!!'        # → 346

# 展开到单个文件
git status --porcelain --ignored -uall | grep -c '^!!'  # → 121947

# 未跟踪且未忽略：折叠视图（git 把整个未跟踪目录折成一条）
git status --porcelain | grep '^??'
# ?? .workbuddy-ai/
# ?? docs/lzy-notes/

# 同一批东西展开到文件是 5 条
git status --porcelain -uall | grep '^??'
# ?? .workbuddy-ai/memory/2026-10-18.md   2026-10-20.md   2026-10-22.md
# ?? docs/lzy-notes/local-build-deviations.md   untracked-files.md
```

346 条目录级条目的分类：

| 类别 | 条数 | 匹配规则 | 产生方式 |
|---|---|---|---|
| 各模块 `build/` | 328 | `.gitignore:54` `build/` | Gradle 任务输出 |
| 根 `build/` | 1 | 同上 | 聚合任务报告目录 |
| `.gradle/` 项目缓存 | 4 | `.gitignore:53` `.gradle/` | Gradle daemon 写的项目本地状态 |
| `.idea/*` | 11 | `.gitignore:23` `.idea/` | IDEA + 本仓库 ide 任务 |
| `checkstyle_ide.xml` | 1 | `.gitignore:91` | `:configureIdeCheckstyle` 生成 |
| `branches.json` | 1 | `.git/info/exclude:7`（非 .gitignore） | 手工放置 |

328 + 1 + 4 + 11 + 1 + 1 = 346。

再钻一层验证"忽略的文件是不是全在构建产物里"：

```bash
git status --porcelain --ignored -uall | grep '^!!' | sed 's/^!! //' \
  | grep -vE '(^|/)build/|(^|/)\.gradle/|^\.idea/'
# branches.json
# checkstyle_ide.xml     ← 121947 条里只有这 2 条在构建目录之外
```

## 3. `*/build/`：Gradle 构建产物（占绝对多数）

329 个 build 目录。把它们的一级子目录名做频次统计，就能反推是哪些任务写的：

```bash
for d in $(git status --porcelain --ignored | grep '^!!' | sed 's/^!! //' | grep '/build/$'); do ls "$d"; done \
  | sort | uniq -c | sort -rn
```

| 次数 | 子目录/文件 | 谁生成的 |
|---|---|---|
| 323 | `tmp/` | 每个任务自己的临时目录（`build/tmp/<taskName>`），jar/compile 的中间 class、inputs snapshot |
| 298 | `distributions/` | ES 自定义 `DistributionTask`（`build-tools` 的 `BasePluginBuildPlugin` / `ElasticsearchDistributionPlugin`），插件 zip/tar、`*-sources.jar`、`*-javadoc.jar`、`.pom` |
| 281 | `generated/` | 注解处理器 / 代码生成任务（esql 的 `compute:ann`+`gen`、protobuf、StringTemplate），下面基本只有 `sources/` |
| 281 | `classes/` | `compileJava` / `compileGroovy` / `compileKotlin` 的 `.class` 输出 |
| 205 | `resources/` | `processResources` / `processTestResources` |
| 167 | `generated-build-info/` | `BuildInfoPlugin`，产出 `<name>-build-info.json`（`build-tools/src/main/java/.../test/TestBuildInfoPlugin`） |
| 166 | `generated-descriptor/` | `BasePluginBuildPlugin.java:152`，产出 `plugin-descriptor.properties` |
| 119 | `explodedBundle/` | `explodedBundlePlugin` 任务（`BasePluginBuildPlugin.java:168-170`，一个 `Sync` 任务），插件的解压形态，供测试集群直接引用 |
| 106 | `notices/` | `licensesAudit` / `DependencyLicensesTask`，生成 `THIRD-PARTY.txt`、`NOTICE.txt` |
| 79 | `docs/` | `javadoc` 任务与 docs 模块的 asciidoc 中间产物 |
| 28 | `libs/` | 纯 `jar` 任务输出（build-conventions、build-tools 等自身） |
| 6 | `restResources/` | `rest-resources-zip` 产物（`client/rest` 的 spec JSON） |
| 4 | `jdk21-foreign-api.jar` | `MrjarPlugin` 分层编译（mrjar source set）在多版本 JDK 下复用同一个 21 的 foreign-api stub |
| 3 | `pluginDescriptors/` | `PluginDescriptorsTask` 校验/复制 `*.describe` |
| 2 | `extracted-protos/`、`extracted-include-protos/` | `protobuf-gradle-plugin` 从依赖 jar 里解出 `.proto`（`qa/vector`、`x-pack/plugin/esql-datasource-parquet-rs`） |
| 2 | `bundledCompatApis/` | `:libs:core` 一类模块的兼容 API bundle |
| 1 | `version.properties`、`testrun/`、`test-results/`、`protos/`、`platform/` | 分别来自 `:build-tools:generateVersionProperties`、测试运行目录、`junitPlatformXmlReport`、protobuf 生成源、`native-libraries:extractLibs` |

体积（实测 `du -sm`，MB）：

```
1595  distribution/build      ← outputs/default 1545 + integ-test-only 50（完整 server 解压镜像，modules/plugins/config/bin）
1549  x-pack/plugin/ml/build  ← ML 的弹库（llama/cpp 依赖、模型库）
 985  .gradle                 ← 见第 4 节
 494  benchmarks/build
 472  server/build
 216  x-pack/plugin/core/build
 203  x-pack/plugin/esql/build
 155  x-pack/plugin/esql-datasource-parquet-rs/build
  60  x-pack/plugin/esql/compute/build
  12  build                   ← 只有 reports/
```

`*/build/distributions/` 下共 772 个文件，按扩展名：jar 520 / zip 175 / pom 73 / tar 4。这些就是"全工程 assemble 全绿"的产物本体。

**能不能删？** 能，代价是下次构建要从头跑（约 6~22 分钟一轮）。`git clean -fdx` 会把它和下面所有东西一起清掉，包括 `branches.json`——别这么干。

## 4. `.gradle/`：项目级（不是用户级）构建状态

```bash
du -sm .gradle/*      # 985 → .gradle/9.7.1
du -sm .gradle/9.7.1/*
# 971  executionHistory   ← 增量构建的输入/输出指纹历史
#  13  fileHashes
#   2  checksums
#   1  buildOutputCleanup / expanded / fileChanges / signature-verification / workspace-id.txt
```

4 个 `.gradle/`：根目录、`build-conventions/`、`build-tools/`、`build-tools-internal/`（后三个是独立构建/Included build 各自的）。内容是 Gradle 9.7.1 的项目本地缓存：任务执行历史、文件哈希、configuration-cache、`workspace-id.txt`。

和 `$GRADLE_USER_HOME`（本机 `D:/01code/gradle`）的区别：那个是跨项目共享的依赖/构建缓存，这个是**本工作区**的增量状态。删了不会丢依赖，只会让所有任务失去 up-to-date 判定——上一轮实测 1809 个任务里 1694 个 up-to-date，就是靠 `executionHistory` 认出来的。

## 5. `.idea/`：一半 IDEA 自己写，一半本仓库的任务写

11 条忽略条目：`.gitignore`（IDEA 自己放的保护文件）、`checkstyle-idea.xml`、`codeStyles/`、`compiler.xml`、`copyright/`、`go.imports.xml`、`gradle.xml`、`misc.xml`、`setting.xml`、`vcs.xml`、`workspace.xml`。

按 mtime 能分出写入者：

```
2026-10-09 07:28  eclipseCodeFormatter.xml / externalDependencies.xml / inspectionProfiles / runConfigurations / scopes   ← clone 自带（已跟踪）
2026-10-09 07:31  .gitignore / codeStyles / copyright / go.imports.xml / setting.xml / vcs.xml                          ← IDEA 首次打开项目时生成
2026-10-23        compiler.xml(1.6MB) / workspace.xml                                                                   ← IDEA 索引与本地状态
2026-10-24 08:50  gradle.xml / misc.xml / checkstyle-idea.xml                                                           ← 本次 sync 的 afterSync 任务改写
```

写入者是本仓库的构建代码 [elasticsearch.ide.gradle](../../build-tools-internal/src/main/groovy/elasticsearch.ide.gradle)（`idea { project { settings { taskTriggers { afterSync ... } } } }`，第 194-199 行）。IDEA sync 成功后会自动跑 4 个任务，正好对应你贴的那段 sync 日志：

| afterSync 任务 | 定义位置 | 写什么 |
|---|---|---|
| `configureIdeCheckstyle` | [:40](../../build-tools-internal/src/main/groovy/elasticsearch.ide.gradle#L40) | 根目录 `checkstyle_ide.xml`（= `build-tools-internal/src/main/resources/checkstyle.xml` 复制后 merge 上 `checkstyle_ide_fragment.xml`，再把 suppressions 路径内联）；同时把模板 `checkstyle-idea.xml` expand 后写进 `.idea/`（`$PROJECT_DIR$` 占位符会被替换，所以故意不入库） |
| `configureIdeaGradleJvm` | [:110](../../build-tools-internal/src/main/groovy/elasticsearch.ide.gradle#L110) | 删掉 `.idea/gradle.xml` 里的 `gradleJvm` 选项，强制 IDEA 用项目 JDK 跑 Gradle |
| `buildDependencyArtifacts` | [:168](../../build-tools-internal/src/main/groovy/elasticsearch.ide.gradle#L168) | 不是写 `.idea`，而是把 IDE 需要的产物先构建出来：`:x-pack:plugin:esql:compute:ann:jar`、`:compute:gen:jar`、`:server:generateModulesList`、`:server:generatePluginsList`、`:generateProviderImpls`、`:libs:native:native-libraries:extractLibs`、`:x-pack:libs:es-opensaml-security-api:shadowJar` → 全部落在第 3 节那些 `build/` 目录里 |
| `enablePreviewFeatures` | [:149](../../build-tools-internal/src/main/groovy/elasticsearch.ide.gradle#L149) | 改 `.idea/modules/**/*.iml` 给 native/server/entitlement 开 JDK21 preview。本机 `.idea/modules` 不存在（IDEA 把模块文件存到了外部），`EnablePreviewFeaturesTask` 找不到 `NewModuleRootManager` 节点就跳过，所以它成功但什么都没改 |

一个值得记的 gitignore 细节：`.gitignore:32-36` 用 `!.idea/eclipseCodeFormatter.xml` 等 5 条想把共享 IDEA 配置"反忽略"，但实际上**无效**——因为第 23 行的 `.idea/` 把父目录整个排除了，Git 不允许重新纳入被排除目录下的文件：

```bash
git check-ignore -v --no-index .idea/eclipseCodeFormatter.xml
# .gitignore:23:.idea/    .idea/eclipseCodeFormatter.xml     ← 仍然匹配，`!` 没生效
```

那 9 个文件之所以还在版本控制里，是因为它们早已被 commit，且 `scopes/Production_minus_fixtures.xml`、`scopes/llrc.xml` 根本不在 `!` 列表里——**跟踪状态优先于 ignore**。结论：想新增共享 IDE 配置必须 `git add -f`。

## 6. 两个"人放的"忽略文件

### `branches.json`（唯一会导致构建失败的）

```bash
git check-ignore -v branches.json
# .git/info/exclude:7:branches.json    branches.json
```

- 387 字节，mtime 2026-10-11 13:22，内容就是官方 `main` 分支 + 各 release 分支的版本映射。
- 忽略通道是 `.git/info/exclude`（仓库本地排除，不随 clone 走、不进版本控制），**不是** `.gitignore`。这样既能让 `git status` 干净，又不用改上游的 `.gitignore`。
- 它是本地偏离 `GlobalBuildInfoPlugin` + `gradle.properties` 的数据源：`org.elasticsearch.build.branches-file-location=branches.json` 把原本要从 `raw.githubusercontent.com` 下载的分支清单指向仓库根，离线/代理挂掉时也能 sync。
- 删掉的后果：`getDevelopmentBranches()` 读不到本地文件 → 联网下载 → 代理不通时整个构建在配置阶段就崩。**不要删。**

### `checkstyle_ide.xml`

纯派生物（`configureIdeCheckstyle` 每次 sync 重写，实测 mtime 08:50 = sync 那一刻）。删了下次 sync 自动回来。

## 7. 未跟踪也未忽略的 2 个目录（展开成文件是 5 个）

| 路径 | 内容 | 来历 |
|---|---|---|
| `.workbuddy-ai/memory/2026-10-18.md`、`2026-10-20.md`、`2026-10-22.md` | 2714 / 9942 / 4550 字节的日记式记录 | 另一个 AI 代理（workbuddy）在本工作区写的记忆文件，不是本仓库构建产物。要不要提交或忽略由你定，目前它既没跟踪也没忽略，会一直出现在 `git status` 里 |
| `docs/lzy-notes/` | 本目录（本学习笔记） | 新建，尚未提交 |

`docs/lzy-notes/` 放在 docs 模块下对构建是惰性的，三条实测依据：
1. [docs/build.gradle:16-17](../../docs/build.gradle#L16) 的 `ext.docsFileTree` 只 `include '**/*.asciidoc'`，`.md` 不进文档构建的输入集合；
2. docs 模块的 precommit 只有 `DependencyLicenses*`，没有 `LicenseHeaders`（`grep -n "licenseHeaders" docs/build.gradle` 无命中），不会因为 md 缺 license 头而失败；且仓库里已有 3278 个 `.md` 是被跟踪的；
3. 建好本目录后实跑任务图，`assemble` 本身是空的、配置阶段无失败、无任何 JDK 下载：

```bash
./gradlew --console=plain :docs:assemble --dry-run -Dtests.bwc.git_fetch_latest=false
# :docs:assemble SKIPPED
# BUILD SUCCESSFUL in 18s
# 18 actionable tasks: 18 up-to-date
# （横幅显示 JDK Version : 21.0.12.1+1-LTS (Azul Zulu)）
```

> `docs/agents.md` 建议改动文档后用仓库根目录的 `docs-builder` 校验链接；本机 PATH 里**没有** `docs-builder`（`which docs-builder` 无结果，它是 elastic 文档工具链的外部命令，不随本仓库提供），所以这里用上面的 gradle dry-run 代替。本笔记目录是个人学习记录，不参与官方文档发布，也不需要链接校验。

## 8. 特殊的一类：被跟踪、但会被构建改写

这类不在"未跟踪"清单里，但它最容易被误当成改动，单独立一节。

esql 的注解处理器（`:x-pack:plugin:esql:compute` 的 gen/ann）会把生成的 java 源码写回**源码树里的被跟踪目录**，不是 `build/`：

```bash
git ls-files x-pack/plugin/esql/src/main/generated | wc -l          # 609
git ls-files x-pack/plugin/esql/compute/src/main/generated | wc -l # 495
```

在 Windows 上它会写出 CRLF，而 `.gitattributes:5` 规定 `*.java text eol=lf`，于是构建完一轮 `git status` 里凭空多出上千个 ` M`——纯行尾差异，`--ignore-cr-at-eol` 一比对就是 0。清理方式（已实测有效，现在 esql 目录 dirty 数为 0）：

```bash
git diff v9.5.5 --ignore-cr-at-eol --name-only | wc -l   # 13，全部是真实改动
git checkout -- x-pack/plugin/esql/src/main/generated x-pack/plugin/esql/compute/src/main/generated
```

另外注意：`.gitignore:92` 忽略的是 `x-pack/plugin/esql/src/main/generated-src/generated/`（另一条生成链的输出），而 `generated-src/` 本身有 20 个**已被跟踪**的文件、当前干净——两者名字像，别混。

## 9. 当前完全没用上的忽略规则

下面这些规则在本机对应路径不存在（逐个 `-e` 实测），列出来是为了以后看到它们时不必怀疑自己漏了什么：
`/logs/`(:65)、`qa/vector/.data`(:101)、`testfixtures_shared/`(:83)、`.artifacts`(:76)、`.vscode/`(:57)、`server/src/main/resources/transport/definitions/manifest.txt`(:95)、`benchmarks/src/main/generated/*`(:39)。
`manifest.txt` 之所以不在，是生成分支清单/transport manifest 的任务没在本次 assemble 路径里跑；`qa/vector/.data` 不在是因为 vector 测试没执行过。

## 10. 速查

```bash
# 看"git 看不到"的全貌（目录级）
git status --porcelain --ignored

# 只看未跟踪且未忽略的
git status --porcelain -uall | grep '^??'

# 判断某个路径被哪条规则忽略（含 .git/info/exclude）
git check-ignore -v <path>

# 只清构建产物、保留 branches.json
rm -rf .gradle */build */*/build x-pack/*/build x-pack/plugin/*/build libs/*/build client/*/build
rm -f checkstyle_ide.xml        # 下次 sync 自动重建

# 千万别用
git clean -fdx                  # 会连 branches.json 一起删掉
```
