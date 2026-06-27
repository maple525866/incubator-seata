## Why

Seata 客户端的"启动失败"和"事务卡住"两类故障，**99% 的根因落在 6 个配置/连通性环节**：TC 网络可达、注册中心可读、配置中心可读、事务分组（`service.vgroupMapping.<group>`）映射存在、`store.mode` 与 `store.lock.mode` / `store.session.mode` 一致、Lock/UndoLog/Session 表 schema 正确。然而当前排障要串起 5–6 种工具：`telnet TC`、`nacos-config.sh`、`zkCli.sh`、业务侧 `application.yml` 比对、手工连 DB 看 `lock_table`、`redis-cli SCAN SEATA_*`。新人启动失败时拿到的报错往往是 `no available service "default"` 或 `can not get cluster name`，**根因不可见**；线上升级（如本仓库刚完成的 [`fix-sqlserver-rowlock-leak`](../fix-sqlserver-rowlock-leak/MIGRATION.md) 升级要求扫描 Redis 残留锁键）也缺一个"官方一行命令"。

业内同类生态都有这种 doctor 类工具：`kubectl cluster-info dump`、`kafka-broker-api-versions.sh`、`pgbench --debug`、`redis-cli --intrinsic-latency`。Seata 缺这一块。

## What Changes

新增独立模块 `test-suite/seata-doctor/`（与现有 `test-suite/seata-benchmark-cli` 镜像，详见 design.md Decision 1），输出可执行 jar + `bin/seata-doctor.{sh,cmd}` 启动脚本，提供如下 CLI：

```
seata-doctor check-all --server 127.0.0.1:8091 --registry nacos --config file
seata-doctor check connect  --server 127.0.0.1:8091
seata-doctor check registry --type nacos --address 127.0.0.1:8848 --group SEATA_GROUP
seata-doctor check config   --type file  --file ./registry.conf
seata-doctor check group    --tx-service-group my_tx_group
seata-doctor check store    # 校验 store.mode / store.lock.mode / store.session.mode 三者一致
seata-doctor check db       --jdbc-url jdbc:mysql://127.0.0.1:3306/seata --user root --password ***
seata-doctor check redis    --address 127.0.0.1:6379 [--scan-orphans 'SEATA_ROW_LOCK_jdbc:sqlserver*']
seata-doctor explain group  my_tx_group   # 一站式：mapping → cluster → 实例列表 → 实测 Netty 握手
```

**输出格式**：彩色对齐表格 + 每条 FAIL/WARN 自动附 `Hint:` 行（指向 `script/` 下的初始化脚本或文档锚点）。`--exit-code` 模式下任一 FAIL 整体返回非零，可用于 K8s readiness probe / CI smoke test。`--json` 模式输出结构化结果便于二次集成。

**实现路径（最大化复用）**：

- **CLI 框架**：复用 [`test-suite/seata-benchmark-cli`](../../../test-suite/seata-benchmark-cli/) 已经接线好的 picocli + Maven shade 模板。
- **TC 连通性**：基于 `core/` 的 `RpcClientBootstrap` 仅发送 `HeartbeatMessage`，验证完整 Netty 握手与协议版本匹配；不发 `GlobalBeginRequest`，避免在 TC 上产生残留 session。
- **注册中心**：通过 SPI 动态加载 `discovery/seata-discovery-*` 的 `RegistryService`，调用 `lookup(txGroup)` 获取实例列表。
- **配置中心**：通过 SPI 动态加载 `config/seata-config-*` 的 `Configuration`，读取 `service.vgroupMapping.<group>`、`store.mode` 等关键 key。
- **事务分组诊断**：组合上述两步 —— 读 mapping 拿到 cluster 名 → 在注册中心 lookup → 对每个返回实例做 Netty 握手。
- **DB 探测**：JDBC + `DatabaseMetaData.getTables()` 校验 `lock_table` / `branch_table` / `global_table` / `undo_log` 是否存在；默认 fat jar 仅打 `mysql-connector-j`（覆盖 MySQL/MariaDB），其他数据库驱动通过 `-Pwith-postgres` / `-Pwith-oracle` / `-Pwith-sqlserver` 等 Maven profile 选择性引入（详见 design.md Decision 6）。
- **Redis 探测**：复用项目已有的 Jedis 依赖；`--scan-orphans` 走 `SCAN MATCH` 匹配，配合 [`fix-sqlserver-rowlock-leak/MIGRATION.md`](../fix-sqlserver-rowlock-leak/MIGRATION.md) 列出可疑残留键。

**模块定位**：`test-suite/seata-doctor`（与 `test-suite/seata-benchmark-cli` 平级），在 `pom.xml` 的 `<modules>` 列表中追加；`distribution/` 增加打包入口，与 `seata-server` 同步发版。注意：发行包内对外暴露的目录布局仍是 `tools/seata-doctor/{bin,lib}/`（与 `seata-server/bin/` 同级），与源码所在模块路径解耦。

