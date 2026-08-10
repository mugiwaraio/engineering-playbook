# MySQL 研发数据库规范

> 一句话：SQL 必须静态、人类可直接审读；Schema Migration 默认不做幂等，"只执行一次"由 Migration Runner 保证。

适用范围：生产环境的表设计、SQL 编写、事务与连接管理、账号权限，以及所有 Schema 变更（建表、改表、字段、索引、约束、类型/默认值变更）和数据回填、数据修复 Migration。

## 表设计

1. 引擎与字符集钉死：InnoDB + utf8mb4，全库统一 collation。表间 collation 不一致会导致 JOIN 隐式转换、索引失效。
2. 注释强制：所有表必须有表 COMMENT，所有字段必须有字段 COMMENT，写清业务含义；状态/枚举类字段在注释中列出每个取值的含义。
3. 主键：必须有主键，优先无业务含义的自增 BIGINT UNSIGNED；业务唯一键用唯一索引承载，不做主键；禁止随机 UUID 字符串做主键（随机写导致页分裂），分布式 ID 须趋势递增。
4. 通用字段：`created_at` DATETIME 默认 CURRENT_TIMESTAMP，`updated_at` DATETIME 默认 CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP；字段优先 NOT NULL + 默认值。唯一索引列避免 NULL——MySQL 中多个 NULL 不视为唯一冲突。
5. 整型：按取值范围选 TINYINT / SMALLINT / INT / BIGINT，非负加 UNSIGNED；不写显示宽度（用 INT，不用 INT(4)，8.0.17 起显示宽度已废弃）；布尔用 TINYINT，状态字段选型见第 9 条；JOIN / 引用两侧字段的类型与符号必须完全一致。
6. 字符串：VARCHAR(N) 的 N 是字符数不是字节数，按业务实际上限取尽可能小的 N，禁止无脑 VARCHAR(255) / VARCHAR(1024)——磁盘按实际内容占用，但 N 折算字节计入 65535 行上限、受索引键长限制，回退 MEMORY 临时表引擎等内存路径仍按 N 分配；CHAR 仅用于真正定长的值；超长文本用 TEXT 并考虑拆表；禁止 VARCHAR 存日期、存金额。
7. 时间：统一 DATETIME（需要毫秒用 DATETIME(3)）；避免 TIMESTAMP（2038 上限、随会话时区隐式转换）；全库统一时区约定。
8. 金额：DECIMAL 且明确精度标度（如 DECIMAL(18, 2)），或统一用最小货币单位整型（分，BIGINT）；禁止 FLOAT / DOUBLE。
9. 枚举：取值稳定才用 ENUM（增删取值需要 DDL）；会持续扩展的状态用 TINYINT + 应用层常量。选定一种后全库统一，禁止混用。
10. JSON：只存不作为查询条件的弱结构数据；需要查询或索引的键提升为独立列，或用生成列 + 索引。
11. 禁止物理外键，引用完整性由应用层保证。
12. 命名：表名与业务关联；表/字段小写下划线；普通索引 `idx_`、唯一索引 `uk_` 前缀；禁用 MySQL 保留字；测试用表加 `dev_` 前缀与线上表区分。

## SQL 编写

13. 禁止 `SELECT *`：列增减会导致行为漂移，且浪费回表和带宽。
14. 禁止无 WHERE 条件的 UPDATE / DELETE。
15. 一律参数化查询，禁止字符串拼接 SQL；无法参数化的标识符（排序列名、表名等）只能取自代码内白名单映射，禁止透传外部输入。
16. 防索引失效：WHERE 里不对字段套函数/运算；不让字符串列与数字比较产生隐式转换；禁止前导通配符的全模糊匹配 `LIKE '%xxx%'`（无法走索引），确需全文检索用 FULLTEXT 或外部搜索引擎。
17. 深分页禁止 `LIMIT 1000000, 20`，用游标（`WHERE id > ?`）或延迟关联。
18. 核心 SQL 上线前必须 EXPLAIN，避免全表扫描；禁止循环内逐条查询（N+1）；写入同理，批量用多值 INSERT 或分批语句，禁止循环逐条写入。

## 事务与连接

19. 事务尽量小，禁止在事务内做远程调用或耗时操作。
20. 连接必须走连接池，池大小、获取超时、SQL 执行超时必须显式配置，禁止无限等待。
21. 死锁（1213，牺牲事务被自动整体回滚）与锁等待超时（1205，默认仅回滚当前语句）是并发常态：由调用方有限次重试或附上下文上抛，禁止静默吞掉；多行更新按固定顺序访问以减少死锁。

## 权限与安全

22. 应用账号最小权限：只授 DML，禁 DDL / DROP；每个应用独立账号，限定来源主机/网段，禁止 `@'%'`；禁止 root 直连。Migration 由独立的发布账号执行（授 DDL），与应用账号严格分离，仅在发布流程中使用。
23. 密码只存单向哈希（bcrypt / scrypt / argon2），禁止可逆加密或明文；手机号、身份证等其他敏感字段加密或脱敏存储；所有敏感字段日志禁打。

## Migration

