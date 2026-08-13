[← 返回索引](./index.md)

# PostgreSQL 正式项目数据库与权限规范

## 1. 总体设计原则

正式项目建议采用：

```text
一个项目一个 database
一个主业务 schema
一个应用用户
业务表不直接放 public schema
生产环境应用用户只允许 CRUD
测试环境可以允许建表、改表、加索引
```

推荐结构：

```text
PostgreSQL Server
└── g_api
    └── app
        ├── users
        ├── api_keys
        ├── configs
        ├── request_logs
        └── operation_logs
```

推荐命名：

```text
Database: g_api
Schema: app
User: g_api_user
```

---

## 2. Database 和 Schema 的使用原则

## 2.1 一个正式项目一个 database

一个正式项目建议单独使用一个 database。

示例：

```text
g_api
g_cms
g_monitor
g_blog
```

例如：

```text
g_api 项目使用 g_api database
g_cms 项目使用 g_cms database
g_monitor 项目使用 g_monitor database
```

原因：

```text
1. 项目之间隔离清晰
2. 权限边界清楚
3. 备份恢复方便
4. 不同项目生命周期不同
5. 避免多个系统混在一个 database 里
```

---

## 2.2 一个正式项目先使用一个主 schema

正式项目初期推荐一个主业务 schema：

```text
app
```

结构：

```text
g_api
└── app
    ├── users
    ├── roles
    ├── permissions
    ├── api_keys
    ├── configs
    └── request_logs
```

优点：

```text
1. 比 public 更规范
2. 不会过度设计
3. 适合大多数正式项目初期
4. 后期可以继续扩展多个 schema
```

---

## 2.3 后期复杂后再拆多个 schema

当系统模块明显增多时，可以拆成多个 schema：

```text
g_api
├── app
│   ├── projects
│   ├── api_keys
│   └── configs
├── auth
│   ├── users
│   ├── roles
│   └── permissions
├── billing
│   ├── orders
│   └── invoices
└── log
    ├── request_logs
    └── operation_logs
```

常见 schema 规划：

```text
app      核心业务表
auth     用户、角色、权限
billing  订单、账单、支付
log      操作日志、请求日志
report   报表、统计数据
```

注意：

```text
不是一开始就必须拆多个 schema
正式项目初期一个 app schema 就够
```

---

## 3. 为什么正式项目不建议直接使用 public schema

`public` 是 PostgreSQL 默认 schema，学习、测试、小项目可以直接使用。

但正式项目建议新建业务 schema，例如：

```text
app
```

原因：

```text
1. 业务边界更清楚
2. 权限控制更规范
3. public schema 保持干净
4. 后期拆分模块更方便
5. 后期迁移、备份、权限治理更清晰
```

正式项目推荐：

```text
不要把业务表直接建在 public 下
统一放到 app schema 下
```

推荐：

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

不推荐：

```sql
CREATE TABLE public.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL
);
```

---

## 4. 推荐最终结构

正式项目推荐采用：

```text
Database: g_api
Schema: app
User: g_api_user
```

数据库结构：

```text
g_api
└── app
    ├── users
    ├── api_keys
    ├── configs
    ├── request_logs
    └── operation_logs
```

应用连接信息：

```text
Host: 数据库地址
Port: 5432
Database: g_api
User: g_api_user
Schema: app
```

---

## 5. 权限分层理解

PostgreSQL 权限可以按 4 层理解：

```text
1. Database 权限
   能不能连接这个数据库

2. Schema 权限
   能不能访问这个 schema

3. Table 权限
   能不能查询、插入、更新、删除表数据

4. Sequence 权限
   能不能使用自增 ID
```

对应授权：

```sql
GRANT CONNECT ON DATABASE g_api TO g_api_user;

GRANT USAGE ON SCHEMA app TO g_api_user;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_user;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_user;
```

---

## 6. 环境权限设计原则

PostgreSQL 权限建议按环境区分：

```text
生产环境：最小权限，只允许业务 CRUD
测试环境：开发权限，可建表、改表、加索引
开发环境：权限可以更宽松，但不建议使用超级用户
```

核心原则：

```text
生产环境应用用户：
只允许 SELECT、INSERT、UPDATE、DELETE
不允许 CREATE TABLE
不允许 ALTER TABLE
不允许 DROP TABLE
不允许 CREATE INDEX
不允许 DROP INDEX

测试环境应用用户：
允许 SELECT、INSERT、UPDATE、DELETE
允许 CREATE TABLE
允许 ALTER TABLE
允许 CREATE INDEX
允许 DROP / 调整测试表
```

---

# 7. 生产环境权限规范

