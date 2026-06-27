## Why

SQL Server JDBC URL 使用 `;` 作为属性分隔符（例如 `jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk`），该字符串会被 `SqlServerResourceIdInitializer` 原样保留为 `resourceId`，进而进入 `AbstractLocker#getRowKey` 拼接出来的 rowKey。然而 Redis 存储模式将 `;` 同时作为 **多个 rowKey 之间的拼接分隔符**（`Constants.ROW_LOCK_KEY_SPLIT_CHAR = ";"`）：在 `RedisLocker` / `RedisLuaLocker` 写入 xid → branch 映射时按 `;` 拼接，在 `releaseLock` / `updateLockStatus` / 控制台 `readGlobalLockByXid` 读取时按 `;` 切分。分隔符与 rowKey 内容字符冲突 → 一个完整 rowKey 在释放阶段被切成两段错误的 key，`DEL` 命中的不是真实锁键，**导致 SQL Server 行锁残留无法释放**，后续事务踩到同一行直接卡死。

## What Changes

- 在 `org.apache.seata.core.lock.AbstractLocker#getRowKey` 末尾对最终字符串执行一次 `replace(ROW_LOCK_KEY_SPLIT_CHAR, LOCK_SPLIT)`，确保单个 rowKey 内部永远不出现 `;`，从根因上消除「拼接 → 拆分」歧义。
- 在 `core` 模块新增 `getRowKey` 含分号场景的单元测试，锁定回归。
- 在 `server` 模块的 Redis 锁测试中新增端到端用例：以含 `;` 的 SQL Server resourceId 走完 `acquireLock → releaseLock` 链，断言真实 lock key 被 `DEL` 命中。
- **不修改** `Constants.ROW_LOCK_KEY_SPLIT_CHAR`、不修改 `SqlServerResourceIdInitializer`、不修改 Redis 释放/状态更新逻辑中按 `;` 切分的代码（根因消除后这些逻辑天然正确）。
- **不影响** DB / File 存储模式：这两种存储把 rowKey 作为整列字符串存取，无 `;` 拆分依赖。
- **非破坏性变更**，但 rowKey 的字符串形态对 SQL Server 数据源会变化（`;` → `^^^`），升级时 Redis 中残留的旧锁键会成为孤儿，需在升级文档提示一次性清理。

## Capabilities

### New Capabilities
- `row-lock-key-encoding`: 定义 Seata 全局事务行锁 rowKey 的字符串构造契约 —— 包括分段分隔符、跨 rowKey 拼接分隔符、字符冲突的脱敏策略，以及该契约对各存储后端（DB / File / Redis）的不变量要求。

### Modified Capabilities
（无：当前 `openspec/specs/` 为空，本次为首次将该规则沉淀为 spec。）

## Impact

- **代码**：
  - `core/src/main/java/org/apache/seata/core/lock/AbstractLocker.java`（核心修复）
  - `core/src/test/java/org/apache/seata/core/lock/AbstractLockerTest.java`（单元测试）
  - `server/src/test/java/org/apache/seata/server/storage/redis/lock/RedisLockerTest.java`（端到端回归测试）
- **不改动但被纳入契约保护**：
  - `server/src/main/java/org/apache/seata/server/storage/redis/lock/RedisLocker.java`
  - `server/src/main/java/org/apache/seata/server/storage/redis/lock/RedisLuaLocker.java`
  - `server/src/main/java/org/apache/seata/server/console/impl/redis/GlobalLockRedisServiceImpl.java`
  - `common/src/main/java/org/apache/seata/common/Constants.java`
  - `rm-datasource/src/main/java/org/apache/seata/rm/datasource/initializer/db/SqlServerResourceIdInitializer.java`
- **API**：无公开 API 变更；`AbstractLocker#getRowKey` 是 `protected` 方法，行为变化对外不可见。
- **依赖**：无新增依赖。
- **运行时数据**：升级前 Redis 中由旧版本写入的、含 `;` 的 SQL Server 行锁键将不再被新代码引用 → 需要运维清理（在 PR/升级文档中提供 `SCAN MATCH SEATA_ROW_LOCK_jdbc:sqlserver*` 的清理建议）。
- **存储后端**：DB / File 模式无行为变化；Redis 模式仅 SQL Server 场景下 rowKey 文本形态发生变化。
