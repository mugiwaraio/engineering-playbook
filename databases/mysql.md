# 生产环境数据库 Migration 规范

## 1. 适用范围

本规范适用于生产环境所有数据库结构变更，包括但不限于：

- `CREATE TABLE`
- `ALTER TABLE`
- `DROP TABLE`
- `ADD COLUMN`
- `MODIFY COLUMN`
- `DROP COLUMN`
- 新增、修改、删除索引
- 唯一约束、外键等约束变更
- Generated Column
- 数据类型变更
- 默认值变更
- Schema 数据回填
- 数据修复 Migration
- 数据库版本升级

AI 在生成、修改、审核生产数据库 Migration 时，必须严格遵守本规范。

---

# 2. 核心原则

生产 Migration 必须描述一个确定的数据库版本迁移过程：

```text
Schema V15
    ↓
Migration 016
    ↓
Schema V16
```

禁止设计成：

```text
未知数据库状态
    ↓
检查当前 Schema
    ↓
根据当前状态决定执行什么
    ↓
存在则跳过
    ↓
最终继续发布
```

生产数据库 Migration 的优先级依次为：

1. 确定性
2. 可审计性
3. Schema 一致性
4. 异常可见性
5. 历史可追踪性
6. 可验证性

核心原则：

> Migration 负责描述数据库应该发生什么变化。

> Migration Runner 负责决定 Migration 是否需要执行。

> 数据库当前状态不符合 Migration 前置条件时，应失败，而不是自动适配。

---

# 3. 生产 Schema Migration 禁止自行实现幂等

生产环境的版本化 Schema Migration 默认禁止为了“可以重复执行”而实现自适应幂等逻辑。

禁止：

```sql
SET @column_exists := (
    SELECT COUNT(*)
    FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'users'
      AND column_name = 'last_login_at'
);

SET @sql := IF(
    @column_exists = 0,
    'ALTER TABLE users ADD COLUMN last_login_at DATETIME NULL',
    'SELECT 1'
);

PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

推荐：

```sql
ALTER TABLE users
    ADD COLUMN last_login_at DATETIME NULL
    COMMENT '用户最后登录时间'
    AFTER updated_at;
```

原因：

- 静态 SQL 更容易阅读。
- 静态 SQL 更容易 Code Review。
- 静态 SQL 更容易 DBA 审计。
- 实际执行内容明确。
- Schema 异常可以及时暴露。
- 不会因为“同名对象存在”而错误认为数据库符合预期。

---

# 4. 禁止隐藏 Schema Drift

Schema Drift 定义：

```text
生产实际 Schema
!=
Migration 预期 Schema
```

Schema Drift 属于异常状态。

例如 Migration 期望：

```sql
ALTER TABLE users
    ADD COLUMN status
        ENUM('active', 'disabled')
        NOT NULL
        DEFAULT 'active';
```

但是生产数据库已经存在：

```sql
status VARCHAR(255)
```

如果 Migration 只检查：

```text
status 字段是否存在？
```

然后：

```text
存在
→ 跳过
```

这是错误行为。

因为：

```text
VARCHAR(255)
!=
ENUM('active', 'disabled')
```

此时 Migration 应该失败，并要求研发或 DBA 调查实际 Schema。

原则：

> 对象“存在”不等于对象“正确”。

---

# 5. 禁止通过 information_schema 自动跳过 DDL

生产 Migration 禁止使用以下模式：

```sql
SELECT COUNT(*)
FROM information_schema.columns
WHERE ...;
```

然后：

```text
存在
→ 跳过

不存在
→ ALTER TABLE
```

禁止大量生成：

```sql
SET @xxx_exists := ...;

SET @xxx_sql := IF(
    @xxx_exists = 0,
    'ALTER TABLE ...',
    'SELECT 1'
);

PREPARE ...;
EXECUTE ...;
DEALLOCATE PREPARE ...;
```

`information_schema` 可以用于：

- 上线前检查
- Schema 验证
- DBA 排查
- Migration 前置条件检查
- 运维诊断

但默认不得用于：

> 根据未知 Schema 状态自动决定是否静默跳过版本化 DDL。

---

# 6. 禁止不必要的动态 DDL

普通 Schema Migration 禁止使用：

```sql
SET @sql = 'ALTER TABLE ...';

PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

应该直接：

```sql
ALTER TABLE ...;
```

动态 SQL 会降低：

- 可读性
- 可审计性
- Code Review 质量
- 静态分析能力
- SQL 风险识别能力
- 执行结果确定性