## 7.1 生产环境推荐角色

生产环境建议至少分两个用户：

```text
g_api_user       应用运行用户
g_api_migrator   数据库迁移用户
```

说明：

```text
g_api_user：
给应用程序连接数据库使用，只负责业务读写。

g_api_migrator：
给 DBA、运维、CI/CD 数据库迁移工具使用，负责建表、改字段、加索引。
```

---

## 7.2 生产环境应用用户权限

生产环境应用用户只给 CRUD 权限。

```sql
GRANT CONNECT ON DATABASE g_api TO g_api_user;

GRANT USAGE ON SCHEMA app TO g_api_user;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_user;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_user;

-- 生产表由 g_api_migrator 创建，默认权限必须写 FOR ROLE g_api_migrator，
-- 否则只对当前执行者（如 postgres）新建的表生效，migrator 新建的表 g_api_user 拿不到权限
ALTER DEFAULT PRIVILEGES FOR ROLE g_api_migrator IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_user;

ALTER DEFAULT PRIVILEGES FOR ROLE g_api_migrator IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_user;

ALTER ROLE g_api_user SET search_path TO app;
```

生产环境应用用户不要授权：

```sql
GRANT CREATE ON SCHEMA app TO g_api_user;
```

也不要给：

```sql
ALTER USER g_api_user CREATEDB;
ALTER USER g_api_user CREATEROLE;
ALTER USER g_api_user SUPERUSER;
```

---

## 7.3 生产环境应用用户不允许做的事

生产环境应用用户不应该能执行：

```sql
CREATE TABLE app.test_table (...);

ALTER TABLE app.users ADD COLUMN nickname TEXT;

DROP TABLE app.users;

CREATE INDEX idx_users_email ON app.users(email);

DROP INDEX app.idx_users_email;

TRUNCATE TABLE app.users;
```

说明：

```text
生产环境的表结构变更、索引变更、字段变更，应该通过迁移工具或管理员账号执行。
```

例如：

```text
Flyway
Liquibase
Prisma Migrate
Django Migration
Rails Migration
自研 SQL 发布脚本
```

---

## 7.4 生产环境迁移用户权限

如果生产环境需要通过 CI/CD 自动执行数据库变更，建议单独创建迁移用户。

```text
User: g_api_migrator
用途：执行建表、改表、加索引等 DDL 操作
```

授权示例：

```sql
CREATE USER g_api_migrator WITH PASSWORD 'StrongPassword123!';

GRANT CONNECT ON DATABASE g_api TO g_api_migrator;

\c g_api

GRANT USAGE, CREATE ON SCHEMA app TO g_api_migrator;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_migrator;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_migrator;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_migrator;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_migrator;

ALTER ROLE g_api_migrator SET search_path TO app;
```

注意：

```text
生产环境不要让应用程序使用 g_api_migrator。
g_api_migrator 只给迁移工具、发布流程或管理员使用。
```

---

# 8. 测试环境权限规范

## 8.1 测试环境推荐角色

测试环境可以简单一些：

```text
g_api_test_user
```

这个用户可以同时负责：

```text
1. 应用连接
2. 建表
3. 改字段
4. 加索引
5. 删除测试表
```

---

## 8.2 测试环境应用用户权限

测试环境可以给 schema 创建权限：

```sql
GRANT CONNECT ON DATABASE g_api_test TO g_api_test_user;

GRANT USAGE, CREATE ON SCHEMA app TO g_api_test_user;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_test_user;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_test_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_test_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_test_user;

ALTER ROLE g_api_test_user SET search_path TO app;
```

关键区别是这里给了：

```sql
GRANT CREATE ON SCHEMA app TO g_api_test_user;
```

这样测试环境可以创建表、创建索引等对象。

---

## 8.3 测试环境允许做的事

测试环境可以允许：

```sql
CREATE TABLE app.test_orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  amount NUMERIC(10, 2),
  created_at TIMESTAMPTZ DEFAULT now()
);
```

```sql
ALTER TABLE app.users
ADD COLUMN nickname TEXT;
```

```sql
CREATE INDEX idx_users_email
ON app.users(email);
```

```sql
DROP TABLE app.test_orders;
```

---

# 9. 开发环境权限规范

开发环境可以比测试环境更宽松，但仍然不建议使用超级用户。

推荐：

```text
Database: g_api_dev
Schema: app
User: g_api_dev_user
```

开发环境用户可允许：

```text
CRUD
CREATE TABLE
ALTER TABLE
DROP TABLE
CREATE INDEX
DROP INDEX
TRUNCATE
```

但仍然不建议给：

