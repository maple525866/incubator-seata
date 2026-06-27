## 1. 模块脚手架

- [ ] 1.1 在 `test-suite/` 下新建 `seata-doctor/` 子目录与标准 Maven 布局（`src/main/java`、`src/main/resources`、`src/main/bin`、`src/test/java`、`src/test/resources`），与现有 `test-suite/seata-benchmark-cli` 同级
- [ ] 1.2 新建 `test-suite/seata-doctor/pom.xml`：`<parent>` 指向根 `seata-parent`；`<artifactId>seata-doctor</artifactId>`；`<packaging>jar</packaging>`；继承 `<revision>`；可参照 [test-suite/seata-benchmark-cli/pom.xml](../../../test-suite/seata-benchmark-cli/pom.xml) 的 shade-plugin 与 picocli 声明
- [ ] 1.3 在根 `pom.xml` 的 `<modules>` 列表追加 `test-suite/seata-doctor`（保持注释掉的 `console` / `namingserver` 不动）
- [ ] 1.4 给所有新 `.java` / `.xml` / `.sh` / `.cmd` 文件追加 ASF Apache-2.0 license header（参考任意现有源文件）
- [ ] 1.5 验证 `./mvnw -pl test-suite/seata-doctor -am compile` 在 JDK 8 / 17 全部 BUILD SUCCESS
- [ ] 1.6 验证 `./mvnw spotless:check` 与 `./mvnw license:check -Dlicense.skip=false` 对新模块零告警

## 2. 依赖与 Profile

- [ ] 2.1 `test-suite/seata-doctor/pom.xml` 默认依赖：`org.apache.seata:seata-core`、`seata-common`、`seata-discovery-file`、`seata-discovery-nacos`、`seata-discovery-zk`、`seata-config-core`、`seata-config-nacos`、`seata-config-zk`、`redis.clients:jedis`、`com.mysql:mysql-connector-j`、`info.picocli:picocli`（与 benchmark-cli 同 `4.7.5`）、`com.fasterxml.jackson.core:jackson-databind`（用于 `--json`）
- [ ] 2.2 配置 `org.apache.maven.plugins:maven-shade-plugin`，生成 `seata-doctor-${revision}.jar` 与可执行 `seata-doctor-${revision}-fatjar.jar`，主类 `org.apache.seata.doctor.DoctorApplication`
- [ ] 2.3 新建 Maven profiles：`with-etcd3` / `with-consul` / `with-apollo` / `with-eureka` / `with-redis-registry` / `with-namingserver` / `with-raft` / `with-postgres` / `with-oracle` / `with-sqlserver` / `with-all`，每个 profile 仅声明需要追加的 `<dependency>`
- [ ] 2.4 验证 `./mvnw -pl test-suite/seata-doctor -am package` 默认 fat jar < 30 MB；`-Pwith-all` 总体 < 100 MB
- [ ] 2.5 验证 `./mvnw -pl test-suite/seata-doctor -am package -Pwith-postgres` BUILD SUCCESS 且产出 jar 含 PG 驱动

## 3. CLI 入口与参数

- [ ] 3.1 `org.apache.seata.doctor.DoctorApplication` —— picocli `@Command(name = "seata-doctor", subcommandsRepeatable = false, mixinStandardHelpOptions = true)`，主类持有公共选项：`--server`、`--registry-type`、`--registry-address`、`--registry-group`、`--config-type`、`--config-file`、`--tx-service-group`、`--timeout`（默认 10s）、`--json`、`--exit-code`、`--verbose`
- [ ] 3.2 子命令 `check-all`：依次执行 connect/registry/config/group/store/db/redis（其中 db/redis 仅在用户提供对应连接参数时执行）
- [ ] 3.3 子命令 `check connect`：仅 TC 握手
- [ ] 3.4 子命令 `check registry`：仅注册中心 ping + lookup
- [ ] 3.5 子命令 `check config`：仅配置中心连接 + 5 个关键 key 读取
- [ ] 3.6 子命令 `check group`：组合 mapping → cluster → 实例 → 握手
- [ ] 3.7 子命令 `check store`：读取 `store.mode` / `store.lock.mode` / `store.session.mode` 三元一致性
- [ ] 3.8 子命令 `check db`：要求 `--jdbc-url` `--user` `--password`；探测连接 + schema
- [ ] 3.9 子命令 `check redis`：要求 `--address`；探测 PING + 可选 `--scan-orphans <pattern>`
- [ ] 3.10 子命令 `explain group <txGroup>`：长格式输出（mapping → cluster → 实例列表 → 每个实例握手结果），无机器格式
- [ ] 3.11 单元测试：每个子命令的 `--help` 输出包含必要参数；缺少必填参数时返回 picocli 标准退出码 `2`