**非破坏性 / 可裁剪**：

- 不修改任何 `core/`、`server/`、`rm-*/`、`tm/`、`config/`、`discovery/` 现有代码 —— 仅作为这些模块的**只读消费者**。
- fat jar 拆 profile：默认 profile 仅打 `file + nacos + zk` 三种 registry/config 客户端 + MySQL 探测；`-Pwith-all` 才打全套（避免单 jar 膨胀到 100MB+）。
- `console` / `namingserver` 模块仍按当前 [`pom.xml`](../../../pom.xml) 的注释状态保留，不被本变更触动。

## Capabilities

### New Capabilities

- `seata-doctor-cli`：定义 Seata 客户端侧自助诊断契约 —— 检查项清单（连通性 / 注册中心 / 配置中心 / 事务分组 / 存储模式 / DB schema / Redis 残留）、每项的成功标准与失败提示模板、输出格式（人读 / JSON）、退出码语义、与现有 `script/`、`MIGRATION.md` 的引用约定。

### Modified Capabilities

（无 —— 本变更不修改任何现有 spec 契约。）

## Impact

- **新增代码**（独立模块，零侵入）：
  - `test-suite/seata-doctor/pom.xml`
  - `test-suite/seata-doctor/src/main/java/org/apache/seata/doctor/DoctorApplication.java`（picocli `@Command` 入口）
  - `test-suite/seata-doctor/src/main/java/org/apache/seata/doctor/check/{ConnectCheck,RegistryCheck,ConfigCheck,GroupCheck,StoreCheck,DbCheck,RedisCheck}.java`
  - `test-suite/seata-doctor/src/main/java/org/apache/seata/doctor/report/{CheckReport,Severity,HintLibrary}.java`
  - `test-suite/seata-doctor/src/main/resources/hints.properties`（FAIL/WARN 提示文案，便于 i18n）
  - `test-suite/seata-doctor/src/main/bin/{seata-doctor.sh,seata-doctor.cmd}`
  - `test-suite/seata-doctor/src/test/java/...`（每项 check 的单测 + 用 `@EnabledIfSystemProperty` 守卫的集成测试，复用 CI 现有的 Nacos / Redis service container；详见 design.md Decision 7）
- **修改**：
  - `pom.xml`：`<modules>` 列表追加 `test-suite/seata-doctor`
  - `distribution/`：打包脚本追加 doctor 发行子目录（产物布局：`apache-seata-${revision}/tools/seata-doctor/{bin,lib}/`）
  - 顶层 `README.md` / `CONTRIBUTING.md`：在"故障排查"小节链接 doctor
  - **不修改** `AGENTS.md`：现有"Test scaffolding"行的列表已隐含覆盖 `test-suite/` 下新增子模块；可选地在该行末尾追加 `, test-suite/seata-doctor` 作为润色
- **API**：无 Java API 变更；新增的是面向运维的 CLI surface，按 SemVer 视为新增能力。
- **依赖**：
  - 新增：`info.picocli:picocli`（`test-suite/seata-benchmark-cli/pom.xml` 已声明 `4.7.5`，doctor 模块沿用同版本号；`com.fasterxml.jackson.core:jackson-databind` 用于 `--json` 输出，已在父 [pom.xml](../../../pom.xml) 中声明）。
  - 复用：`discovery/seata-discovery-{file,nacos,zk}`、`config/seata-config-{core,nacos,zk}`、`core`、`common`、`redis.clients:jedis`、`com.zaxxer:HikariCP`、`com.mysql:mysql-connector-j`。默认 fat jar 包含上述全部；其他注册/配置中心与数据库驱动按 `-Pwith-<x>` profile 选打（详见 design.md Decision 2）。
  - **不引入** Apache-2.0 不兼容的新依赖；遵循 [`.licenserc.yaml`](../../../.licenserc.yaml)。
- **运行时数据**：纯只读探测；`--scan-orphans` 仅做 `SCAN`，不做 `DEL`（删除动作仍保留在 [`fix-sqlserver-rowlock-leak/MIGRATION.md`](../fix-sqlserver-rowlock-leak/MIGRATION.md) 的运维流程里，避免工具误删）。
- **存储后端**：DB / File / Redis 三种 store 模式都被覆盖检查；不修改任何 store 后端的运行时行为。
- **测试矩阵**：与项目现有 [.github/workflows/build.yml](../../../.github/workflows/build.yml) 的 JDK 8/17/21/25 矩阵兼容（picocli 4.x 全 JDK 通过）；`-DnacosCaseEnabled=true` / `-DredisCaseEnabled=true` 可选触发集成测试。
- **发布节奏**：建议作为下一个 minor release 的"附属工具"随主包发版，不强绑 `<revision>` 升级。