```text
SUPERUSER
CREATEROLE
```

开发环境授权可以参考测试环境：

```sql
GRANT CONNECT ON DATABASE g_api_dev TO g_api_dev_user;

GRANT USAGE, CREATE ON SCHEMA app TO g_api_dev_user;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_dev_user;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_dev_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_dev_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_dev_user;

ALTER ROLE g_api_dev_user SET search_path TO app;
```

---

# 10. 生产、测试、开发权限对比

| 权限项               | 生产环境应用用户 | 测试环境应用用户 | 开发环境用户 |
| ----------------- | -------- | -------- | ------ |
| 连接数据库             | 允许       | 允许       | 允许     |
| 使用 schema         | 允许       | 允许       | 允许     |
| 查询数据 SELECT       | 允许       | 允许       | 允许     |
| 插入数据 INSERT       | 允许       | 允许       | 允许     |
| 更新数据 UPDATE       | 允许       | 允许       | 允许     |
| 删除数据 DELETE       | 允许       | 允许       | 允许     |
| 使用自增序列            | 允许       | 允许       | 允许     |
| 创建表 CREATE TABLE  | 不允许      | 允许       | 允许     |
| 修改表 ALTER TABLE   | 不允许      | 允许       | 允许     |
| 删除表 DROP TABLE    | 不允许      | 允许       | 允许     |
| 创建索引 CREATE INDEX | 不允许      | 允许       | 允许     |
| 删除索引 DROP INDEX   | 不允许      | 允许       | 允许     |
| TRUNCATE 表        | 不允许      | 可允许      | 可允许    |
| 创建数据库 CREATEDB    | 不允许      | 不建议      | 不建议    |
| 创建角色 CREATEROLE   | 不允许      | 不建议      | 不建议    |
| 超级用户 SUPERUSER    | 不允许      | 不允许      | 不建议    |

---

# 11. 正式项目完整初始化脚本

> 执行说明：以下脚本含 `\c`（psql 元命令）和 `CREATE DATABASE`（不能在事务块中执行），
> 需用 psql 逐条执行，不要包进单个事务（不要加 `-1` / `--single-transaction`）。

## 11.1 生产环境初始化脚本

示例：

```text
Database: g_api
Schema: app
应用用户: g_api_user
迁移用户: g_api_migrator
```

完整脚本：

```sql
-- 1. 创建数据库（统一使用 template0 并显式指定编码/排序规则，与其他规范文档保持一致）
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;

-- 2. 创建生产应用用户
CREATE USER g_api_user WITH PASSWORD 'StrongPassword123!';

-- 3. 创建生产迁移用户
CREATE USER g_api_migrator WITH PASSWORD 'StrongPassword123!';

-- 4. 连接到业务数据库
\c g_api

-- 5. 创建业务 schema
CREATE SCHEMA app;

-- 6. 授权应用用户连接数据库
GRANT CONNECT ON DATABASE g_api TO g_api_user;

-- 7. 授权迁移用户连接数据库
GRANT CONNECT ON DATABASE g_api TO g_api_migrator;

-- 8. 授权应用用户使用 app schema
GRANT USAGE ON SCHEMA app TO g_api_user;

-- 9. 授权迁移用户使用和创建 app schema 下对象
GRANT USAGE, CREATE ON SCHEMA app TO g_api_migrator;

-- 10. 授权应用用户已有表 CRUD 权限
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_user;

-- 11. 授权应用用户已有 sequence 权限
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_user;

-- 12. 授权迁移用户已有表 CRUD 权限
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_migrator;

-- 13. 授权迁移用户已有 sequence 权限
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_migrator;

-- 14. 授权应用用户对 migrator 未来新建表的 CRUD 权限
--     必须写 FOR ROLE g_api_migrator：默认权限只对指定角色新建的对象生效，
--     生产表由 g_api_migrator 创建，不写则 g_api_user 对新表没有权限
ALTER DEFAULT PRIVILEGES FOR ROLE g_api_migrator IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_user;

-- 15. 授权应用用户对 migrator 未来新建 sequence 的权限
ALTER DEFAULT PRIVILEGES FOR ROLE g_api_migrator IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_user;

-- 16. 迁移用户建表后本身即为 owner，自带全部权限，
--     以下两条仅作为“非 migrator 创建的表”的保险，可按需保留
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_migrator;

-- 17. 授权迁移用户未来新建 sequence 权限（同上，保险）
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_migrator;

-- 18. 设置应用用户默认 schema
ALTER ROLE g_api_user SET search_path TO app;

-- 19. 设置迁移用户默认 schema
ALTER ROLE g_api_migrator SET search_path TO app;
```