24. Migration 描述确定的版本迁移：已知 Schema A → 本 Migration → 已知 Schema B。禁止"检查当前状态再决定执行什么"。
25. SQL 必须静态：直接写 `ALTER TABLE ...`。禁止 `information_schema` 判断 + `IF` + `SET @sql` + `PREPARE/EXECUTE` 动态拼接。确实无法静态实现时例外，须注明原因。
26. 禁止自适应幂等：不为"可重复执行"加存在性检查、"存在则跳过"逻辑。"可无限重复执行"不是生产 Migration 的质量目标。
27. 禁止把 `IF EXISTS` / `IF NOT EXISTS` 当防报错机制。仅当"存在与否都是本 Migration 明确定义的合法状态"时可用。
28. 对象"存在"≠"正确"。实际 Schema 与预期不符（Schema Drift）即异常：立即失败、人工调查；禁止静默跳过、自动修复、自动兼容。
29. "是否执行过"由 Runner 判断，不由 SQL 自己判断。维护 `schema_migrations(version, checksum, applied_at)` 记录表。
30. 版本号唯一递增（如 `016_xxx.sql`）。已执行的 Migration 不可修改，出问题新增版本修复；禁止 `016_v2` / `016_final`。
31. Checksum（建议 SHA-256）不一致 → 阻断发布、人工调查；禁止自动更新 checksum 后继续。
32. 执行失败后：先查清哪条 SQL 成功/失败、是否部分执行、`schema_migrations` 是否已记录，再决定处理；禁止直接加 `IF NOT EXISTS` 重跑整个文件。
33. MySQL 的 DDL 隐式提交，事务内不可 ROLLBACK；8.0 原子 DDL 仅保证单条语句失败/崩溃时不残留半成品，不提供跨语句回滚。
34. 重要结构变更分阶段：Expand → Migrate → Verify → Contract，不在一个 Migration 里做完所有高风险操作；Expand 阶段须保证旧版本应用仍可正常读写，兼容滚动发布期间新旧版本并存。
35. 大规模数据回填与 DDL 拆成独立 Migration。回填须分批、限速（以主从延迟为反馈，超阈值暂停）、小事务、可恢复、有进度记录；数据修复前先将受影响行的主键与原值落备份表，验证通过并过保留期后清理；禁止无边界的大表全量 `UPDATE`。
36. 大表 DDL 先做风险评估：行数、读写 QPS、主从延迟、临时/最终磁盘空间、执行窗口，必要时用 Online Schema Change 工具；Online DDL 显式声明 ALGORITHM / LOCK（如 `ALGORITHM=INPLACE, LOCK=NONE`；INSTANT 只能配 LOCK=DEFAULT），不支持时直接报错，禁止静默降级为锁表 COPY。
37. 执行 DDL 前确认无长事务/慢查询持有目标表 Metadata Lock，并在会话内调小 `lock_wait_timeout`（默认一年）——DDL 等锁期间会阻塞该表所有后续读写，超时报错优于挂住全表。
38. 索引必须有明确查询依据（WHERE / JOIN / ORDER BY / 选择性）；联合索引 `INDEX(p1, p2, p3)` 已等效覆盖 `(p1)`、`(p1, p2)` 左前缀，禁止再建这些重复索引；删索引先确认无依赖。禁止"SQL 慢就无脑加索引"。
39. 禁止手工直连生产改 Schema 或执行数据修复 SQL。紧急 DDL / DML 须 DBA 参与、留痕（SQL、执行人、时间、结果、影响范围），事后补正式 Migration，保证仓库 Migration 历史 = 生产实际 Schema 演进。
40. 区分两种幂等：Schema Migration 默认不幂等；业务操作（支付回调、消息消费、对账、定时任务、基础数据初始化等）按语义照常设计幂等，如 `ON DUPLICATE KEY UPDATE`。

## 附：存储与长度参考（MySQL 8+）

- 单行所有字段总字节长度上限 65535 字节（TEXT / BLOB 等溢出列只计指针）；VARCHAR 的 N 折算成字节后计入该上限。
- 单字符最大字节数：latin1 = 1，gbk = 2，utf8mb3 = 3，utf8mb4 = 4。MySQL 8.0 中 `utf8` 是 utf8mb3 的别名、已弃用，建库建表一律显式 utf8mb4。
- VARCHAR 为变长：长度前缀占 1 或 2 字节，由 N 折算的最大字节数是否超过 255 决定（utf8mb4 下 N > 63 即 2 字节），与实际存入内容无关。VARCHAR(100) 与 VARCHAR(20) 存 `'abc'` 的数据部分占用相同，差别在 N 计入行上限与内存路径分配（见第 6 条）。
- CHAR 为定长：上限 255 字符，不足部分用空格补齐；超长写入在严格模式（8.0 默认）下直接报错，不是静默截断。

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
[ ] 表和字段均有 COMMENT，类型符合表设计条款
[ ] 大表 DDL 已显式声明 ALGORITHM / LOCK，并评估 MDL 与主从延迟
[ ] 大规模回填 / 数据修复已给风险提示，且有前像备份
[ ] 已考虑滚动发布期间新旧应用兼容
[ ] 失败后的数据库状态明确，有上线后验证方式
```

发现问题 → 明确指出，禁止用幂等逻辑掩盖。

不要问"怎样让这个 SQL 在任何数据库状态下都能执行成功"，应该问"当前数据库是否严格处于本 Migration 要求的前置版本"。答案是否 → STOP，暴露差异，研发 / DBA 调查。
