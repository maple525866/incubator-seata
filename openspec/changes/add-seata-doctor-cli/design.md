## Context

Seata 客户端的「启动失败」与「事务卡住」两类故障有一个反复出现的诊断成本结构：根因高度集中在 **6 类配置/连通性环节**（TC 网络、注册中心、配置中心、事务分组映射、`store.mode` 三元一致性、DB/Redis/File 后端可读写性），但定位它们要在 5–6 个工具间反复跳跃。仓库内已经积累了所有原料：

- `discovery/seata-discovery-*` 9 套注册中心 SPI（`RegistryService.lookup`）
- `config/seata-config-*` 6 套配置中心 SPI（`Configuration.getString`）
- `core/` 完整的 Netty + 协议栈（`HeartbeatMessage`、`RpcClientBootstrap`）
- `test-suite/seata-benchmark-cli` 已经把 picocli + Maven shade + Seata SDK 接线打通的 CLI 模板
- 已有 `script/server/db/<dialect>.sql` 与 `script/client/at/db/<dialect>.sql` 等初始化脚本路径——doctor 报错时的 hint 锚点

**约束**：

- **零侵入**：不修改 `core` / `server` / `rm-*` / `tm` / `discovery/*` / `config/*` 任何运行时代码；doctor 只能作为这些模块的只读消费者。
- **JDK 8 floor**：与 [build/pom.xml](../../../build/pom.xml) 中 `source/target = 1.8` 一致；同时要在 CI matrix（JDK 8/17/21/25，见 [.github/workflows/build.yml](../../../.github/workflows/build.yml)）下编译通过。
- **打包体积**：fat jar 默认 < 30MB；全 profile 不超过 100MB，且不裹挟 Apache-2.0 不兼容依赖。
- **行为安全**：v1 严格只读——任何会改变 TC / 注册中心 / 配置中心 / DB / Redis 状态的操作都不允许；产生 server-side ghost session 的协议消息（如 `RegisterTMRequest`、`GlobalBeginRequest`）也不允许。
- **不与 `console` 重叠**：console 是面向运维的 web，做事务查询；doctor 是面向**启动前 / 故障第一现场**的 CLI 体检，两者面向场景不同。
- **`console` / `namingserver` 当前在 [pom.xml](../../../pom.xml) 中被注释**——doctor 不能依赖它们，也不会触发它们的解禁。

利益相关方：

- **新用户**：首次接 Seata 时把 `application.yml`、`registry.conf`、`file.conf`、Nacos 上的 `seataServer.properties` 串起来失败的人群（doctor 给 actionable hint）
- **运维**：线上事务卡住时第一时间需要"快速排除环境问题再深入业务逻辑"的人（doctor 提供 5 秒一键体检）
- **CI/CD 流水线**：K8s readiness probe 与部署后的 smoke test（doctor 提供 `--exit-code` 集成点）
- **Seata 升级链**：例如刚完成的 [`fix-sqlserver-rowlock-leak`](../fix-sqlserver-rowlock-leak/MIGRATION.md) 升级要求扫描 Redis 残留锁键（doctor 把"扫描"标准化）

## Goals / Non-Goals

**Goals:**

- 提供单一可执行入口 `seata-doctor`（jar + sh/cmd），覆盖前述 6 类故障的自助诊断。
- 输出对人友好（彩色对齐表 + `Hint:` 行）、对机器友好（`--json`）、对 CI 友好（`--exit-code`）三种形态。
- **完全只读**：所有 check 在失败/警告路径下都不会改变被检测系统的状态。
- 复用现有 SPI 与协议栈，不写一行重复的 RPC / discovery / config 实现。
- 与 [.github/workflows/build.yml](../../../.github/workflows/build.yml) 的 JDK 矩阵 + spotless / pmd / license 三道质量门兼容。

**Non-Goals:**

- 不取代 `console`：不实现事务列表、全局事务详情、回滚等运维操作。
- 不取代 `seata-benchmark-cli`：不做压测、不做工作负载注入。
- v1 不做"自动修复"：例如不会自动 `DEL` Redis 残留锁、不会自动 `CREATE TABLE undo_log`。
- v1 不做主动写探测（例如不会主动在业务库 `INSERT` 测试行）；只读取 metadata / `SCAN` / `EXISTS`。
- v1 不实现 `--markdown` / `--junit-xml` 输出（YAGNI；按需在 v2 增）。
- 不引入 Testcontainers（项目惯例是 `@EnabledIfSystemProperty` + GitHub Actions service containers，doctor 跟随）。

## Decisions