说明：

```text
g_api_user：
生产应用连接使用，只允许业务 CRUD。

g_api_migrator：
数据库迁移、建表、改字段、加索引使用，不给应用程序使用。
```

---

## 11.2 测试环境初始化脚本

示例：

```text
Database: g_api_test
Schema: app
User: g_api_test_user
```

完整脚本：

```sql
-- 1. 创建测试数据库（与生产一致，使用 template0 + 显式编码，避免测试与生产编码/排序不一致）
CREATE DATABASE g_api_test
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;

-- 2. 创建测试用户
CREATE USER g_api_test_user WITH PASSWORD 'StrongPassword123!';

-- 3. 连接到测试数据库
\c g_api_test

-- 4. 创建业务 schema
CREATE SCHEMA app;

-- 5. 授权测试用户连接数据库
GRANT CONNECT ON DATABASE g_api_test TO g_api_test_user;

-- 6. 授权测试用户使用和创建 app schema 下对象
GRANT USAGE, CREATE ON SCHEMA app TO g_api_test_user;

-- 7. 授权已有表 CRUD 权限
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_test_user;

-- 8. 授权已有 sequence 权限
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_test_user;

-- 9. 授权未来新建表 CRUD 权限
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_test_user;

-- 10. 授权未来新建 sequence 权限
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_test_user;

-- 11. 设置默认 schema
ALTER ROLE g_api_test_user SET search_path TO app;
```

说明：

```text
测试环境用户允许 CRUD + CREATE + ALTER + INDEX。
方便测试环境快速建表、改字段、加索引。
```

---

## 11.3 开发环境初始化脚本

示例：

```text
Database: g_api_dev
Schema: app
User: g_api_dev_user
```

完整脚本：

```sql
-- 1. 创建开发数据库（与生产一致，使用 template0 + 显式编码）
CREATE DATABASE g_api_dev
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;

-- 2. 创建开发用户
CREATE USER g_api_dev_user WITH PASSWORD 'StrongPassword123!';

-- 3. 连接到开发数据库
\c g_api_dev

-- 4. 创建业务 schema
CREATE SCHEMA app;

-- 5. 授权开发用户连接数据库
GRANT CONNECT ON DATABASE g_api_dev TO g_api_dev_user;

-- 6. 授权开发用户使用和创建 app schema 下对象
GRANT USAGE, CREATE ON SCHEMA app TO g_api_dev_user;

-- 7. 授权已有表 CRUD 权限
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_dev_user;

-- 8. 授权已有 sequence 权限
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_dev_user;

-- 9. 授权未来新建表 CRUD 权限
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_dev_user;

-- 10. 授权未来新建 sequence 权限
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_dev_user;

-- 11. 设置默认 schema
ALTER ROLE g_api_dev_user SET search_path TO app;
```

---

# 12. 应用建表示例

推荐所有业务表都放到 `app` schema 下。

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  username TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

