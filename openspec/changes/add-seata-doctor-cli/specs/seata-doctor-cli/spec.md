## ADDED Requirements

### Requirement: 单一可执行入口

Seata 项目 SHALL 提供一个名为 `seata-doctor` 的独立命令行工具，源码定位于 `test-suite/seata-doctor/` 模块（与 `test-suite/seata-benchmark-cli` 镜像）；构建产物 SHALL 包含一个可执行 fat jar 与对应的启动脚本（`seata-doctor.sh` / `seata-doctor.cmd`）；发行包中的对外目录布局 SHALL 为 `tools/seata-doctor/{bin,lib}/`（与 `seata-server/bin/` 同级，与源码模块路径解耦）；该入口 MUST NOT 依赖 `console` 或 `namingserver` 模块（它们在当前 [pom.xml](../../../../pom.xml) 中处于注释状态）。

#### Scenario: 显示版本与帮助

- **WHEN** 用户执行 `seata-doctor --version`
- **THEN** 输出形如 `seata-doctor <revision>` 且退出码为 `0`

- **WHEN** 用户执行 `seata-doctor --help`
- **THEN** 输出包含全部子命令列表（`check-all` / `check connect` / `check registry` / `check config` / `check group` / `check store` / `check db` / `check redis` / `explain group`）与公共选项

#### Scenario: 无参数时的默认行为

- **WHEN** 用户不带任何子命令直接执行 `seata-doctor`
- **THEN** 输出 usage 信息并退出码 `2`（picocli 标准的 missing-required-subcommand 语义）

### Requirement: TC 连通性检查

`seata-doctor check connect --server <host:port>` 子命令 SHALL 与目标 TC 完成完整的 Netty 协议握手并发送一个 `HeartbeatMessage` 验证协议层畅通；该检查 MUST NOT 发送 `RegisterTMRequest`、`RegisterRMRequest`、`GlobalBeginRequest` 或任何会在 server 端创建持久状态的消息；连接 MUST 在结果输出前主动关闭。

#### Scenario: TC 可达且协议匹配

- **WHEN** 目标 host:port 上运行兼容版本的 Seata Server，且 `--timeout` 内 heartbeat 收到回包
- **THEN** 输出一行 `[ OK ]  TC reachable        <host:port>  (Netty handshake <RTT>ms, protocol v<n>)`
- **AND** 整体退出码（在 `--exit-code` 模式下）为 `0`

#### Scenario: DNS 不可解析

- **WHEN** 目标 host 无法被 DNS 解析（`UnknownHostException`）
- **THEN** 输出 `[FAIL]  TC reachable        <host:port>` 与子行 `↳ DNS lookup failed for <host>` 与 hint 行（指向 hint key `connect.dnsUnresolved`）
- **AND** `--exit-code` 模式下退出码为 `2`

#### Scenario: 连接被拒绝

- **WHEN** 目标 host:port 上没有进程监听（`ConnectException`）
- **THEN** 输出 FAIL 行并附 hint key `connect.refused`（提示用户检查 server 是否启动、端口是否被防火墙拦截）

#### Scenario: 协议版本不匹配

- **WHEN** TCP 建连成功但 Seata Netty handshake 失败（codec 抛 `DecoderException`）
- **THEN** 输出 FAIL 行并附 hint key `connect.protocolMismatch`（提示用户客户端与 server 版本对齐）

#### Scenario: 不污染被检测系统

- **WHEN** 任意 ConnectCheck 完成（无论 OK/FAIL）
- **THEN** TC server 的 metrics（branch/global session counters）值不变
- **AND** TC server 的 transaction-store（DB / File / Redis）中不出现新的 session 记录

### Requirement: 注册中心可读性检查

`seata-doctor check registry --type <type> --address <addr>` 子命令 SHALL 通过 `discovery/seata-discovery-*` 提供的 SPI 加载对应 `RegistryService` 实现，并执行至少一次 `lookup(<txServiceGroup>)`；不支持的 `--type` 取值 MUST 返回非零退出码并提示已支持的类型清单。

#### Scenario: Nacos 注册中心可达且实例存在

- **WHEN** `--type nacos --address 127.0.0.1:8848 --registry-group SEATA_GROUP --tx-service-group default_tx_group`，且 Nacos 上 `default` cluster 注册了 1 个以上 TC 实例
- **THEN** 输出 `[ OK ]  Registry           nacos://127.0.0.1:8848  (group=SEATA_GROUP, instances=N)`

#### Scenario: 注册中心不可达

- **WHEN** Nacos 服务不可达（连接超时或 connection refused）
- **THEN** 输出 FAIL 行并附 hint key `registry.unreachable`