只有确实无法通过静态 SQL 实现的特殊场景，才允许使用动态 SQL。

使用动态 SQL 时必须明确说明原因。

---

# 7. 禁止滥用 IF EXISTS / IF NOT EXISTS

生产 Schema Migration 不得仅仅为了“重复执行不报错”而自动增加：

```sql
IF EXISTS
```

或者：

```sql
IF NOT EXISTS
```

例如：

```sql
DROP TABLE IF EXISTS old_table;
```

如果按照 Migration 前置版本设计：

```text
old_table 必须存在
```

那么不存在本身就是异常。

此时应该失败，而不是静默继续。

同样：

```sql
ADD COLUMN IF NOT EXISTS status ...
```

如果 Migration 前置版本明确规定：

```text
status 不应该存在
```

那么字段已经存在时应该调查，而不是跳过。

允许使用 `IF EXISTS / IF NOT EXISTS` 的前提：

> 对象存在与不存在本来就是该 Migration 明确定义并经过审核的合法状态。

禁止把它作为通用的“防报错机制”。

---

# 8. Migration 由版本号管理

所有 Migration 必须具有唯一、递增的版本号。

例如：

```text
014_channel_health.sql
015_add_recharge_reconciliation.sql
016_external_channel_health_routing.sql
017_fix_channel_health_projection.sql
```

禁止：

```text
016_v2.sql
016_new.sql
016_final.sql
016_fix.sql
016_final_final.sql
```

Migration 一旦执行：

```text
016
```

后续发现问题必须新增：

```text
017
```

不得修改已经执行的 `016`。

---

# 9. Migration Runner 负责保证只执行一次

Migration SQL 本身不负责判断：

```text
我是不是执行过？
```

该职责必须由 Migration Runner 负责。

建议维护：

```sql
CREATE TABLE schema_migrations (
    version VARCHAR(128) NOT NULL,
    checksum VARCHAR(64) NOT NULL,
    applied_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),

    PRIMARY KEY (version)
);
```

至少记录：

- `version`
- `checksum`
- `applied_at`

可选记录：

- `description`
- `execution_time_ms`
- `operator`
- `application_version`
- `deployment_id`

---

# 10. Migration Runner 执行逻辑

Migration Runner 应执行：

```text
读取 migrations
        ↓
按照 version 排序
        ↓
读取 schema_migrations
        ↓
检查当前 Migration
        ↓
是否已经执行？
   ┌──────────────┴──────────────┐
   是                            否
   ↓                             ↓
校验 checksum                执行 SQL
   ↓                             ↓
是否一致？                   是否成功？
 ┌─────┴─────┐              ┌─────┴─────┐
 是          否              是          否
 ↓           ↓               ↓           ↓
跳过        阻断发布       记录版本      阻断发布
```

Migration 已成功执行后不得再次执行。

---

# 11. 已执行 Migration 必须不可变

Migration 一旦进入生产环境：

> 文件内容必须视为不可变。

禁止修改：

```text
016_external_channel_health_routing.sql
```

应该新增：

```text
017_fix_external_channel_health_routing.sql
```

正确历史：

```text
015
 ↓
016
 ↓
017
 ↓
018
```

错误历史：

```text
016-v1
 ↓
016-v2
 ↓
016-final
 ↓
016-final-final
```

---

# 12. 必须使用 Checksum

每个 Migration 执行后必须保存 checksum。

推荐：

```text
SHA-256
```

例如：

```text
version:
016_external_channel_health_routing

checksum:
a47b7d...
```

每次部署时重新计算文件 checksum。

如果：

```text
当前 Migration 文件 checksum
!=
schema_migrations 中的 checksum
```

必须：

```text
立即阻断发布
```

禁止：

```text
发现 checksum 不一致
→ 自动更新 checksum
→ 继续执行
```

Checksum 不一致意味着：

> 已执行 Migration 可能被修改。

必须人工调查。

---

# 13. Migration SQL 必须人工可审计

推荐：

```sql
ALTER TABLE channel_health_platform_model_states
    ADD COLUMN availability_status
        ENUM('healthy', 'at_risk', 'unavailable')
        NOT NULL
        DEFAULT 'healthy'
        COMMENT '平台模型当前默认路由可用性'
        AFTER enabled,

    ADD COLUMN normal_candidate_count
        INT UNSIGNED NOT NULL
        DEFAULT 0
        COMMENT '当前通过硬过滤的普通候选数量'
        AFTER configured_candidate_count,

    ADD COLUMN last_resort_candidate_count
        INT UNSIGNED NOT NULL
        DEFAULT 0
        COMMENT '当前通过硬过滤的外部兜底候选数量'
        AFTER normal_candidate_count;
```

