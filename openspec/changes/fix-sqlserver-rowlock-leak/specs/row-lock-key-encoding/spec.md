## ADDED Requirements

### Requirement: rowKey 段内分隔符
Seata 行锁的 rowKey SHALL 由 `resourceId`、`tableName`、`pk` 三段拼接组成，段与段之间使用常量 `LOCK_SPLIT = "^^^"` 作为分隔符。该常量 MUST 与跨 rowKey 拼接分隔符不同字符，且不得与任何 JDBC URL 标准属性分隔符冲突。

#### Scenario: 标准 MySQL resourceId 拼接
- **WHEN** 调用 `AbstractLocker#getRowKey("jdbc:mysql://127.0.0.1:3306/seata", "ORDER", "1001")`
- **THEN** 返回字符串 `jdbc:mysql://127.0.0.1:3306/seata^^^ORDER^^^1001`
- **AND** 字符串中 `^^^` 出现且仅出现 2 次

#### Scenario: 复合主键 pk 拼接
- **WHEN** 调用 `AbstractLocker#getRowKey("jdbc:mysql://127.0.0.1:3306/seata", "ORDER_ITEM", "1001_1")`
- **THEN** 返回的 rowKey 中 `pk` 段保留为 `1001_1` 不被改写

### Requirement: rowKey 内必须不含跨 rowKey 拼接分隔符
任何由 `AbstractLocker#getRowKey` 返回的 rowKey 字符串 MUST NOT 包含字符 `Constants.ROW_LOCK_KEY_SPLIT_CHAR`（即 `";"`）。当 `resourceId`、`tableName`、`pk` 中任一段包含 `;` 时，实现 SHALL 在返回前对最终字符串执行字符脱敏，将所有 `;` 替换为 `LOCK_SPLIT`，以保证「多 rowKey 用 `;` 拼接 → 拆分」过程对单个 rowKey 是不可分割的原子单元。

#### Scenario: SQL Server resourceId 含分号
- **WHEN** 调用 `AbstractLocker#getRowKey("jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk", "BPM_ACT_RU_TASK", "123")`
- **THEN** 返回字符串 `jdbc:sqlserver://127.0.0.1:1433^^^databaseName=lxk^^^BPM_ACT_RU_TASK^^^123`
- **AND** 该字符串中不再出现字符 `;`

#### Scenario: pk 中意外携带分号
- **WHEN** 调用 `AbstractLocker#getRowKey("jdbc:mysql://127.0.0.1:3306/seata", "ORDER", "1001;abnormal")`
- **THEN** 返回的 rowKey 中不再出现字符 `;`
- **AND** 替换后字符串依然能唯一对应输入元组 `(resourceId, tableName, pk)`

#### Scenario: SQL Server 多属性 URL
- **WHEN** `resourceId = "jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk;encrypt=false;trustServerCertificate=true"`，调用 `getRowKey` 拼接任意 table 与 pk
- **THEN** 返回的 rowKey MUST NOT 包含字符 `;`

### Requirement: rowKey 唯一性
经过分隔符脱敏后，对于不同的 `(resourceId, tableName, pk)` 输入元组，`getRowKey` 返回值 SHALL 仍然保持一一对应（无哈希碰撞）。允许两个原本含 `;` 的不同 resourceId 替换为 `^^^` 后产生的字符串与某个不含 `;` 的 resourceId 相等，仅当对应的 `(resourceId, tableName, pk)` 元组真的不同 —— 但该情形在生产 JDBC URL 集合中视为不可达。

#### Scenario: 同一输入元组多次调用结果幂等
- **WHEN** 对同一组 `(resourceId, tableName, pk)` 调用 `getRowKey` 两次
- **THEN** 两次返回的字符串完全相等

### Requirement: Redis 存储后端的拼接 / 拆分契约
Redis 存储模式 SHALL 使用 `Constants.ROW_LOCK_KEY_SPLIT_CHAR` 作为「同一分支多个 rowKey 拼接为单个 hash field 值」的分隔符，并在 `releaseLock`、`updateLockStatus`、控制台 `readGlobalLockByXid` 等所有读取场景按同一分隔符拆分。该契约依赖单个 rowKey 内部不含该分隔符（由 rowKey 编码契约保证），实现层 MUST NOT 自行额外引入分号。

#### Scenario: 释放含 SQL Server resourceId 的行锁
- **WHEN** 一个事务分支锁住单条 SQL Server 行（resourceId 含 `;`），其后调用 `RedisLocker#releaseLock(xid, branchId)`
- **THEN** Redis 收到的 `DEL` 调用参数 MUST 是与加锁阶段写入的完整 lock key 字面量一致的单个 key
- **AND** 加锁阶段写入的真实行锁 key 在 Redis 中被删除

#### Scenario: 一个分支多个 rowKey 释放
- **WHEN** 同一分支锁住 N 个行（来自任意数据源），其后调用 `releaseLock`
- **THEN** Redis 收到 N 个 `DEL` key，分别等于加锁阶段每个 rowKey 对应的完整 lock key
- **AND** 不会因拼接 / 拆分操作造成任何 key 被错误切分或合并

### Requirement: 非 Redis 存储后端不受拼接契约约束
DB 与 File 存储后端把 rowKey 当作整段不可分割的字符串读写，不依赖 `ROW_LOCK_KEY_SPLIT_CHAR` 进行序列化。这些后端 SHALL 不受 rowKey 内字符脱敏行为影响，行为保持兼容。

#### Scenario: DB 存储模式中 SQL Server 行锁
- **WHEN** 在 DB 存储模式下，使用含 `;` 的 SQL Server resourceId 加锁、释放
- **THEN** `lock_table.row_key` 列的值与 `getRowKey` 返回值完全一致（含分号脱敏后的形态）
- **AND** 释放阶段按 row_key 的等值匹配能命中相同记录
