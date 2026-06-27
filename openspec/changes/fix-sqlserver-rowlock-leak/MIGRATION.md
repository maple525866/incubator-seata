# Migration Guide — fix-sqlserver-rowlock-leak

## 受影响范围

仅当 **同时满足** 以下两个条件时，需要执行本指南：

1. 你的 Seata Server 使用 **Redis 作为锁存储后端**（`store.lock.mode=redis`）；
2. 至少有一个业务数据源是 **SQL Server**（即注册到 Seata 的 `resourceId` 形如 `jdbc:sqlserver://host:port;property=value;...`）。

如果你只用 DB / File 锁存储，或者完全没有 SQL Server 数据源，本次升级无任何数据迁移工作量。

## 行为变化

修复前：`AbstractLocker#getRowKey` 直接拼接 `resourceId ^^^ tableName ^^^ pk`，输出会保留 SQL Server URL 中的分号 `;`。
修复后：拼接结果中所有 `;` 会被替换为 `^^^`，最终 rowKey 形如 `jdbc:sqlserver://host:port^^^databaseName=xxx^^^TABLE^^^pk`，**不再含 `;`**。

因此 Redis 中行锁键的字面量会从：

```
SEATA_ROW_LOCK_jdbc:sqlserver://127.0.0.1:1433;databaseName=lxk^^^BPM_ACT_RU_TASK^^^123
```

变为：

```
SEATA_ROW_LOCK_jdbc:sqlserver://127.0.0.1:1433^^^databaseName=lxk^^^BPM_ACT_RU_TASK^^^123
```

## 升级前的清理（可选但推荐）

升级前 Redis 中可能残留两类「坏数据」：

1. **由本 Bug 直接产生的孤儿锁**：之前事务正常结束但 `releaseLock` 切坏了 key，真实锁键留在 Redis 没被删；
2. **升级瞬间 in-flight 的 SQL Server 全局事务**：升级后新代码不会再用旧 key 形态去匹配，旧 key 也会成为孤儿。

推荐的处理流程：

### 步骤 1：选择低峰窗口或确保 SQL Server 全局事务已结束

最稳妥：升级前停止业务流量、等待所有进行中的全局事务（含 SQL Server 数据源）提交或回滚完毕。

### 步骤 2：扫描 Redis 中可疑的 SQL Server 行锁键

```bash
# 列出所有 SQL Server 行锁键（按需调整 Redis 连接）
redis-cli --scan --pattern 'SEATA_ROW_LOCK_jdbc:sqlserver*'
```

人工核对返回的 key 形态。**升级前**所有键都应包含 `;`（这是旧版的字面量），**升级后**所有新键都不应包含 `;`。

### 步骤 3：删除孤儿键（升级后执行）

```bash
# 升级 + 启动新版 Seata Server 后执行：
# 删除所有「以旧分号形态」残留的 SQL Server 行锁键
redis-cli --scan --pattern 'SEATA_ROW_LOCK_jdbc:sqlserver*' | \
  awk -F';' 'NF>1' | \
  xargs -r -n 100 redis-cli del
```

> 该命令的语义：扫描所有以 `SEATA_ROW_LOCK_jdbc:sqlserver` 为前缀的键，过滤出键字面量中**仍含分号**的（即旧版残留），批量 `DEL`。
> 升级后由新代码写入的键全部不含 `;`，会被 `awk -F';' 'NF>1'` 过滤掉，**不会被误删**。

如果你的环境同时存在 **多个 Redis 实例**（分片 / 哨兵 / 集群），请对每个实例分别执行。

### 步骤 4：清理 xid 映射 hash 中的过期 field（可选）

`SEATA_GLOBAL_LOCK_<xid>` 这类 xid → branch 映射 hash 中可能也残留了已经无效的 branch field。这些 field 会随对应全局事务结束而被 Seata 自动清理，通常无需手工干预。如需彻底清理：

```bash
# 找出所有 SEATA_GLOBAL_LOCK_ 前缀的 hash，对每个 hash 检查其 field 值是否含「旧分号」格式
redis-cli --scan --pattern 'SEATA_GLOBAL_LOCK_*'
```

人工抽查后再决定是否清理；不建议盲删。

## 回滚预案

如需回滚到修复前的版本：

1. revert `AbstractLocker#getRowKey` 一次提交即可；
2. 升级期间由新版本写入的「不含 `;`」rowKey 会成为新一批孤儿，按 **步骤 3 反向匹配** 清理：

```bash
# 回滚后清理「升级期间产生」的不含分号的 SQL Server 锁键
redis-cli --scan --pattern 'SEATA_ROW_LOCK_jdbc:sqlserver*' | \
  awk -F';' 'NF==1' | \
  xargs -r -n 100 redis-cli del
```

> 注意：回滚操作仅在新版本上线时间很短、新写入数据有限的情况下才推荐执行；正常情况下应优先排查问题再决定是否回滚。

## 不需要做的事

- ❌ 不需要修改 `Constants.ROW_LOCK_KEY_SPLIT_CHAR` 或任何配置项。
- ❌ 不需要修改 `SqlServerResourceIdInitializer` 或业务侧 JDBC URL。
- ❌ 不需要为 DB / File 锁存储模式做任何处理。
- ❌ 不需要重启 RM（业务端）。