### Decision 1：模块位置 —— `test-suite/seata-doctor`，与 `seata-benchmark-cli` 镜像

**选择**：把 doctor 模块定位在 `test-suite/seata-doctor`，与现有 `test-suite/seata-benchmark-cli` 形成同级镜像。`pom.xml` 的 `<modules>` 列表追加 `test-suite/seata-doctor`。**不**新建顶级目录，**不**修改 [AGENTS.md](../../../AGENTS.md) 的"Roles in the code"表格（"Test scaffolding | mock-server/, test-suite/..."这一行的 glob 已经覆盖新增子模块）。

**理由**：

- 与 `test-suite/seata-benchmark-cli` 共享相同的工程模式 —— 同为 Seata 客户端 SDK 的 picocli 包装产物（picocli `@Command` + maven-shade + 复用 `core` / `discovery` / `config` 等 SPI），放在一起形成"用户向 CLI"的语义聚簇，评审者熟悉该布局。
- `seata-benchmark-cli` 本身严格来讲也不是"测试 Seata 项目自身的脚手架"（它是面向用户的压测工具），但仓库已经默认接受了这个先例；doctor 跟随同一惯例 → 不引入新的目录层级、不引入新的 AGENTS 表格条目、对仓库布局变更最小。
- 不与 `core` / `server` / `rm-*` 等产品本体平级 → 不让人误以为 doctor 是 RM/TM/TC 的必需依赖。
- `distribution/` 仅有打包脚本与许可证文件，没有 `src/main/java`，不适合放 Java 源码。

**Trade-off（明确写出，便于未来回看）**：

`test-suite/` 这个目录名按 [AGENTS.md](../../../AGENTS.md) 当前措辞偏向"用于测试 Seata 项目自身的脚手架",把面向**终端用户**的 doctor 放进去存在**轻微语义错位**；但实操层面 `seata-benchmark-cli` 已经在该位置先开了这个口子,doctor 跟随的代价小于"为它一个模块新建顶级 `tools/`"的目录变更代价。如果未来再有第三个用户向 CLI（例如配置生成器、trace 分析器），届时考虑统一从 `test-suite/` 抽出到顶级 `tools/`，此次不做此抽象。

**否决方案**：

| 替代方案 | 否决理由 |
|---|---|
| 顶级 `tools/seata-doctor`（新建 `tools/` 目录） | 与 `seata-benchmark-cli` 工程模式相同却被刻意分到两个父目录，造成认知割裂；要顺带改 [AGENTS.md](../../../AGENTS.md)"Roles in the code"表格、改 [distribution/](../../../distribution/) 打包描述符、可能还要改 [.github/workflows/build.yml](../../../.github/workflows/build.yml) 中按目录过滤的步骤；改动面大于收益。 |
| `distribution/seata-doctor`（仅脚本） | doctor 必须复用 Java SDK（discovery/config/core），不能只做 shell 脚本。 |
| 独立仓库 `incubator-seata-doctor` | 释放周期、版本对齐、协议演进脱钩成本高；失去"主仓自带体检"的开箱体验。 |
| `seata-doctor`（顶级模块，平级 `core/`） | 与产品本体平级会让人误以为它是必需依赖；放在 `test-suite/` 与 benchmark-cli 同级更准确。 |

### Decision 2：单 fat jar + profile 化的 SPI 装配

**选择**：`test-suite/seata-doctor/pom.xml` 默认 profile 通过 `maven-shade-plugin` 打成 ~25MB 的可执行 fat jar，**默认包含**：

- `discovery/seata-discovery-{file,nacos,zk}`
- `config/seata-config-{file,nacos,zk}`
- `core` + `common`
- `redis.clients:jedis`（Redis 探测必需）
- `com.mysql:mysql-connector-j`（MySQL 探测，覆盖 MariaDB）
- `info.picocli:picocli`

其余注册/配置中心通过 Maven profile 加载：

| Profile | 加入 |
|---|---|
| `-Pwith-etcd3` | `discovery-etcd3` + `config-etcd3` |
| `-Pwith-consul` | `discovery-consul` + `config-consul` |
| `-Pwith-apollo` | `config-apollo` |
| `-Pwith-eureka` | `discovery-eureka` |
| `-Pwith-redis-registry` | `discovery-redis` |
| `-Pwith-namingserver` | `discovery-namingserver` |
| `-Pwith-raft` | `discovery-raft` |
| `-Pwith-postgres` | `org.postgresql:postgresql` |
| `-Pwith-oracle` | `com.oracle.database.jdbc:ojdbc8` |
| `-Pwith-sqlserver` | `com.microsoft.sqlserver:mssql-jdbc` |
| `-Pwith-all` | 上述全部 |