审核人员应该可以立即确认：

- 修改哪张表
- 新增哪些字段
- 字段类型是什么
- 默认值是什么
- 是否允许 NULL
- 字段位置是什么
- 是否创建索引

禁止要求审核人员追踪大量变量和动态 SQL 才能知道真正执行什么。

---

# 14. Schema 状态异常必须 Fail Fast

如果生产 Schema 不符合 Migration 预期：

```text
立即停止 Migration
```

禁止自动：

- 修复
- 跳过
- 忽略
- 兼容
- 猜测
- 重建
- 修改 Migration 历史

应调查：

1. 当前实际 Schema
2. `schema_migrations`
3. Migration checksum
4. 历史发布记录
5. 是否存在人工 DDL
6. 当前应用版本
7. Migration 是否曾部分执行
8. 主从 Schema 是否一致

原则：

> Schema 状态异常是一条重要的生产故障信号，不应该被 SQL 隐藏。

---

# 15. Migration 执行失败后的处理

Migration 执行失败后，禁止第一时间：

```text
加 IF NOT EXISTS
→ 再跑一次
```

必须先确认：

```text
哪条 SQL 成功？
哪条 SQL 失败？
当前 Schema 是什么？
数据是否已经发生变化？
schema_migrations 是否已经记录？
是否存在部分 DDL 成功？
```

根据结果决定。

## 情况 A：没有产生任何实际变化

可以修复导致失败的问题后重新执行。

## 情况 B：Migration 已部分成功

必须检查当前数据库状态。

然后明确选择：

- 恢复到 Migration 前状态
- 手工完成剩余步骤
- 创建修复 Migration
- 使用 DBA 审核后的恢复方案

禁止：

```text
直接重新执行整个 SQL 文件
```

---

# 16. 不得假设 DDL 一定可以事务回滚

禁止默认认为：

```sql
START TRANSACTION;

ALTER TABLE ...;
ALTER TABLE ...;

ROLLBACK;
```

一定能够恢复所有数据库结构。

MySQL DDL 的行为取决于：

- MySQL 版本
- Storage Engine
- DDL 类型
- Online DDL 能力
- Atomic DDL 支持情况

AI 在生成或审核生产 DDL 时，不得未经确认就宣称：

```text
失败可以直接 ROLLBACK
```

---

# 17. 大型 Schema 变更采用 Expand → Migrate → Contract

对于重要字段替换、结构重构，优先采用：

```text
Expand
   ↓
Migrate
   ↓
Verify
   ↓
Contract
```

例如：

## 第一步：新增字段

```sql
ALTER TABLE users
    ADD COLUMN new_status VARCHAR(32) NULL;
```

## 第二步：应用兼容新旧字段

必要时：

```text
旧字段继续读取
新字段开始写入
```

或者双写。

## 第三步：回填历史数据

```text
旧数据
→ new_status
```

## 第四步：验证

确认：

```text
历史数据全部迁移完成
```

## 第五步：应用停止依赖旧字段

## 第六步：后续 Migration 删除旧字段

不要在一个 Migration 中一次完成所有高风险操作。

---

# 18. 大规模数据回填与 DDL 分离

对于大表，不推荐：

```sql
ALTER TABLE orders
    ADD COLUMN payment_status VARCHAR(32);

UPDATE orders
SET payment_status = ...
WHERE ...;
```

在同一个 Migration 中执行。

推荐：

```text
016_add_payment_status.sql
017_backfill_payment_status.sql
018_enforce_payment_status.sql
```

大规模 Backfill 必须考虑：

- 分批
- 限速
- 小事务
- 可暂停
- 可恢复
- 可继续
- 进度记录
- 数据验证
- 主从延迟
- CPU
- IO
- 锁等待

禁止生成没有边界的大规模生产：

```sql
UPDATE huge_table
SET ...
WHERE ...;
```

而不进行风险提示。

---

# 19. 大表 DDL 必须进行 DBA 风险评估

涉及大表时必须评估：

- 总行数
- 数据大小
- 索引大小
- 写 QPS
- 读 QPS
- 是否热点表
- 主从结构
- 主从延迟风险
- Metadata Lock 风险
- 是否需要重建表
- 临时磁盘空间
- 最终磁盘空间
- Online DDL 能力
- MySQL 版本
- 执行窗口
- 是否需要 Online Schema Change 工具

原则：

> SQL 能执行，不代表 SQL 可以安全地在生产执行。

---

# 20. 索引必须有明确查询依据