```sql
CREATE TABLE app.api_keys (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL,
  api_key TEXT NOT NULL,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

```sql
CREATE TABLE app.request_logs (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT,
  path TEXT NOT NULL,
  method TEXT NOT NULL,
  status_code INT,
  cost_ms INT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

# 13. 应用连接建议

如果已经设置：

```sql
ALTER ROLE g_api_user SET search_path TO app;
```

应用代码里可以直接写：

```sql
SELECT * FROM users;
```

否则需要写完整 schema：

```sql
SELECT * FROM app.users;
```

正式项目推荐：

```text
数据库层面设置 search_path
SQL 里重要场景仍然可以显式写 app.users
```

---

# 14. 正式项目权限建议

## 14.1 应用用户不要使用超级用户

不推荐应用直接使用：

```text
postgres
root
admin
superuser
```

推荐：

```text
g_api_user
```

原因：

```text
1. 降低误操作风险
2. 限制权限范围
3. 更符合生产安全规范
4. 出问题方便审计
```

---

## 14.2 应用用户一般不建议拥有建库权限

不建议：

```sql
ALTER USER g_api_user CREATEDB;
```

应用用户只需要：

```text
CONNECT
USAGE ON SCHEMA
SELECT
INSERT
UPDATE
DELETE
USAGE ON SEQUENCES
```

---

## 14.3 生产环境不建议应用用户拥有 DDL 权限

生产环境不建议给应用用户：

```sql
GRANT CREATE ON SCHEMA app TO g_api_user;
```

更推荐：

```text
建表、改表、加索引交给迁移用户或管理员账号执行
应用用户只负责业务读写
```

---

# 15. 常见问题

## 15.1 明明授权了表，为什么还是查不了？

可能少了 schema 权限：

```sql
GRANT USAGE ON SCHEMA app TO g_api_user;
```

---

## 15.2 插入数据报 sequence 权限错误怎么办？

本规范所有示例表都用 `GENERATED ALWAYS AS IDENTITY`，其隐式序列由系统内部管理，
插入时只校验表的 INSERT 权限，无需单独的序列权限，正常情况下不会遇到本节报错。
下面这个错误主要出现在使用 `SERIAL` 列的场景：

```text
permission denied for sequence users_id_seq
```

若确实使用了 `SERIAL`，处理：

```sql
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_user;
```

未来新建的 sequence 也要授权（生产表由 g_api_migrator 创建，需加 FOR ROLE）：

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE g_api_migrator IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_user;
```

---

## 15.3 为什么新建表后应用用户没权限？

因为之前的：

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_user;
```

只对已经存在的表生效。

未来新表需要（生产环境表由 g_api_migrator 创建，务必加 `FOR ROLE g_api_migrator`，
否则默认权限只对当前执行者创建的表生效，migrator 建的新表仍然没权限）：

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE g_api_migrator IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_user;
```

---

## 15.4 为什么应用里 SELECT * FROM users 找不到表？

如果表在 `app.users`，但没有设置 `search_path`，直接查 `users` 可能找不到。

解决方式一：SQL 写完整路径：

```sql
SELECT * FROM app.users;
```

解决方式二：设置默认 schema：

```sql
ALTER ROLE g_api_user SET search_path TO app;
```

---

## 15.5 生产环境为什么不让应用用户建表、改表？

原因：

```text
1. 防止应用 bug 误删表、误改表
2. 防止 SQL 注入造成结构破坏
3. 防止未经审核的字段变更进入生产
4. 保证数据库变更有发布流程
5. 便于回滚、审计和追责
```

正确方式：

```text
应用用户负责 CRUD
迁移用户负责 DDL
管理员负责高权限操作
```

---

# 16. 和 MySQL 的理解对比

| 项目     | MySQL               | PostgreSQL                              |
| ------ | ------------------- | --------------------------------------- |
| 服务实例   | MySQL Server        | PostgreSQL Server                       |
| 数据库    | Database            | Database                                |
| schema | 通常和 database 概念接近   | database 下的命名空间                         |
| 表路径    | db.table            | db.schema.table                         |
| 用户     | 'user'@'host'       | role/user                               |
| 授权     | GRANT ON db.*       | GRANT ON database/schema/table/sequence |
| 自增     | AUTO_INCREMENT      | IDENTITY / SERIAL / SEQUENCE            |
| 默认空间   | database            | public schema                           |
| 权限刷新   | 常见 FLUSH PRIVILEGES | 不需要 FLUSH PRIVILEGES                    |
| 生产应用权限 | 常见 db.*             | 建议只给 CRUD                               |
| DDL 权限 | 可混用但不推荐             | 建议迁移用户单独管理                              |

---

# 17. 最终推荐权限模型

## 17.1 生产环境

```text
Database: g_api
Schema: app

User:
  g_api_user       应用运行用户，只允许 CRUD
  g_api_migrator   数据库迁移用户，允许 DDL
```

生产环境应用使用：

```text
g_api_user
```

生产环境数据库发布使用：

```text
g_api_migrator
```

---

## 17.2 测试环境

```text
Database: g_api_test
Schema: app

User:
  g_api_test_user
```

测试环境可以让 `g_api_test_user` 拥有：

```text
CRUD + CREATE + ALTER + INDEX
```

---

## 17.3 开发环境

```text
Database: g_api_dev
Schema: app

User:
  g_api_dev_user
```

开发环境可以让 `g_api_dev_user` 拥有：

```text
CRUD + CREATE + ALTER + DROP + INDEX
```

但仍不建议使用超级用户。

---

# 18. 最终结论

正式项目不要纠结，直接采用：

```text
Database: 项目名
Schema: app
User: 项目名_user
```

例如：

```text
Database: g_api
Schema: app
User: g_api_user
```

生产环境采用：

```text
g_api_user       只允许 CRUD
g_api_migrator   允许 DDL
```

测试环境采用：

```text
g_api_test_user  允许 CRUD + 建表 + 改字段 + 加索引
```

开发环境采用：

```text
g_api_dev_user   权限可更宽松，但不建议超级用户
```

一句话总结：

```text
正式项目：一个 database，一个 app schema。
生产环境：应用用户只 CRUD，DDL 交给迁移用户。
测试环境：允许应用/测试用户建表、改表、加索引，方便验证。
```

