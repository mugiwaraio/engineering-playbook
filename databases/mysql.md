# MySQL 研发数据库规范

> 一句话：SQL 必须静态、人类可直接审读；Schema Migration 非必须不做幂等，"只执行一次"由 Migration Runner 保证。

适用范围：生产环境的表设计、SQL 编写、事务与连接管理、账号权限，以及所有 Schema 变更（建表、改表、字段、索引、约束、类型/默认值变更）和数据回填、数据修复 Migration。

## 表设计

1. 引擎与字符集钉死：InnoDB + utf8mb4，全库统一 collation。表间 collation 不一致会导致 JOIN 隐式转换、索引失效。
2. 建表必备项：必须有主键（优先无业务含义的自增 BIGINT）；`created_at` / `updated_at` DATETIME 带默认值；字段优先 NOT NULL + 默认值；表和字段必须写 COMMENT。
3. 类型红线：金额用 DECIMAL，禁止 FLOAT/DOUBLE；日期用日期类型，禁止 VARCHAR 存日期；TEXT/BLOB/大 JSON 慎用，考虑拆表。
4. 禁止物理外键，引用完整性由应用层保证。
5. 命名：表/字段小写下划线；普通索引 `idx_`、唯一索引 `uk_` 前缀；禁用 MySQL 保留字。

## SQL 编写

6. 禁止 `SELECT *`：列增减会导致行为漂移，且浪费回表和带宽。
7. 禁止无 WHERE 条件的 UPDATE / DELETE。
8. 一律参数化查询，禁止字符串拼接 SQL。
9. 防索引失效：WHERE 里不对字段套函数/运算；不让字符串列与数字比较产生隐式转换。
10. 深分页禁止 `LIMIT 1000000, 20`，用游标（`WHERE id > ?`）或延迟关联。
11. 核心 SQL 上线前必须 EXPLAIN，避免全表扫描；禁止循环内逐条查询（N+1）。

## 事务与连接

12. 事务尽量小，禁止在事务内做远程调用或耗时操作。
13. 连接必须走连接池，池大小、获取超时、SQL 执行超时必须显式配置，禁止无限等待。

## 权限与安全

14. 应用账号最小权限：只授 DML，禁 DDL / DROP；每个应用独立账号；禁止 root 直连。
15. 敏感字段（密码、手机号、身份证等）加密或脱敏存储，日志禁打。

## Migration

16. Migration 描述确定的版本迁移：已知 Schema A → 本 Migration → 已知 Schema B。禁止"检查当前状态再决定执行什么"。
17. SQL 必须静态：直接写 `ALTER TABLE ...`。禁止 `information_schema` 判断 + `IF` + `SET @sql` + `PREPARE/EXECUTE` 动态拼接。确实无法静态实现时例外，须注明原因。
18. 禁止自适应幂等：不为"可重复执行"加存在性检查、"存在则跳过"逻辑。"可无限重复执行"不是生产 Migration 的质量目标。
19. 禁止把 `IF EXISTS` / `IF NOT EXISTS` 当防报错机制。仅当"存在与否都是本 Migration 明确定义的合法状态"时可用。
20. 对象"存在"≠"正确"。实际 Schema 与预期不符（Schema Drift）即异常：立即失败、人工调查；禁止静默跳过、自动修复、自动兼容。
21. "是否执行过"由 Runner 判断，不由 SQL 自己判断。维护 `schema_migrations(version, checksum, applied_at)` 记录表。
22. 版本号唯一递增（如 `016_xxx.sql`）。已执行的 Migration 不可修改，出问题新增版本修复；禁止 `016_v2` / `016_final`。
23. Checksum（建议 SHA-256）不一致 → 阻断发布、人工调查；禁止自动更新 checksum 后继续。
24. 执行失败后：先查清哪条 SQL 成功/失败、是否部分执行、`schema_migrations` 是否已记录，再决定处理；禁止直接加 `IF NOT EXISTS` 重跑整个文件。
25. 不得假设 DDL 可以事务回滚——取决于 MySQL 版本、引擎、DDL 类型，未经确认不得宣称"失败可直接 ROLLBACK"。
26. 重要结构变更分阶段：Expand → Migrate → Verify → Contract。不在一个 Migration 里做完所有高风险操作。
27. 大规模数据回填与 DDL 拆成独立 Migration。回填须分批、限速、小事务、可恢复、有进度记录；禁止无边界的大表全量 `UPDATE`。
28. 大表 DDL 先做风险评估：行数、读写 QPS、Metadata Lock、主从延迟、临时/最终磁盘空间、Online DDL 能力、执行窗口，必要时用 Online Schema Change 工具。
29. 索引必须有明确查询依据（WHERE / JOIN / ORDER BY / 选择性），检查等价索引与左前缀重复；删索引先确认无依赖。禁止"SQL 慢就无脑加索引"。
30. 禁止手工直连生产改 Schema。紧急 DDL 须 DBA 参与、留痕（SQL、执行人、时间、结果、影响范围），事后补正式 Migration，保证仓库 Migration 历史 = 生产实际 Schema 演进。
31. 区分两种幂等：Schema Migration 默认不幂等；业务操作（支付回调、消息消费、对账、定时任务、基础数据初始化等）按语义照常设计幂等，如 `ON DUPLICATE KEY UPDATE`。

## 示例

禁止（动态防御式，审核者无法直接看出实际执行内容）：

```sql
SET @column_exists := (
    SELECT COUNT(*) FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'users'
      AND column_name = 'last_login_at'
);
SET @sql := IF(@column_exists = 0,
    'ALTER TABLE users ADD COLUMN last_login_at DATETIME NULL',
    'SELECT 1');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

推荐（静态、可直接人工审计）：

```sql
ALTER TABLE users
    ADD COLUMN last_login_at DATETIME NULL
    COMMENT '用户最后登录时间'
    AFTER updated_at;
```

## AI 生成 / Review 检查清单

```text
[ ] 版本号唯一，前置 Migration 与前置 Schema 明确
[ ] SQL 静态，可直接人工审读
[ ] 无不必要的 information_schema / PREPARE / IF (NOT) EXISTS
[ ] 未修改已执行的 Migration
[ ] 无被静默跳过的 Schema Drift
[ ] 大表 DDL / 大规模回填已给风险提示
[ ] 已考虑滚动发布期间新旧应用兼容
[ ] 失败后的数据库状态明确，有上线后验证方式
```

发现问题 → 明确指出，禁止用幂等逻辑掩盖。

不要问"怎样让这个 SQL 在任何数据库状态下都能执行成功"，应该问"当前数据库是否严格处于本 Migration 要求的前置版本"。答案是否 → STOP，暴露差异，研发 / DBA 调查。
