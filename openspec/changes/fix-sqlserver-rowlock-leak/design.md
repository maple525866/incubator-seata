## Context

Seata 全局事务的行锁在底层依赖一个三段拼接的 rowKey：`resourceId ^^^ tableName ^^^ pk`，由 `org.apache.seata.core.lock.AbstractLocker#getRowKey` 生成。该字符串被三种存储后端共用：

- **DB 存储**：以整列字符串方式写入 `lock_table.row_key`，不做任何二次切分。
- **File 存储**：同样作为不透明字符串使用。
- **Redis 存储**：除了把 rowKey 作为单个 hash 主键写入，还会把同一分支锁住的多个 rowKey 用常量 `Constants.ROW_LOCK_KEY_SPLIT_CHAR = ";"` 拼接为单个 hash field 值，便于 `releaseLock` / `updateLockStatus` / 控制台 `readGlobalLockByXid` 一次性反查。

`SqlServerResourceIdInitializer` 在生成 SQL Server 数据源的 resourceId 时，会保留 JDBC URL 中的 `;` 与若干属性（例如 `databaseName=lxk`），最终 resourceId 形如 `jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk`。这与 Redis 拼接分隔符 `;` 字面量冲突 —— 单个 rowKey 内部就含有「跨 rowKey 分隔符」，导致写入 Redis 后再读出时按 `;` 切分会把一个完整 rowKey 切成多段错误碎片，`DEL` 命中的不是真实锁键，行锁残留。

约束：

- 不能改动 `Constants.ROW_LOCK_KEY_SPLIT_CHAR` 的值（属于 `common` 接口里的公开常量，跨模块且属于运行时数据契约，改值会让旧 Redis 数据完全不兼容）。
- 不能动 `SqlServerResourceIdInitializer`（其生成的 resourceId 形态对外接口、监控、日志等多处可见）。
- DB / File 模式不能出现行为变化。
- 影响面要尽可能收敛在 1 个改动点上，便于审计与回滚。

利益相关方：使用 Seata + Redis 锁存储 + SQL Server 数据源的用户（直接受益），其它存储 / 数据库组合用户（必须无感）。

## Goals / Non-Goals

**Goals:**
- 单点根因修复：让任何 `getRowKey` 返回的字符串内部都不含 `;`，从源头消除 Redis 「拼接 → 拆分」的歧义。
- 同时治愈 `releaseLock`、`updateLockStatus`、控制台 `readGlobalLockByXid` 三处由同一根因引发的问题。
- 不引入新的运行时依赖、不修改公开 API 与持久化模式。
- 用单元测试 + 端到端 Redis mock 测试锁定回归。

**Non-Goals:**
- 不重构 Redis 存储模式中「多 rowKey 拼成单 hash field」的数据结构（理论上更彻底的方案是改成 Set 或一行一 field，但回归面太大，本次不做）。
- 不为旧 Redis 中残留的孤儿锁键提供自动迁移能力（仅在升级文档中提示一次性清理脚本）。
- 不修改 `SqlServerResourceIdInitializer` 让它去掉 `;`（resourceId 对外可见，改它会污染日志与监控）。
- 不引入额外的编码层（如 URL encode），保持改动最小化。

## Decisions

### Decision 1：在 `AbstractLocker#getRowKey` 末尾对最终字符串做一次 `replace(";", "^^^")` 脱敏

**选择**：对拼接结果整体调用 `replace(ROW_LOCK_KEY_SPLIT_CHAR, LOCK_SPLIT)`，并用 `indexOf(';') >= 0` 作为短路判断减少非 SQL Server 场景的开销。

**理由**：
- rowKey 在整个仓库内**没有任何位置**做 `split("^^^")` 反向解析（已通过全仓搜索确认），是不透明字符串，可以放心改写内部分隔符碰撞字符。
- 整体 replace 同时覆盖三段中可能出现的 `;`（resourceId / tableName / pk），比仅替换 resourceId 更稳健 —— pk 是用户数据，理论上可含任何字符。
- 用 `LOCK_SPLIT = "^^^"` 替换而不是空串或其它字符：保持「分隔风格」一致，便于排障时人眼识别；不会引入额外字符集。
- 复用既有常量 `Constants.ROW_LOCK_KEY_SPLIT_CHAR` 而非裸字符串字面量，语义自明。

**考虑过的替代方案**：