**理由**：

- 单 jar 是新人最低摩擦的入口（"download → java -jar → 立刻用"）。
- 默认配置覆盖 Seata 用户调研中最普遍的环境（Nacos 注册 + MySQL 业务库），同时控制 jar 体积。
- profile 化避免把所有 native client 一次性塞进一个 jar 导致 ~120MB 的发行包。
- profile 名采用 `with-<x>` 命名贴近 Maven / 用户认知。

**否决方案**：

| 替代方案 | 否决理由 |
|---|---|
| 单全量 fat jar | 体积爆炸；ojdbc / oracle 还会引入 OTN 许可证摩擦。 |
| `ext/` 目录 + `ServiceLoader` 动态加载 | 新人首次使用要先把 jar 放进 `ext/`，违反"零摩擦"目标。 |
| 注册中心 × 配置中心 × 数据库 N×M×K 个 jar | 组合爆炸；CI 矩阵翻倍。 |

### Decision 3：TC 探测 —— 复用 Netty 协议栈，仅发 `HeartbeatMessage`

**选择**：

- 通过 `core/` 内的 Netty bootstrap 建立完整 TCP + 协议握手。
- 握手成功后立即发送一个 `HeartbeatMessage`，等待 server 回包，记录 RTT，然后主动关闭连接。
- **不发** `RegisterTMRequest` / `RegisterRMRequest` / `GlobalBeginRequest`。

**理由**：

- 走完整协议栈 → 自动覆盖 protocol version 不匹配、序列化器不匹配、TLS handshake 失败、Netty 版本兼容性等真实失败模式。
- `HeartbeatMessage` 在 server 端是无副作用的：[server 模块](../../../server) 的 `BatchLogHandler` / `ServerOnRequestProcessor` 对 heartbeat 仅回 ack，不分配 session、不触达事务状态机、不打 metric counter（doctor 不会污染监控）。
- 失败分类原生具备可解释性（DNS unresolved / connection refused / read timeout / handshake protocol error），doctor 可直接照搬异常类型作为 `Hint:` 锚点。

**否决方案**：

| 替代方案 | 否决理由 |
|---|---|
| 仅 `Socket.connect()` 探测 TCP | 漏检 protocol/version skew；这是真实出现过的故障模式（如 v1.x → v2.x 客户端连旧 server）。 |
| `RegisterTMRequest` | 在 server 上注册 ghost client，留下 metric/log 噪声，违反"不污染被检测系统"。 |
| `GlobalBeginRequest` | 创建真实 global session，在 TC 形成孤儿事务，存储后端写入数据。**绝对禁止**。 |
| 自己重写一个最小 Netty client | 重写就要追协议演进；相当于在仓库里复制一份会漂移的 RPC 实现。 |

### Decision 4：v1 严格只读 —— `--scan-orphans` 仅列出，不删除

**选择**：

- 任何 check 都不写入 TC / 注册中心 / 配置中心 / DB / Redis。
- `seata-doctor check redis --scan-orphans <pattern>`：仅执行 `SCAN MATCH <pattern>`，列出匹配键 + 复制粘贴用的 `redis-cli del` 命令片段，**不发 `DEL`**。
- 文档显式声明 v2 才考虑加 `--confirm` 启用销毁性操作。

**理由**：

- doctor 类工具的核心信任契约是"绝不让被检测的系统更糟"。
- 即使 `pattern` 看起来安全，误删 in-flight 事务的真实锁键 = 业务数据一致性风险，代价过高。
- 与 [`fix-sqlserver-rowlock-leak/MIGRATION.md`](../fix-sqlserver-rowlock-leak/MIGRATION.md) 步骤 3 既有运维路径一致：手册给 `awk + xargs + redis-cli del` 一行命令，doctor 帮用户**找到**要删的范围；执行权仍在运维手里。
- v1 收紧 → v2 放开符合"最小可信表面"原则；反过来 v1 放开 → v2 想收紧需要用户教育成本。

**否决方案**：

| 替代方案 | 否决理由 |
|---|---|
| 默认 `--confirm` 即删 | 过于激进，与"零副作用"目标冲突。 |
| 引入 dry-run 默认、`--apply` 启用删除 | 多了一个心智负担的旗标；既然 v1 不需要删，干脆不做。 |
| 让 doctor 自动 `DEL` 但带 `SET NX` 锁 + 备份 | 复杂度暴涨；运维更怕"不可解释的自动操作"。 |