新增索引前必须确认：

- 对应哪条 SQL
- `WHERE` 条件
- `JOIN` 条件
- `ORDER BY`
- `GROUP BY`
- 字段顺序
- 字段选择性
- 是否存在等价索引
- 是否存在左前缀重复
- 索引大小
- 写入成本

禁止：

```text
SQL 慢
→ 无脑加索引
```

删除索引前必须确认没有其他查询依赖。

---

# 21. 生产环境禁止常规手工修改 Schema

正常生产 Schema 变更必须经过：

```text
Migration 文件
        ↓
研发 Code Review
        ↓
DBA Review（需要时）
        ↓
CI/CD / Migration Runner
        ↓
生产数据库
```

研发不得正常情况下直接：

```text
登录生产数据库
→ ALTER TABLE
```

---

# 22. 紧急生产 DDL

生产故障确实需要紧急人工修改数据库时：

必须：

1. DBA 参与。
2. 保存完整 SQL。
3. 尽可能双人 Review。
4. 保存执行前 Schema。
5. 记录执行人员。
6. 记录执行时间。
7. 记录执行结果。
8. 记录影响范围。
9. 观察数据库和应用指标。
10. 事后补正式 Migration。

最终必须保证：

```text
代码仓库 Migration 历史
=
生产数据库实际 Schema 演进历史
```

禁止让生产数据库长期存在：

```text
只有生产有
代码仓库不知道
```

的 Schema。

---

# 23. Schema Migration 幂等与业务幂等必须区分

## Schema Migration

默认不追求自幂等。

目标：

```text
已知 Schema A
→ 确定 Migration
→ 已知 Schema B
```

重点是：

- 确定性
- 可审计
- 严格失败
- Schema 正确

## 业务操作

根据业务语义，可以并且通常应该设计成幂等。

例如：

- 支付回调
- 消息消费
- 补偿任务
- 对账任务
- 定时任务
- 数据同步
- API 去重
- 基础数据初始化

允许：

```sql
UPDATE system_config
SET config_value = 'enabled'
WHERE config_key = 'feature_x';
```

或者：

```sql
INSERT INTO system_roles (
    role_code,
    role_name
)
VALUES (
    'admin',
    '管理员'
)
ON DUPLICATE KEY UPDATE
    role_name = VALUES(role_name);
```

因此：

> 禁止的是生产 Schema Migration 通过自适应逻辑隐藏 Schema 异常。

> 不是禁止业务层面的幂等设计。

---

# 24. AI 生成 Migration 时的 MUST 规则

AI 生成生产 Migration 时必须：

- 使用静态 SQL。
- 使用唯一 Migration 版本。
- 默认前置 Migration 已成功执行。
- 基于明确的前置 Schema 生成 SQL。
- 保持 DDL 确定性。
- 保持 SQL 可人工阅读。
- 保持字段定义完整。
- 保持索引定义完整。
- Schema 不符合预期时优先失败。
- 对大表 DDL 给出风险提示。
- 对大规模数据 Backfill 给出风险提示。
- 重要结构变更优先采用分阶段 Migration。
- 考虑滚动发布期间的新旧应用兼容。
- 已执行 Migration 出现问题时创建新版本 Migration。

---

# 25. AI 生成 Migration 时的 MUST NOT 规则

AI 禁止：

- 自动把普通 Migration 改造成幂等 Migration。
- 自动增加 `information_schema` 判断。
- 自动增加“字段存在则跳过”逻辑。
- 自动增加“索引存在则跳过”逻辑。
- 为普通 DDL 自动使用 `PREPARE / EXECUTE`。
- 无理由使用动态 SQL。
- 自动增加 `IF EXISTS`。
- 自动增加 `IF NOT EXISTS`。
- 修改已经执行的 Migration。
- 重复使用已经存在的 Migration 编号。
- 隐藏 Schema Drift。
- 把“可以无限重复执行”作为生产 Migration 的质量目标。
- 假设所有 DDL 都可以事务回滚。
- 无风险提示地生成大表全量 `UPDATE`。
- Schema 异常时自行猜测用户意图并绕过错误。

---

# 26. AI Review Migration 时必须检查

AI 审核生产 Migration 时必须检查：