## 4. 报告引擎

- [ ] 4.1 `org.apache.seata.doctor.report.Severity`：枚举 `OK` / `WARN` / `FAIL`，每项含 ANSI 色码
- [ ] 4.2 `org.apache.seata.doctor.report.CheckResult`：immutable POJO，字段 `category`、`severity`、`summary`、`details`（List<String>）、`hint`（可空）、`latencyMs`、`timestamp`
- [ ] 4.3 `org.apache.seata.doctor.report.ReportPrinter`：`printHuman(List<CheckResult>)` 彩色对齐输出（Windows 自动降级无色，检测 `System.console() == null` 或 `os.name` 含 "Windows"）；`printJson(List<CheckResult>, Writer)` 用 Jackson；`computeExitCode(List<CheckResult>)` 返回 0/1/2
- [ ] 4.4 `org.apache.seata.doctor.report.HintLibrary`：从 classpath `hints.properties` 加载 `(category, errorCode) → hintText` 映射；提供 `lookup(category, errorCode)` API；缺失时返回 `null`（不抛异常）
- [ ] 4.5 `test-suite/seata-doctor/src/main/resources/hints.properties` —— v1 必备条目（每条对应一个具体 Scenario，下面 spec.md 会列）：
  - `connect.dnsUnresolved`、`connect.refused`、`connect.timeout`、`connect.protocolMismatch`
  - `registry.unreachable`、`registry.authFailed`、`registry.zeroInstances`
  - `config.unreachable`、`config.keyMissing`
  - `group.mappingMissing`、`group.clusterEmpty`
  - `store.modeInconsistent`
  - `db.connectionRefused`、`db.tableMissing.lockTable`、`db.tableMissing.branchTable`、`db.tableMissing.globalTable`、`db.tableMissing.undoLog`
  - `redis.unreachable`、`redis.authFailed`、`redis.permissionInsufficient`
- [ ] 4.6 单元测试：`ReportPrinter` 在不同 severity 组合下退出码正确；`HintLibrary` 缺失键不抛异常

## 5. 各 Check 实现

### 5.1 ConnectCheck（TC Netty 握手）

- [ ] 5.1.1 `org.apache.seata.doctor.check.ConnectCheck`：构造函数注入 `host`、`port`、`timeoutMs`
- [ ] 5.1.2 内部用 Netty `Bootstrap` + Seata 协议 `MessageCodecHandler` / `IdleStateHandler`（直接 import `core` 模块的现有 handler 链）建立连接
- [ ] 5.1.3 握手成功后 `writeAndFlush(new HeartbeatMessage(true))`，等待响应（最多 timeoutMs / 2），记录 RTT
- [ ] 5.1.4 失败分类：`UnknownHostException` → `connect.dnsUnresolved`；`ConnectException` → `connect.refused`；`ReadTimeoutException` → `connect.timeout`；codec 异常 → `connect.protocolMismatch`
- [ ] 5.1.5 必须 `channel.close()` 释放资源，无论成功失败
- [ ] 5.1.6 单元测试：mock Netty handler 验证 5 种分支；用 `ServerSocket` 起一个本地端口验真实连接成功路径
- [ ] 5.1.7 不发送 `RegisterTMRequest` / `RegisterRMRequest` / `GlobalBeginRequest`（在 PR 自检中显式断言 git diff 不含这些类的 import）

### 5.2 RegistryCheck

- [ ] 5.2.1 `org.apache.seata.doctor.check.RegistryCheck`：基于 `--registry-type` 通过 SPI 加载对应 `RegistryService` 实现
- [ ] 5.2.2 调用 `lookup(txServiceGroup)` 获取 `List<InetSocketAddress>`；空列表 → `registry.zeroInstances`；连接异常 → `registry.unreachable` / `registry.authFailed`（按异常分类）
- [ ] 5.2.3 单元测试：mock `RegistryService` 验证 3 种路径；集成测试用 `@EnabledIfSystemProperty(named = "nacosCaseEnabled")` 验真实 Nacos lookup

### 5.3 ConfigCheck

- [ ] 5.3.1 `org.apache.seata.doctor.check.ConfigCheck`：基于 `--config-type` 通过 SPI 加载 `Configuration`
- [ ] 5.3.2 读取 5 个关键 key：`service.vgroupMapping.<group>`、`store.mode`、`store.lock.mode`、`store.session.mode`、`service.disableGlobalTransaction`
- [ ] 5.3.3 任一 key 缺失 → 输出该 key 的 WARN，附 hint `config.keyMissing`
- [ ] 5.3.4 配置中心不可达 → FAIL，附 hint `config.unreachable`
- [ ] 5.3.5 单元测试：mock `Configuration` 验证 5 种 key 缺失组合