#### Scenario: 集群 0 实例

- **WHEN** Nacos 可达但 `lookup(<txServiceGroup>)` 返回空列表
- **THEN** 输出 FAIL 行并附 hint key `registry.zeroInstances`（提示用户检查 server 是否注册到该 group 与 cluster）

### Requirement: 配置中心可读性检查

`seata-doctor check config --type <type>` 子命令 SHALL 通过 `config/seata-config-*` 提供的 SPI 加载 `Configuration` 实现，并尝试读取以下关键 key：

- `service.vgroupMapping.<txServiceGroup>`
- `store.mode`
- `store.lock.mode`
- `store.session.mode`
- `service.disableGlobalTransaction`

任一 key 缺失 SHALL 单独输出一行 `[WARN]`，不阻断后续 key 的读取。

#### Scenario: file 模式配置加载

- **WHEN** `--type file --config-file ./registry.conf`，且文件存在并合法
- **THEN** 5 个 key 全部读取成功 → 输出 5 行 OK；若有缺失则输出对应 WARN

#### Scenario: 配置中心不可达

- **WHEN** `--type nacos` 但目标 Nacos 不可达
- **THEN** 输出单条 FAIL 行（不再尝试逐 key 读取）并附 hint key `config.unreachable`

#### Scenario: 关键 key 缺失

- **WHEN** Nacos 可达，但 `service.vgroupMapping.<txServiceGroup>` 在配置中心不存在
- **THEN** 输出 `[WARN]  Config key         service.vgroupMapping.<txServiceGroup>  (missing)` 并附 hint key `config.keyMissing`

### Requirement: 事务分组映射检查（新人首要痛点）

`seata-doctor check group --tx-service-group <group>` 子命令 SHALL 组合 ConfigCheck 与 RegistryCheck，按以下顺序执行并在任一步骤失败时立即标 FAIL（但仍输出已完成步骤的结果）：

1. 通过配置中心读取 `service.vgroupMapping.<group>` 得到 `<clusterName>`
2. 通过注册中心 `lookup(<group>)` 得到该 cluster 下的实例列表
3. 对每个实例执行 ConnectCheck（Netty 握手）

#### Scenario: 完整链路通

- **WHEN** mapping 存在、cluster 实例非空、所有实例握手成功
- **THEN** 输出 OK 行汇总：`[ OK ]  TxGroup chain      <group> → <clusterName> → N reachable instances`

#### Scenario: mapping 不存在

- **WHEN** 配置中心读不到 `service.vgroupMapping.<group>`
- **THEN** 输出 FAIL 行并附 hint key `group.mappingMissing`（提示常见错配：`application.yml` 中的 `tx-service-group` 与配置中心实际 key 大小写或分隔符不一致）

#### Scenario: cluster 0 实例

- **WHEN** mapping 存在但 `lookup` 返回空
- **THEN** 输出 FAIL 行并附 hint key `group.clusterEmpty`（提示用户检查 server 启动日志中的 cluster 名）

#### Scenario: 部分实例握手失败

- **WHEN** mapping 存在、cluster 有 N 个实例、其中 K 个 (0 < K < N) 握手失败
- **THEN** 输出 WARN 行 `[WARN]  TxGroup chain      <group> → <clusterName> → <N-K>/N instances reachable`，并列出失败实例的具体错误分类

### Requirement: 存储模式一致性检查

`seata-doctor check store` SHALL 读取配置中心的 `store.mode` / `store.lock.mode` / `store.session.mode` 三个 key，并按 `server/src/main/java/org/apache/seata/server/config/ServerConfig.java` 的真实运行时校验逻辑判断三者是否一致。

#### Scenario: 三元一致

- **WHEN** 三者都为 `db` / `file` / `redis` 中的同一个值
- **THEN** 输出 OK 行：`[ OK ]  Store mode         store.mode=<v>, store.lock.mode=<v>, store.session.mode=<v>`

#### Scenario: 子模式缺失（fallback 场景）

- **WHEN** `store.mode=db`，`store.lock.mode` 与 `store.session.mode` 均未显式设置
- **THEN** 输出 WARN 行：`[WARN]  Store mode         store.lock.mode/store.session.mode unset, will fallback to store.mode=db`（提示用户显式设置避免歧义）

#### Scenario: 三元矛盾

- **WHEN** `store.mode=db` 但 `store.lock.mode=redis`
- **THEN** 输出 FAIL 行并附 hint key `store.modeInconsistent`（提示常见错配：`db` 模式下不应混入 `redis` 子模式，反之亦然）