| 替代方案 | 否决理由 |
|---|---|
| **B. 把 `ROW_LOCK_KEY_SPLIT_CHAR` 从 `;` 改成 `\u0001` 之类不可见字符** | 该常量定义在 `common.Constants` 接口上，是公开 API；改值会让所有旧 Redis 数据完全不兼容，且属于跨模块破坏性变更。 |
| **C. 把 Redis 中「多 rowKey 拼为单 field」改成「一行一 field」或 Redis Set** | 改动面横跨 `RedisLocker` / `RedisLuaLocker` / Lua 脚本 / 控制台查询；回归面巨大，超出本 issue 范畴。 |
| **D. 仅替换 resourceId 中的 `;`，再拼接** | 漏掉 pk 含 `;` 的极少数业务场景；同时也意味着脱敏点散落在拼接前的多处，可读性差。 |
| **E. 在 Redis 写入前 URL-encode rowKey、读出后 decode** | 引入双向编码层与 schema 兼容问题，且 DB 模式下不需要、还得做后端区分判断；远复杂于直接消除冲突字符。 |

### Decision 2：保留 `Constants.ROW_LOCK_KEY_SPLIT_CHAR = ";"` 不变

**选择**：常量值不变。

**理由**：
- 跨模块公开常量，盲改风险高。
- 根因被 Decision 1 消除后，单个 rowKey 不再含 `;`，原拼接 / 拆分逻辑天然正确，没有改它的必要。

### Decision 3：保留 `RedisLocker#doReleaseLock` / `updateLockStatus` / `GlobalLockRedisServiceImpl#readGlobalLockByXid` 中按 `;` 切分的代码不变

**选择**：不动这三处「下游消费」逻辑。

**理由**：根因修复后这些逻辑天然正确；改它们等于增加无意义改动面，提高 review 与 regression 风险。

### Decision 4：测试分两层

**选择**：
- **单元层（`core` 模块）**：在 `AbstractLockerTest` 中新增用例，断言含 `;` 的 resourceId 经 `getRowKey` 后输出不含 `;`，并显式验证字符串等于期望形态。
- **集成回归层（`server` 模块）**：在 `RedisLockerTest` 中新增端到端用例，构造含 `;` 的 SQL Server `RowLock`，走完 `acquireLock → releaseLock` 调用链，校验 `Pipeline.del` 的入参是与加锁阶段写入完全一致的单个 key（而不是被切碎的多段）。

**理由**：
- 单元层快、定位精准，验证脱敏契约。
- 集成层复现 issue 现场，证明业务面行为正确，防止有人未来在 `RedisLocker` 端「优化」掉脱敏前提条件。

### Decision 5：升级时残留锁的处理 —— 文档化清理而非自动迁移

**选择**：在 PR 描述与发布说明中提供 Redis 清理脚本指引，不写代码层面的自动数据迁移。

**理由**：
- 受影响数据是「升级瞬间还在 in-flight 的 SQL Server 全局事务」 + 「以前因为本 Bug 残留下来的孤儿锁」。前者数量极少，可通过低峰期升级回避；后者本来就是脏数据。
- 自动迁移涉及扫描 Redis 全库 + 模式匹配，风险与代码量都不成比例。
- 清理脚本足够简洁：`SCAN MATCH SEATA_ROW_LOCK_jdbc:sqlserver*` + 人工/脚本 `DEL`，运维侧成本低。

## Risks / Trade-offs

- **[Risk] 升级前残留 in-flight SQL Server 行锁孤儿化** → Mitigation：在发布说明中要求低峰升级 / 滚动升级前确保事务已提交；提供 Redis 清理脚本。
- **[Risk] 已有自定义代码继承 `AbstractLocker` 并复用 `getRowKey` 假设其结果含原始 `;`** → Mitigation：rowKey 是不透明字符串契约，从未文档化「保留 resourceId 原文」；新 spec 明确写入「rowKey MUST NOT 包含 `;`」契约，作为对未来扩展的指导。属于可接受的契约收紧。
- **[Risk] pk 中含 `;` 的业务场景下，被脱敏后两条不同 pk 可能视觉上更难区分** → Mitigation：极少见；新增的单元测试对 pk 含 `;` 也覆盖；运维排障时按真实 pk 反查即可。
- **[Trade-off] 没有彻底重构 Redis 存储结构** → 接受。本次以「修 Bug」为目标，结构性优化留待后续独立 proposal。

## Migration Plan

1. 合并代码 → 发布带版本号的 release（含 release note 与清理脚本指引）。
2. 用户升级路径：
   - 推荐路径：低峰期停服升级 → 升级后用脚本一次性清理 Redis 中以 `SEATA_ROW_LOCK_jdbc:sqlserver` 开头且 `^^^` 段数不为 3 的孤儿键。
   - 滚动升级路径：先确保所有 SQL Server 全局事务已结束 → 再滚动升级。
3. 回滚预案：单点 revert `AbstractLocker#getRowKey`。回滚后旧版本继续按含 `;` 的 rowKey 工作；本次升级期间新写入的「不含 `;`」rowKey 在回滚后会变成新一批孤儿，按相同清理脚本反向匹配清理（脚本指引中一并给出）。