### 5.4 GroupCheck（最关键，新人故障 #1）

- [ ] 5.4.1 `org.apache.seata.doctor.check.GroupCheck`：依赖 `Configuration` 与 `RegistryService`
- [ ] 5.4.2 步骤：
  1. `Configuration.getString("service.vgroupMapping." + txGroup)` → 若返回空字符串 → FAIL `group.mappingMissing`
  2. 拿到 `clusterName`，调 `RegistryService.lookup(txGroup)` → 若空列表 → FAIL `group.clusterEmpty`
  3. 对每个返回实例发起 `ConnectCheck` → 任一失败标 WARN 并列出失败实例
- [ ] 5.4.3 单元测试覆盖 3 种路径

### 5.5 StoreCheck

- [ ] 5.5.1 `org.apache.seata.doctor.check.StoreCheck`：读 `store.mode` / `store.lock.mode` / `store.session.mode` 三元
- [ ] 5.5.2 一致性规则（参考 `server/src/main/java/org/apache/seata/server/config/ServerConfig.java` 中的真实校验逻辑）：
  - 三者都为 `db` / `file` / `redis` → OK
  - `store.mode` 设置但 `store.lock.mode` / `store.session.mode` 任一缺失 → WARN（运行时会 fallback，但建议显式配置）
  - 三者出现矛盾组合（如 `store.mode=db` 但 `store.lock.mode=redis`）→ FAIL `store.modeInconsistent`
- [ ] 5.5.3 单元测试：穷举三元真值表的 7 个非全 OK 组合

### 5.6 DbCheck

- [ ] 5.6.1 `org.apache.seata.doctor.check.DbCheck`：构造函数注入 `jdbcUrl`、`user`、`password`、`mode`（`server` / `client`）
- [ ] 5.6.2 用 HikariCP 创建临时 DataSource（`maximumPoolSize=1`），获取连接 → `Class.forName(driverClass)` 显式触发；找不到驱动 → 清晰错误提示 `-Pwith-<dialect>` 或外部 `-cp`
- [ ] 5.6.3 `mode=server`：`DatabaseMetaData.getTables(null, null, "%", new String[]{"TABLE"})` 校验 `lock_table` / `branch_table` / `global_table` 存在；缺失即 FAIL，hint 指向 `script/server/db/<dialect>.sql`
- [ ] 5.6.4 `mode=client`：校验 `undo_log` 存在；缺失即 FAIL，hint 指向 `script/client/at/db/<dialect>.sql`
- [ ] 5.6.5 完成后必须 `dataSource.close()` 释放
- [ ] 5.6.6 单元测试：用 H2 in-memory（`runtime` 范围已经在父 pom 测试依赖里）+ 手工 `CREATE TABLE` 模拟全/部分缺失场景

### 5.7 RedisCheck

- [ ] 5.7.1 `org.apache.seata.doctor.check.RedisCheck`：构造函数注入 `host`、`port`、`password`（可选）、`timeoutMs`
- [ ] 5.7.2 用 Jedis `try-with-resources` 建连，`PING` → 若 `WRONGPASS` → FAIL `redis.authFailed`；连接异常 → FAIL `redis.unreachable`
- [ ] 5.7.3 权限探测：尝试 `SCAN 0 MATCH SEATA_* COUNT 10` —— 若 `NOPERM` → FAIL `redis.permissionInsufficient`
- [ ] 5.7.4 `--scan-orphans <pattern>`：用 `SCAN MATCH <pattern>` 完整迭代（`COUNT 100`，cursor 直到 0），返回所有匹配键；输出格式：行 1 = "Found N matched keys"；行 2..N+1 = key 列表；行 N+2 = "To delete: redis-cli --scan --pattern '<pattern>' | xargs -r -n 100 redis-cli del"
- [ ] 5.7.5 **绝对不发** `DEL` / `UNLINK` / `FLUSHDB` —— 在 PR 自检中显式断言 `RedisCheck.java` git diff 不含这些字符串
- [ ] 5.7.6 单元测试 mock Jedis；集成测试 `@EnabledIfSystemProperty(named = "redisCaseEnabled")`

## 6. 启动脚本与 distribution 集成