### Requirement: 数据库 schema 探测

`seata-doctor check db --jdbc-url <url> --user <u> --password <p> --mode <server|client>` 子命令 SHALL 通过 JDBC 建立连接，使用 `DatabaseMetaData.getTables(...)` 校验 schema：

- `--mode server`：必须存在 `lock_table`、`branch_table`、`global_table`
- `--mode client`：必须存在 `undo_log`

驱动加载 SHALL 按 `--jdbc-url` 推断 dialect 并显式 `Class.forName(<driverClass>)`；找不到驱动时 SHALL 给出明确指引（提示 `-Pwith-<dialect>` profile 或 `-cp <driver-jar>`）。该检查 MUST NOT 发起任何 `INSERT` / `UPDATE` / `DELETE` / `CREATE` / `DROP` 语句。

#### Scenario: server 模式 schema 完整

- **WHEN** `--mode server`，目标库存在三张表
- **THEN** 输出 `[ OK ]  DB schema (server) <jdbcUrlSummary>  (lock_table, branch_table, global_table verified)`

#### Scenario: server 模式 lock_table 缺失

- **WHEN** `--mode server`，`global_table` / `branch_table` 存在但 `lock_table` 不存在
- **THEN** 输出 FAIL 行并附 hint key `db.tableMissing.lockTable`，hint 文本中引用 `script/server/db/<dialect>.sql` 路径

#### Scenario: client 模式 undo_log 缺失

- **WHEN** `--mode client`，业务库不存在 `undo_log`
- **THEN** 输出 FAIL 行并附 hint key `db.tableMissing.undoLog`，hint 文本中引用 `script/client/at/db/<dialect>.sql` 路径

#### Scenario: 找不到 JDBC 驱动

- **WHEN** `--jdbc-url jdbc:oracle:thin:@host:1521:orcl` 但当前 classpath 不含 ojdbc
- **THEN** 输出 FAIL 行：`Driver class oracle.jdbc.OracleDriver not found. Use -Pwith-oracle when packaging, or pass driver via java -cp doctor.jar:ojdbc8.jar ...`

#### Scenario: 不发起写操作

- **WHEN** 任何 DbCheck 路径执行完毕（无论 OK/FAIL）
- **THEN** 目标库的 `lock_table` / `branch_table` / `global_table` / `undo_log` 行计数与 ROW_COUNT 不变
- **AND** 数据库的 audit log（如 MySQL general_log）中不出现 `INSERT` / `UPDATE` / `DELETE` / `CREATE` / `DROP` 语句

### Requirement: Redis 探测

`seata-doctor check redis --address <host:port> [--password <p>] [--scan-orphans <pattern>]` 子命令 SHALL 通过 Jedis 建立连接，验证 `PING` 与 `SCAN` 权限；`--scan-orphans` 启用时 SHALL 完整迭代 `SCAN MATCH <pattern>`（COUNT 100，cursor 直到 0）并列出全部匹配键。**禁止**调用 `DEL` / `UNLINK` / `FLUSHDB` / `FLUSHALL` / `EVAL` 等任何写命令。

#### Scenario: Redis 可达且权限充足

- **WHEN** 目标 Redis 可达，`PING` 返回 `PONG`，`SCAN MATCH SEATA_* COUNT 10` 不抛异常
- **THEN** 输出 OK 行 `[ OK ]  Redis             <host:port>  (PING <RTT>ms, SCAN permission verified)`

#### Scenario: 鉴权失败

- **WHEN** 提供的 `--password` 错误（Jedis 抛 `WRONGPASS` 或 `NOAUTH`）
- **THEN** 输出 FAIL 行并附 hint key `redis.authFailed`

#### Scenario: 权限不足

- **WHEN** Redis ACL 禁用了 SCAN（`NOPERM`）
- **THEN** 输出 FAIL 行并附 hint key `redis.permissionInsufficient`，hint 文本提示 ACL 至少需要 `+@read +scan`

#### Scenario: scan-orphans 列出残留锁键

- **WHEN** `--scan-orphans 'SEATA_ROW_LOCK_jdbc:sqlserver*'`，Redis 中存在 3 个匹配键
- **THEN** 输出格式：
  ```
  [ OK ]  Redis scan         pattern=SEATA_ROW_LOCK_jdbc:sqlserver*, matched=3
        ↳ SEATA_ROW_LOCK_jdbc:sqlserver://...;databaseName=lxk^^^TBL^^^pk1
        ↳ SEATA_ROW_LOCK_jdbc:sqlserver://...;databaseName=lxk^^^TBL^^^pk2
        ↳ SEATA_ROW_LOCK_jdbc:sqlserver://...;databaseName=lxk^^^TBL^^^pk3
        ↳ To delete: redis-cli --scan --pattern 'SEATA_ROW_LOCK_jdbc:sqlserver*' | xargs -r -n 100 redis-cli del
  ```