### Decision 5：输出格式三元组 —— 人读 + JSON + exit code

**选择**：

- 默认：彩色对齐表（Linux ANSI 全色，Windows cmd 自动降级到无色），每条 FAIL/WARN 紧跟一行 `Hint:` 锚点。
- `--json`：结构化数组，每个元素含 `category / severity / summary / details / hint / latencyMs / timestamp`。
- `--exit-code`：任一 FAIL 整体退出码 `2`，仅 WARN 退 `1`，全 OK 退 `0`。无该标志时始终退 `0`，便于交互式阅读不被中断。
- v1 不做 `--markdown` / `--junit-xml`（YAGNI）。

**理由**：

- 三元组覆盖三类典型消费方：
  - 人 → 表格
  - 脚本 / Datadog/Prometheus → JSON
  - K8s readiness probe / GitHub Action → exit code
- 退出码语义参考 `shellcheck`、`pylint`：`0=clean`、`1=warn-only`、`2=error`。
- 默认无退出码使新人交互不会因首次出错被 IDE 打断（"为什么我 control-D 了 shell 还退出？"）。

### Decision 6：JDBC 驱动 —— 默认仅打 MySQL，其他通过 profile

**选择**：fat jar 默认仅含 `mysql-connector-j`；其他数据库通过 `-Pwith-postgres` / `-Pwith-oracle` / `-Pwith-sqlserver` profile 打入；用户也可以通过 `java -cp doctor.jar:my-driver.jar org.apache.seata.doctor.DoctorApplication ...` 自带驱动运行。

**理由**：

- Seata 在 [script/server/db/](../../../script/server/db/) 已经覆盖 MySQL/Oracle/PostgreSQL/SQL Server/DM/KingBase/OceanBase/Oscar 八种 dialect，但**实际部署中 MySQL 占比最高**（社区 issue 搜索归纳，非数据指标）；默认 profile 命中最大群体即可。
- 多个 JDBC 驱动 jar 共在同一 ClassLoader 下偶尔引发版本冲突（PG 与 Oracle 老版本最经典）；按需加载更稳。
- Oracle/SQL Server 驱动有 license 复杂性（OTN / Microsoft EULA）；不默认打入避免 license 摩擦。
- doctor 启动时若用户传入 `--jdbc-url jdbc:oracle:...` 但当前 classpath 找不到 Oracle 驱动，给清晰错误并建议 `-Pwith-oracle` 或外部 `-cp`。

**否决方案**：

| 替代方案 | 否决理由 |
|---|---|
| 全打 | jar 体积 + license 复杂。 |
| 全外置（用户自带） | 默认 MySQL 用户也要先准备 driver，违反"零摩擦"。 |
| 用 H2 自带驱动当 fallback | 实际 DB 一般不是 H2，没意义。 |

### Decision 7：测试策略 —— 跟随项目惯例，不引入 Testcontainers

**选择**：

- **单元层**：所有 check 类对外通过构造函数注入 SPI 依赖（`RegistryService` / `Configuration` / `DataSource` / `Jedis`），用 Mockito 模拟行为。
- **集成层**：使用 `@EnabledIfSystemProperty(named = "nacosCaseEnabled" / "redisCaseEnabled", matches = "true")` 守卫，CI 已配置 Nacos / Redis service container 即直接复用。
- **端到端**：在 `test-suite/seata-doctor/src/test/resources/` 放 fixture 配置，用 `picocli.CommandLine` 直接调用 `main()` 验证 CLI 表面。
- **不在 doctor 模块引入** `org.testcontainers:testcontainers`（虽然仓库 `test-suite/seata-benchmark-cli` 与 `discovery/seata-discovery-etcd3` 已经使用 testcontainers，但 doctor 是用户运维侧的诊断工具，希望集成测试本身的运行依赖与 server 模块对齐——不强制要求 Docker daemon）。

**理由**：

- 跟随仓库 server / 锁存储相关模块的现有模式（参见 `server/src/test/java/org/apache/seata/server/lock/redis/RedisLockManagerTest.java` 的 `@EnabledIfSystemProperty` 用法）。
- Mockito + AssertJ 已经在父 [pom.xml](../../../pom.xml) 中预声明，零额外依赖。
- doctor 自身是被诊断系统的"轻量观察者"——其测试也应当反映这一定位：能在 `mvnw clean test` 默认路径下完整跑过，需要 nacos/redis 时通过 system property 显式启用，与现有 CI service container 规约对齐。

## Risks / Trade-offs