- [ ] 6.1 `test-suite/seata-doctor/src/main/bin/seata-doctor.sh`：POSIX `#!/bin/sh`；解析 `JAVA_HOME` / `JAVA_OPTS` / `SEATA_DOCTOR_OPTS`；`exec java $JAVA_OPTS -jar "$BASE_DIR/seata-doctor.jar" "$@"`
- [ ] 6.2 `test-suite/seata-doctor/src/main/bin/seata-doctor.cmd`：Windows 版；首行 `chcp 65001 >nul` 解决中文 hint 在 cmd 下乱码
- [ ] 6.3 在 `distribution/` 下定位现有的 `seata-server` 打包描述符（`maven-assembly-plugin` 的 `release-seata.xml` 或同等 descriptor），追加发行子目录条目，将 doctor 的 fat jar 与 sh/cmd 脚本打包到产物的 `tools/seata-doctor/{bin,lib}/`（**该路径是发行包内的对外目录,与源码所在的 `test-suite/seata-doctor` 解耦**）；产物布局示例：`apache-seata-${revision}/tools/seata-doctor/bin/seata-doctor.sh`、`apache-seata-${revision}/tools/seata-doctor/lib/seata-doctor-${revision}.jar`
- [ ] 6.4 端到端验证：从源码构建 distribution 包 → 解压 → `./tools/seata-doctor/bin/seata-doctor.sh check connect --server 127.0.0.1:8091`（预期：本地未起 server 时 FAIL `connect.refused`，本地起 server 时 OK 且 RTT < 100ms）

## 7. 文档

- [ ] 7.1 `test-suite/seata-doctor/README.md`：包含「Quick Start」「Subcommand reference」「Exit codes」「Profiles」「Troubleshooting」5 个小节；引用根 `script/` 与 `MIGRATION.md` 的相对路径
- [ ] 7.2 顶层 [README.md](../../../README.md) 在 "Components" 或新建 "Operator Tools" 小节链接 doctor（链接指向源码 `test-suite/seata-doctor` 与发行包用法 `tools/seata-doctor/bin/seata-doctor.sh ...` 各一份）
- [ ] 7.3 [AGENTS.md](../../../AGENTS.md) **可选润色**：在现有 "Test scaffolding" 行末尾追加 `, test-suite/seata-doctor`；不强制（详见 design.md Decision 1）
- [ ] 7.4 在 `changes/en-us/<latest-version>.md` 与 `changes/zh-cn/<latest-version>.md` 各加一行：`feature: add seata-doctor CLI for client-side self-diagnosis (#<PR-num>)`
- [ ] 7.5 在 [CONTRIBUTING.md](../../../CONTRIBUTING.md) 的 "Local Development" 小节新增提示："Before reporting a startup failure, please run `./mvnw -pl test-suite/seata-doctor -am package && java -jar test-suite/seata-doctor/target/seata-doctor-*.jar check-all` and attach output."

## 8. CI 集成

- [ ] 8.1 验证 [.github/workflows/build.yml](../../../.github/workflows/build.yml) 默认 `./mvnw -T 4C clean test` 自动覆盖 `test-suite/seata-doctor`（无需修改 workflow，因为根 pom modules 已加该子模块）
- [ ] 8.2 集成测试 IT 类用 `@EnabledIfSystemProperty(named = "redisCaseEnabled" / "nacosCaseEnabled", matches = "true")` 守卫，复用现有 service container；不修改 workflow
- [ ] 8.3 验证 `./mvnw -T 4C clean test -Dpmd.skip=false -Dlicense.skip=false` 对新模块零告警（与 [build/pom.xml](../../../build/pom.xml) 中的 PMD/license 规则对齐）

## 9. Spec 与契约文档

- [ ] 9.1 创建 `openspec/changes/add-seata-doctor-cli/specs/seata-doctor-cli/spec.md`（已与本 tasks.md 同步起草，与 design.md 中的 Decisions 一一映射）
- [ ] 9.2 验证 `openspec status --change add-seata-doctor-cli` 中 `proposal` / `design` / `tasks` / `spec` 全部状态为 `done`

## 10. 收尾

- [ ] 10.1 自我 review proposal/design/tasks/spec 一致性：每个 Decision 都对应一组 Task；每个 Task 都对应至少一条 Scenario；不留 placeholder/TBD
- [ ] 10.2 准备 `git status` / `git diff` 输出，等待用户确认后再 `git add` 与 `git commit`（遵循「未明确要求不主动 commit」原则）
- [ ] 10.3 PR 提交时附上：① 复用了哪些现有 SPI/类（一一列出），证明零侵入；② 默认 fat jar 大小、`-Pwith-all` fat jar 大小；③ `seata-doctor check-all --json` 在本地 demo 环境的真实输出截屏
