# PostgreSQL 文档索引

本目录收录 PostgreSQL 在正式项目中的**规范**、**命令速查**与**初始化模板**共 6 篇文档。规范类讲“应该怎么做、为什么”,速查类讲“具体命令怎么敲”,模板类是可直接套用的成品脚本。

---

## 推荐阅读顺序

1. 先读 [PostgreSQL 正式项目数据库与权限规范](#1-postgresql-正式项目数据库与权限规范) —— 建立 database / schema / 用户权限的整体设计观。
2. 建库前读 [template0 / template1 使用说明](#2-postgresql-template0--template1-使用说明) 和 [字符集、大小写、时区、备份恢复速查](#5-postgresql-字符集大小写时区备份恢复命令速查)。
3. 日常操作查 [psql 常用命令速查](#4-postgresql-psql-常用命令速查)。
4. 涉及半结构化字段时读 [JSONB 使用规范](#3-postgresql-jsonb-使用规范)。
5. 真正动手建开发 / 测试库时,套用 [开发测试库初始化模板](#6-postgresql-开发测试库初始化模板)。

---

## 按场景快速定位

| 我想做什么 | 看哪篇 |
| --- | --- |
| 规划一个新项目的库、schema、用户 | [数据库与权限规范](./postgresql-project-db-and-privileges.md) 第 1–4 章 |
| 生产 / 测试 / 开发环境权限怎么分 | [数据库与权限规范](./postgresql-project-db-and-privileges.md) 第 6–10 章 |
| 拿一份可直接跑的初始化脚本 | [数据库与权限规范](./postgresql-project-db-and-privileges.md) 第 11 章 |
| 建开发 / 测试库,要一份改个名就能跑的完整脚本 | [开发测试库初始化模板](./开发测试数据库初始化模板.md) |
| 授权/权限相关的报错排查 | [数据库与权限规范](./postgresql-project-db-and-privileges.md) 第 15 章 |
| 建库该用 template0 还是 template1 | [template 使用说明](./postgresql-template-databases.md) |
| 建库指定字符集、排序规则、时区 | [字符集…备份恢复速查](./postgresql-charset-timezone-backup-cheatsheet.md) 第 1–4 章 |
| 备份与恢复数据库 | [字符集…备份恢复速查](./postgresql-charset-timezone-backup-cheatsheet.md) 第 9–14 章 |
| 查表、改表、建索引、增删改查等日常命令 | [psql 常用命令速查](./postgresql-psql-cheatsheet.md) |
| 从 MySQL 迁移过来对照命令 | [psql 常用命令速查](./postgresql-psql-cheatsheet.md) 第 32 章 / [权限规范](./postgresql-project-db-and-privileges.md) 第 16 章 |
| 存不固定 / 扩展 / 半结构化字段 | [JSONB 使用规范](./postgresql-jsonb-guide.md) |

---

## 文档列表

### 📐 规范类

#### 1. [PostgreSQL 正式项目数据库与权限规范](./postgresql-project-db-and-privileges.md)

正式项目的库/Schema/权限顶层设计规范,是本目录的**核心**。

- 总体设计:一个项目一个 database、一个主 `app` schema、业务表不放 `public`
- 权限分层(Database / Schema / Table / Sequence)与生产、测试、开发三套环境的权限边界
- 生产双角色模型:`g_api_user`(只 CRUD)+ `g_api_migrator`(DDL 迁移)
- 三套环境的**完整可执行初始化脚本**(第 11 章)
- 常见权限报错 FAQ(第 15 章)与 MySQL 概念对照(第 16 章)

#### 2. [PostgreSQL template0 / template1 使用说明](./postgresql-template-databases.md)

讲清两个模板库的区别与正式项目的建库姿势。

- template0(出厂干净母版)与 template1(默认工作母版)的区别
- 为什么指定字符集/排序规则时要用 `TEMPLATE = template0`
- 不污染 template0、不随意改 template1,结构交给 migration 管理
- `C.UTF-8` 不可用时的替代方案、模板库查看命令

#### 3. [PostgreSQL JSONB 使用规范](./postgresql-jsonb-guide.md)

半结构化字段的设计与查询规范。

- JSON vs JSONB 的取舍(正式项目默认 JSONB)
- 适合 / 不适合放 JSONB 的字段边界:核心字段单独建列,扩展字段放 JSONB
- 查询(`->`、`->>`、`@>`、`?`/`?|`/`?&`)、更新(`jsonb_set`、`||`、`-`、`#-`)、GIN / 表达式索引
- 字段命名、默认值(`'{}'` / `'[]'`)与设计原则

### ⚡ 速查类

#### 4. [PostgreSQL psql 常用命令速查](./postgresql-psql-cheatsheet.md)

带注释的 psql 命令大全,日常操作首选查询手册,共 32 节。

- 连接/切库/Schema、`\dt` `\d` `\di` `\ds` `\du` `\dp` 等查看命令
- ALTER TABLE、约束、索引(含 `CONCURRENTLY`)、序列、用户与授权
- 增删改查、TRUNCATE、分批删除、事务保护、导入导出(`\copy`)
- 系统查询、字符集/时区、建删库、备份恢复、MySQL 对照

#### 5. [PostgreSQL 字符集、大小写、时区、备份恢复命令速查](./postgresql-charset-timezone-backup-cheatsheet.md)

聚焦“编码 / 排序 / 大小写 / 时区 / 备份恢复”五类命令。

- 查看与指定数据库字符集、排序规则(collation)
- 时区查看与设置(库级 / 用户级 / 会话级),`TIMESTAMPTZ` 时间字段写法
- 标识符大小写规范、大小写敏感/不敏感查询、`lower()` 唯一索引
- `pg_dump` / `pg_restore` / `pg_dumpall` 备份恢复全流程与恢复后检查

### 🧩 模板类

#### 6. [PostgreSQL 开发测试库初始化模板](./开发测试数据库初始化模板.md)

开发 / 测试环境从零建库的可执行脚本,12 个步骤用管理员账号按顺序执行即可。

- 与规范第 11.3 节是**两种做法**:那边用 GRANT + ALTER DEFAULT PRIVILEGES 授权,这边用 `SET ROLE` 让开发用户自建 `app` schema,从而直接成为 owner
- 数据库仍归管理员持有,开发用户只拿 database 级 `ALL`(即 CREATE / CONNECT / TEMPORARY 三项)
- 收紧 `PUBLIC` 的数据库权限与 `public` schema 建对象权限
- 开发用户默认 `search_path` 指向 `app`,时区在库级与角色级均设为 UTC

---

> 约定:前 5 篇文档示例统一使用 `Database: g_api`、`Schema: app`、`User: g_api_user`,可按实际项目替换;第 6 篇是已填好实际项目名(`llm_gatecheck` / `llm_gatecheck_user`)的成品脚本,套用时全文替换库名与角色名。