```text
[ ] Migration version 是否唯一
[ ] 前置 Migration 是否明确
[ ] 前置 Schema 是否明确
[ ] SQL 是否静态
[ ] SQL 是否可以直接人工审计
[ ] 是否存在不必要的 information_schema
[ ] 是否存在不必要的动态 DDL
[ ] 是否存在不必要的 PREPARE / EXECUTE
[ ] 是否使用 IF EXISTS / IF NOT EXISTS 隐藏异常
[ ] 是否修改已经执行的 Migration
[ ] 是否存在 Schema Drift 被静默跳过
[ ] 是否存在大表 DDL
[ ] 是否存在大规模 Backfill
[ ] 是否存在长事务风险
[ ] 是否存在 Metadata Lock 风险
[ ] 索引设计是否合理
[ ] 是否存在重复索引
[ ] 是否考虑应用兼容
[ ] 是否考虑滚动发布
[ ] Migration 失败后的数据库状态是否明确
[ ] 是否提供上线后验证方式
```

如果发现问题：

> AI 应明确指出问题，而不是自动通过增加幂等逻辑掩盖问题。

---

# 27. 推荐 Migration 文件头

推荐每个生产 Migration 包含：

```sql
-- Migration: 016_external_channel_health_routing.sql
--
-- Purpose:
--   增加 Platform Model 可用性投影和 Route Call Statistics。
--
-- Prerequisite:
--   015_add_recharge_reconciliation.sql
--
-- Expected source schema:
--   数据库已成功执行所有 <= 015 的 Migration。
--
-- Execution:
--   只能通过 Migration Runner 执行一次。
--
-- Important:
--   本 Migration 有意不设计成自幂等。
--   如果目标字段、索引或其他 Schema 与预期冲突，
--   必须失败并调查，禁止静默跳过。
```

---

# 28. 推荐 Migration SQL 风格

推荐：

```sql
ALTER TABLE channel_health_runner_heartbeats
    ADD COLUMN runtime_capabilities JSON NOT NULL
        DEFAULT (JSON_OBJECT())
        COMMENT 'Runner运行时能力声明，用于滚动升级安全门'
        AFTER last_error_summary;


ALTER TABLE channel_health_platform_model_states
    ADD COLUMN availability_status
        ENUM('healthy', 'at_risk', 'unavailable')
        NOT NULL
        DEFAULT 'healthy'
        COMMENT '平台模型当前默认路由可用性'
        AFTER enabled,

    ADD COLUMN normal_candidate_count
        INT UNSIGNED NOT NULL
        DEFAULT 0
        COMMENT '当前通过硬过滤的普通候选数量'
        AFTER configured_candidate_count,

    ADD COLUMN last_resort_candidate_count
        INT UNSIGNED NOT NULL
        DEFAULT 0
        COMMENT '当前通过硬过滤的外部兜底候选数量'
        AFTER normal_candidate_count;


UPDATE channel_health_platform_model_states
SET
    availability_status = CASE
        WHEN enabled = TRUE
             AND eligible_candidate_count = 0
            THEN 'unavailable'
        ELSE 'healthy'
    END,
    normal_candidate_count = eligible_candidate_count,
    last_resort_candidate_count = 0;


ALTER TABLE channel_health_platform_model_states
    ADD INDEX idx_channel_health_platform_model_states_availability (
        enabled,
        availability_status,
        updated_at
    );
```

禁止仅仅为了让上述 SQL 可以重复执行，将其改写成：

```text
information_schema
+
IF
+
SET @sql
+
PREPARE
+
EXECUTE
+
SELECT 1
```

---

# 29. 研发、DBA、AI 最终共识

统一遵循以下原则：

> Migration 文件负责描述明确的数据库变更。

> Migration Runner 负责保证 Migration 只执行一次。

> Checksum 负责保证已经执行的 Migration 不被修改。

> 前置 Schema 必须与预期版本一致。

> Schema 状态异常必须明确暴露。

> 禁止生产 Schema Migration 静默适配 Schema Drift。

> 已执行 Migration 不允许修改。

> 历史问题通过新增 Migration 修复。

> 静态、确定、可审计 SQL 优先于动态防御性 SQL。

> Schema Migration 幂等和业务幂等是两个不同问题。

---

# 30. 最简规则

```text
生产 Schema Migration
=
一次执行
+ 静态 SQL
+ 确定性变更
+ 严格失败
+ 历史不可变
+ Checksum 校验
+ Schema Drift 不隐藏
```

AI 在处理生产数据库 Migration 时：

```text
不要问：
“怎样让这个 SQL 在任何数据库状态下都能执行成功？”

应该问：
“当前数据库是否严格处于这个 Migration 所要求的前置版本？”
```

如果答案是否定的：

```text
STOP
→ 暴露差异
→ 研发 / DBA 调查
```

而不是：

```text
自动兼容
→ 静默跳过
→ 继续发布
```