#### Scenario: scan-orphans 严格只读

- **WHEN** 任意 `--scan-orphans` 路径执行完毕
- **THEN** 目标 Redis 实例的 `dbsize` 输出值不变
- **AND** Redis MONITOR 抓包中不出现 `DEL` / `UNLINK` / `FLUSHDB` / `FLUSHALL` / `EVAL`

### Requirement: 报告输出契约

`seata-doctor` 任何子命令 SHALL 支持以下三种输出形态：

- 默认（人读）：彩色对齐表 + 每条 FAIL/WARN 紧跟 `↳ Hint: <hint-text>` 行
- `--json`：标准 JSON 数组，每元素含 `category`、`severity`、`summary`、`details`（数组）、`hint`（可空字符串）、`latencyMs`、`timestamp`（ISO-8601）
- `--exit-code`：影响进程退出码 —— 任一 `FAIL` → `2`，仅 `WARN` → `1`，全 `OK` → `0`；不带该选项时退出码恒为 `0`

#### Scenario: 默认表格在 Linux 终端着色

- **WHEN** 在 Linux/macOS 终端执行 `seata-doctor check connect --server 127.0.0.1:8091`，目标可达
- **THEN** stdout 含 ANSI 转义序列且 `[ OK ]` 前缀以绿色显示

#### Scenario: 默认表格在 Windows cmd 自动降级

- **WHEN** 在 Windows cmd（无 ANSI 支持）执行同命令
- **THEN** stdout 不含 ANSI 转义序列，输出为纯文本

#### Scenario: --json 输出为合法 JSON

- **WHEN** 执行 `seata-doctor check-all ... --json`
- **THEN** stdout 整体为合法 JSON 数组（可被 `jq .` 解析），每元素键集合等于 `{category, severity, summary, details, hint, latencyMs, timestamp}`
- **AND** stderr 不含任何状态行（人读模式的 `Checking ...` 进度行 SHALL 路由到 stderr 或在 `--json` 模式下被抑制）

#### Scenario: --exit-code 区分严重程度

- **WHEN** `check-all --exit-code` 中至少 1 条 FAIL
- **THEN** 进程退出码为 `2`

- **WHEN** `check-all --exit-code` 中无 FAIL 但有至少 1 条 WARN
- **THEN** 进程退出码为 `1`

- **WHEN** `check-all --exit-code` 中全部 OK
- **THEN** 进程退出码为 `0`

### Requirement: 严格只读契约

`seata-doctor` v1 的任何子命令 / 任何参数组合下 MUST NOT 执行以下操作：

- 在 TC server 上发送除 `HeartbeatMessage` 之外的任何业务消息
- 向注册中心 `register` / `unregister` / `update` 任何节点
- 向配置中心 `publish` / `update` / `delete` 任何 key
- 向 DB 发送 `INSERT` / `UPDATE` / `DELETE` / `DDL` 语句
- 向 Redis 发送 `DEL` / `UNLINK` / `FLUSHDB` / `FLUSHALL` / `EVAL` 等写命令

#### Scenario: PR 自检约束

- **WHEN** PR 提交时
- **THEN** `test-suite/seata-doctor/src/main/java/**/*.java` 范围内 git diff MUST NOT 包含上述被禁止的字符串字面量（除注释中显式说明的"为何不发"外）

### Requirement: 模块边界与零侵入

`test-suite/seata-doctor` 模块 MUST NOT 修改 `core/` / `server/` / `rm-*/` / `tm/` / `discovery/*/` / `config/*/` 下任何 Java 文件；仅允许：

- 在 `pom.xml` 的 `<modules>` 列表追加 `test-suite/seata-doctor`
- 在顶层 `README.md` / `CONTRIBUTING.md` / `changes/<lang>/<version>.md` 中追加文档引用（`AGENTS.md` 可选追加，详见 design.md Decision 1）
- 在 `distribution/` 的打包脚本中追加 `tools/seata-doctor/` 发行子目录条目

#### Scenario: PR 自检约束

- **WHEN** PR 提交时
- **THEN** git diff 在 `core/` / `server/` / `rm-*/` / `tm/` / `discovery/` / `config/` 下的 `src/main/java` 路径中变更行数为 `0`
- **AND** `console/` 与 `namingserver/` 路径下变更行数为 `0`
