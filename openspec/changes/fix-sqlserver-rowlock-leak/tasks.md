## 1. 核心修复（core 模块）

- [x] 1.1 在 `core/src/main/java/org/apache/seata/core/lock/AbstractLocker.java` 顶部新增 `import static org.apache.seata.common.Constants.ROW_LOCK_KEY_SPLIT_CHAR;`
- [x] 1.2 修改 `AbstractLocker#getRowKey(String, String, String)`：在 `StringBuilder.toString()` 之后，若结果含 `;`，则执行 `replace(ROW_LOCK_KEY_SPLIT_CHAR, LOCK_SPLIT)` 后再返回；保留原 javadoc，并补充一段简短注释说明「rowKey 内禁止出现跨 rowKey 拼接分隔符」的不变量来源
- [x] 1.3 验证 `core` 模块编译通过：`mvn -pl core -am test ...` 已成功（含完整编译链路）

## 2. 单元测试（core 模块）

- [x] 2.1 在 `core/src/test/java/org/apache/seata/core/lock/AbstractLockerTest.java` 中新增测试 `testGetRowKeyWithSqlServerResourceId`：输入 `resourceId = "jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk"`、`tableName = "BPM_ACT_RU_TASK"`、`pk = "123"`，断言：① 返回值不含 `;`；② 返回值等于 `jdbc:sqlserver://127.0.0.1:1433^^^databaseName=lxk^^^BPM_ACT_RU_TASK^^^123`
- [x] 2.2 在同一测试类新增 `testGetRowKeyWithSqlServerMultiPropertyUrl`：输入 `resourceId = "jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk;encrypt=false;trustServerCertificate=true"`，断言返回值不含 `;`
- [x] 2.3 在同一测试类新增 `testGetRowKeyWithSemicolonInPk`：输入 `pk = "1001;abnormal"`，断言返回值不含 `;`，且仍能保持对原元组的可识别性（与同输入再次调用幂等）
- [x] 2.4 保留并验证现有 `getRowKey` 测试（`locker.getRowKey("resource1", "table1", "123")`）行为不变 —— 输出仍为 `resource1^^^table1^^^123`，零回归
- [x] 2.5 运行 `mvn -pl core -am test -Dtest=AbstractLockerTest`：**Tests run: 5, Failures: 0, Errors: 0, Skipped: 0**，全部通过

## 3. 端到端回归测试（server 模块）

- [x] 3.1 新增独立测试类 `server/src/test/java/org/apache/seata/server/storage/redis/lock/RedisLockerSqlServerKeyEncodingTest.java`，**不依赖真实 Redis**，CI 默认跑；通过私有 `TestableRedisLocker` 暴露 `protected` 方法 `convertToLockDO` / `buildLockKey`，端到端复用 `RedisLocker` 真实代码路径
- [x] 3.2 用例 `rowKeyOfSqlServerResourceIdMustNotContainSemicolon`：构造 `resourceId = "jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk"` 的 `RowLock`，断言 `LockDO.rowKey` 不含 `;` 且字面量精确匹配
- [x] 3.3 用例 `redisJoinSplitRoundTripPreservesEveryLockKey`：模拟 Redis 中 `String.join(";",...)` → `split(";")` 的真实 round-trip，断言每个 lock key 完整保留 —— 这正是原 Bug 的攻击面
- [x] 3.4 用例 `singleRowKeyOfSqlServerSurvivesSplit`：单行锁场景下 lock key 仍不含 `;`
- [x] 3.5 在 `server/src/test/java/org/apache/seata/server/lock/redis/RedisLockManagerTest.java` 追加 `releaseLockOfSqlServerResourceId`（带 `@EnabledIfSystemProperty("redisCaseEnabled")`）：在有真实 Redis 环境时，验证 `acquire → release → 第二个 xid 重新 acquire` 链路成功（若行锁残留则会失败）
- [x] 3.6 server 模块编译验证通过：`mvn -pl server test-compile` BUILD SUCCESS（按用户决策跳过本机执行测试，由 PR CI 完整验证）

## 4. 跨后端不变量验证

- [x] 4.1 DB / File 存储模式：rowKey 是不透明字符串（`LockStoreDataBaseDAO` 仅 `ps.setString(...)` 整列存取，无 `;` 切分依赖）；行为零变化，无需新增测试
- [x] 4.2 人肉确认 `RedisLocker#doReleaseLock`、`RedisLocker#updateLockStatus`、`GlobalLockRedisServiceImpl#readGlobalLockByXid`、`Constants.ROW_LOCK_KEY_SPLIT_CHAR`、`SqlServerResourceIdInitializer` **无任何修改**（git diff 仅涉及 `AbstractLocker.java` + 测试文件）

## 5. 静态检查与代码质量

- [x] 5.1 对所有改动文件运行 `ReadLints`：无任何新增告警
- [x] 5.2 仅引入 `org.apache.seata.common.Constants.ROW_LOCK_KEY_SPLIT_CHAR` 一项 import；无 unused warning

## 6. 文档与发布说明

- [x] 6.1 已在本 change 目录新增 `MIGRATION.md`：含影响范围说明、Redis 残留 SQL Server 行锁键的扫描/清理脚本（`redis-cli --scan --pattern 'SEATA_ROW_LOCK_jdbc:sqlserver*' | awk -F';' 'NF>1' | xargs -r -n 100 redis-cli del`）、回滚预案
- [ ] 6.2 在 PR 描述中链接本 issue、贴出复现步骤与修复后行为对照（待提交 PR 时填写）

## 7. 收尾

- [x] 7.1 运行 `openspec status --change fix-sqlserver-rowlock-leak`：已确认所有 artifact 为 done
- [ ] 7.2 准备 `git status` / `git diff` 输出，等待用户确认后再提交（遵循「未明确要求不主动 commit」原则）