- **[Risk] Seata 主版本升级了协议（RPC magic / 序列化器）后，doctor 必须同步发版**
  - Mitigation：doctor 模块绑定父 `<revision>`，与 seata-server 同生命周期发版；CI 在 protocol 变更 PR 中也会编译 doctor，编译失败即被发现。
- **[Risk] nacos / zk 等 native client 启动连接耗时（5–30s 重试）会让 doctor 看起来"卡住"**
  - Mitigation：每个 check 默认 `--timeout 10s`；超时归为 `FAIL` 并在 hint 中提示原因；CLI 持续打印 `Checking <category>...` 进度行避免静默。
- **[Risk] 用户业务库 JDBC 驱动版本与 doctor 默认驱动版本不同，`DatabaseMetaData.getTables()` 行为可能有细微差异**
  - Mitigation：doctor 仅依赖最稳定的 metadata API；对 schema 检查仅断言"表存在"，不断言列类型。极端情况下用户可用 `-cp` 覆盖 driver。
- **[Risk] `--json` 输出在 Windows cmd 下编码混乱（GBK 默认）**
  - Mitigation：JVM 输出强制 UTF-8（`-Dfile.encoding=UTF-8`，由启动脚本注入）；Jackson 序列化 `hint` 字段时启用 `JsonGenerator.Feature.ESCAPE_NON_ASCII`，把非 ASCII 字符 emit 为 `\uXXXX` 转义序列，确保任何终端编码下 stdout 都是 7-bit clean 合法 JSON；`seata-doctor.cmd` 首行 `chcp 65001 >nul` 作为人读模式的保险。
- **[Trade-off] 默认 fat jar 不包含 etcd3 / consul / apollo / eureka 等较小众的注册/配置中心**
  - Accepted：用户首页 README 顶部清单列出所有 profile；选择性体积换可选性。
- **[Trade-off] v1 不做主动写探测（不在业务库 INSERT 测试行）**
  - Accepted：metadata 探测覆盖了 95% 真实启动失败场景；写权限不足在真实事务里会以 `INSERT failed: permission denied` 形式直接报出，不需要 doctor 提前模拟。

## Migration Plan

doctor 是**纯新增模块**，无现存数据迁移问题：

1. **首次发布**：作为 `<revision>` 同步推进的下一个 minor release 的"附属工具"随主包发版；不强绑核心版本号变更。
2. **分发渠道**：
   - 主包：`distribution/` 打包脚本在产物中新增 `tools/seata-doctor/` 子目录（与 `bin/`、`conf/` 同级；该路径**仅指发行包内的对外目录布局**，与源码所在的 `test-suite/seata-doctor` 模块路径无关），用户 `tar -xzf apache-seata-x.y.z.tar.gz` 后即可 `tools/seata-doctor/bin/seata-doctor.sh check-all ...`。
   - 单独 jar：CI 同时上传 `seata-doctor-x.y.z.jar` 到 release artifacts，便于无需主包的轻量场景。
3. **用户感知路径**：
   - 首页 [README.md](../../../README.md) 在"Troubleshooting"或新建"Operator Tools"小节链接 doctor。
   - [changes/en-us/<version>.md](../../../changes/en-us/) 与 zh-cn 镜像各加一行（"feature: add seata-doctor CLI for client-side self-diagnosis"）。
   - 不修改 [AGENTS.md](../../../AGENTS.md)：现有"Test scaffolding | mock-server/, test-suite/test-new-version, test-suite/test-old-version, test-suite/seata-benchmark-cli"已经隐含覆盖 `test-suite/` 下新增子模块；如果团队希望严格列举，可在该行末尾追加 `, test-suite/seata-doctor`，但属于可选润色。
4. **回滚预案**：doctor 是只读工具，不写任何持久化系统；回滚等同于"删除该 release 的 doctor jar"，无数据修复工作量。

## Open Questions（留给后续 spec 与 implementation 阶段进一步澄清，不阻塞 design 落地）

- **OQ-1**：是否需要在 doctor 中预设 SQL 方言到 schema 文件路径的映射表（比如 MySQL → `script/server/db/mysql.sql`），便于 hint 直接引用初始化脚本？倾向：是，作为 `hints.properties` 的一部分。
- **OQ-2**：`explain group` 输出是否包含一张 mermaid 流程图（mapping → cluster → instances → handshake）？倾向：v1 用文本树，v2 加 `--mermaid`。
- **OQ-3**：是否提供 `seata-doctor watch <category> --interval 30s` 长驻模式，便于 K8s liveness 复用？倾向：v2 再考虑，避免 v1 边界扩散。
