[← 返回索引](./index.md)

# PostgreSQL psql 常用命令速查：命令 + 注释

## 1. 连接数据库

```bash
# 使用 postgres 用户连接本机 PostgreSQL
psql -U postgres
```

```bash
# 使用 postgres 用户连接指定数据库
psql -U postgres -d g_api
```

```bash
# 指定主机、端口、用户、数据库连接
psql -h 127.0.0.1 -p 5432 -U g_api_user -d g_api
```

```bash
# 临时指定密码连接，不推荐长期写在脚本中
PGPASSWORD='your_password' psql -h 127.0.0.1 -p 5432 -U g_api_user -d g_api
```

```sql
-- 查看当前连接信息
\conninfo
```

```sql
-- 退出 psql
\q
```

---

## 2. 数据库查看与切换

```sql
-- 查看所有数据库
\l
```

```sql
-- 查看所有数据库，等价于 \l
\list
```

```sql
-- 切换到 g_api 数据库
\c g_api
```

```sql
-- 使用指定用户切换到指定数据库
\c g_api g_api_user
```

```sql
-- 查看当前所在数据库
SELECT current_database();
```

---

## 3. Schema 操作

```sql
-- 查看所有 schema
\dn
```

```sql
-- 查看 schema 详细信息，包括权限
\dn+
```

```sql
-- 设置当前会话默认 schema 为 app
SET search_path TO app;
```

```sql
-- 查看当前 search_path
SHOW search_path;
```

```sql
-- 设置某个用户默认使用 app schema
ALTER ROLE g_api_user SET search_path TO app;
```

---

## 4. 表查看命令

```sql
-- 查看当前 schema 下的表
\dt
```

```sql
-- 查看 app schema 下的表
\dt app.*
```

```sql
-- 查看所有 schema 下的表
\dt *.*
```

```sql
-- 查看 app.users 表结构
\d app.users
```

```sql
-- 查看 app.users 表详细结构，包括大小、索引等
\d+ app.users
```

```sql
-- 查看 app.users 表权限
\dp app.users
```

```sql
-- 查看 app schema 下所有表权限
\dp app.*
```

---

## 5. 表结构修改：ALTER TABLE

```sql
-- 给 app.users 表新增字段
ALTER TABLE app.users
ADD COLUMN nickname TEXT;
```

```sql
-- 给 app.users 表新增带默认值的字段
ALTER TABLE app.users
ADD COLUMN status TEXT DEFAULT 'active';
```

```sql
-- 给字段增加默认值
ALTER TABLE app.users
ALTER COLUMN status SET DEFAULT 'active';
```

```sql
-- 删除字段默认值
ALTER TABLE app.users
ALTER COLUMN status DROP DEFAULT;
```

```sql
-- 修改字段类型
ALTER TABLE app.users
ALTER COLUMN age TYPE INT;
```

```sql
-- 修改字段类型，并指定转换方式
ALTER TABLE app.users
ALTER COLUMN age TYPE BIGINT
USING age::BIGINT;
```

```sql
-- 字段设置为 NOT NULL
ALTER TABLE app.users
ALTER COLUMN email SET NOT NULL;
```

```sql
-- 字段取消 NOT NULL
ALTER TABLE app.users
ALTER COLUMN nickname DROP NOT NULL;
```

```sql
-- 修改字段名
ALTER TABLE app.users
RENAME COLUMN nickname TO display_name;
```

```sql
-- 删除字段
ALTER TABLE app.users
DROP COLUMN display_name;
```

```sql
-- 删除字段，如果存在才删除
ALTER TABLE app.users
DROP COLUMN IF EXISTS display_name;
```

```sql
-- 修改表名
ALTER TABLE app.users
RENAME TO app_users;
```

说明：

```text
RENAME TO 后面只写新表名，不写 schema。
app.users 会变成 app.app_users。
```

---

## 6. 约束操作

```sql
-- 添加唯一约束
ALTER TABLE app.users
ADD CONSTRAINT uk_users_email UNIQUE (email);
```

```sql
-- 添加检查约束
ALTER TABLE app.users
ADD CONSTRAINT ck_users_age CHECK (age >= 0);
```

```sql
-- 添加外键约束
ALTER TABLE app.api_keys
ADD CONSTRAINT fk_api_keys_user_id
FOREIGN KEY (user_id) REFERENCES app.users(id);
```

```sql
-- 删除约束
ALTER TABLE app.users
DROP CONSTRAINT uk_users_email;
```

```sql
-- 删除约束，如果存在才删除
ALTER TABLE app.users
DROP CONSTRAINT IF EXISTS uk_users_email;
```

---

## 7. 索引操作

```sql
-- 查看当前 schema 下的索引
\di
```

```sql
-- 查看 app schema 下的索引
\di app.*
```

```sql
-- 查看表结构，里面也会显示索引
\d app.users
```

```sql
-- 查看 users 表的索引定义
SELECT indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'app'
  AND tablename = 'users';
```

```sql
-- 给 email 字段创建普通索引
CREATE INDEX idx_users_email
ON app.users(email);
```

```sql
-- 创建唯一索引
CREATE UNIQUE INDEX uk_users_email
ON app.users(email);
```

```sql
-- 创建联合索引
CREATE INDEX idx_users_status_created_at
ON app.users(status, created_at);
```

```sql
-- 创建降序索引
CREATE INDEX idx_users_created_at_desc
ON app.users(created_at DESC);
```

```sql
-- 创建表达式索引，适合大小写不敏感邮箱查询
CREATE INDEX idx_users_email_lower
ON app.users(lower(email));
```

```sql
-- 创建部分索引，只给 active 用户建索引
CREATE INDEX idx_users_active_email
ON app.users(email)
WHERE status = 'active';
```

```sql
-- 创建 JSONB GIN 索引
CREATE INDEX idx_users_profile_gin
ON app.users
USING GIN (profile);
```

```sql
-- 创建索引时不锁表写入，生产环境推荐
CREATE INDEX CONCURRENTLY idx_users_email
ON app.users(email);
```

```sql
-- 删除索引
DROP INDEX app.idx_users_email;
```

```sql
-- 如果索引存在才删除
DROP INDEX IF EXISTS app.idx_users_email;
```

```sql
-- 生产环境删除索引，减少锁影响
DROP INDEX CONCURRENTLY IF EXISTS app.idx_users_email;
```

说明：

```text
CREATE INDEX CONCURRENTLY 和 DROP INDEX CONCURRENTLY 不能放在事务块里执行。
```

---

## 8. 序列操作

```sql
-- 查看当前 schema 下的序列
\ds
```

```sql
-- 查看 app schema 下的序列
\ds app.*
```

```sql
-- 查看指定序列详情
\d app.users_id_seq
```

```sql
-- 查看 app schema 下对象权限，包含表和序列权限
\dp app.*
```

---

## 9. 用户和角色

```sql
-- 查看用户和角色
\du
```

```sql
-- 查看用户和角色详细信息
\du+
```

```sql
-- 查看当前用户
SELECT current_user;
```

```sql
-- 查看当前会话用户
SELECT session_user;
```

```sql
-- 创建用户
CREATE USER g_api_user WITH PASSWORD 'StrongPassword123!';
```

```sql
-- 修改用户密码
ALTER USER g_api_user WITH PASSWORD 'NewPassword123!';
```

```sql
-- 删除用户
DROP USER g_api_user;
```

---

## 10. 权限查看

```sql
-- 查看 app schema 下所有表、序列权限
\dp app.*
```

```sql
-- 查看 schema 权限
\dn+
```

```sql
-- 查看数据库权限
\l+
```

```sql
-- 查看指定用户的角色属性
\du+ g_api_user
```

---

## 11. 常用授权命令

```sql
-- 授权用户可以连接 g_api 数据库
GRANT CONNECT ON DATABASE g_api TO g_api_user;
```

```sql
-- 授权用户可以使用 app schema
GRANT USAGE ON SCHEMA app TO g_api_user;
```

```sql
-- 授权用户可以在 app schema 下创建对象，测试环境可用，生产应用用户不建议
GRANT CREATE ON SCHEMA app TO g_api_user;
```

```sql
-- 授权用户对 app schema 下已有表进行 CRUD
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO g_api_user;
```

```sql
-- 授权用户使用 app schema 下已有序列
-- （SERIAL 列插入才需要序列权限；GENERATED AS IDENTITY 列不需要，此处为兼容 SERIAL 的保险）
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO g_api_user;
```

```sql
-- 授权未来新建表的 CRUD 权限
-- 注意：默认权限只对“执行本命令的角色”后续新建的对象生效。
-- 多角色场景（如 migrator 建表、g_api_user 读写）必须写 FOR ROLE 建表角色，
-- 否则新表 g_api_user 拿不到权限，详见《PostgreSQL 正式项目数据库与权限规范》。
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO g_api_user;
```

```sql
-- 授权未来新建序列的使用权限
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES TO g_api_user;
```

---

## 12. 执行 SQL 文件

```bash
# 从命令行执行 SQL 文件
psql -h 127.0.0.1 -U postgres -d g_api -f init.sql
```

```sql
-- 进入 psql 后执行指定 SQL 文件
\i /path/to/init.sql
```

```sql
-- 执行当前目录下的 SQL 文件
\i init.sql
```

---

## 13. 查询数据

```sql
-- 查询整张表
SELECT *
FROM app.users;
```

```sql
-- 查询指定字段
SELECT id, username, email
FROM app.users;
```

```sql
-- 条件查询
SELECT *
FROM app.users
WHERE status = 'active';
```

```sql
-- 排序
SELECT *
FROM app.users
ORDER BY created_at DESC;
```

```sql
-- 分页
SELECT *
FROM app.users
ORDER BY id DESC
LIMIT 10 OFFSET 0;
```

```sql
-- 统计数量
SELECT count(*)
FROM app.users;
```

```sql
-- 分组统计
SELECT status, count(*)
FROM app.users
GROUP BY status;
```

---

## 14. 更新数据：UPDATE

```sql
-- 更新单个用户状态
UPDATE app.users
SET status = 'disabled'
WHERE id = 1;
```

```sql
-- 更新多个字段
UPDATE app.users
SET
  username = 'tom',
  updated_at = now()
WHERE id = 1;
```

```sql
-- 更新并返回修改后的数据
UPDATE app.users
SET status = 'active'
WHERE id = 1
RETURNING *;
```

```sql
-- 批量更新
UPDATE app.users
SET status = 'disabled'
WHERE last_login_at < now() - interval '180 days';
```

生产建议：

```text
UPDATE 生产环境先 SELECT 确认影响范围。
```

---

## 15. 删除数据：DELETE

```sql
-- 删除指定用户
DELETE FROM app.users
WHERE id = 1;
```

```sql
-- 删除某个状态的数据
DELETE FROM app.users
WHERE status = 'disabled';
```

```sql
-- 删除指定时间之前的数据
DELETE FROM app.request_logs
WHERE created_at < now() - interval '30 days';
```

```sql
-- 删除并返回被删除的数据
DELETE FROM app.users
WHERE id = 1
RETURNING *;
```

```sql
-- 删除前先查看将要删除的数据
SELECT *
FROM app.users
WHERE status = 'disabled';
```

生产建议：

```text
先 SELECT 确认，再 DELETE。
```

---

## 16. 清空表：TRUNCATE

```sql
-- 清空表数据，速度快，但风险高
TRUNCATE TABLE app.request_logs;
```

```sql
-- 清空表，并重置自增 ID
TRUNCATE TABLE app.request_logs RESTART IDENTITY;
```

```sql
-- 清空表，并级联清空依赖外键的表
TRUNCATE TABLE app.request_logs CASCADE;
```

说明：

```text
TRUNCATE 属于高风险操作，生产环境谨慎使用。
生产环境更推荐先备份，再执行。
```

---

## 17. 批量删除建议

```sql
-- 分批删除旧日志，每次删除 1000 条
DELETE FROM app.request_logs
WHERE id IN (
  SELECT id
  FROM app.request_logs
  WHERE created_at < now() - interval '30 days'
  LIMIT 1000
);
```

说明：

```text
大表不要一次 DELETE 太多数据。
建议分批删除，降低锁和 WAL 压力。
```

---

## 18. 生产环境操作建议

```sql
-- 先确认影响数据量
SELECT count(*)
FROM app.users
WHERE status = 'disabled';
```

```sql
-- 再抽样查看
SELECT *
FROM app.users
WHERE status = 'disabled'
LIMIT 20;
```

```sql
-- 使用事务保护
BEGIN;

DELETE FROM app.users
WHERE status = 'disabled';

-- 确认没问题后提交
COMMIT;

-- 如果发现问题，提交前可以回滚
-- ROLLBACK;
```

说明：

```text
执行 COMMIT 后无法直接 ROLLBACK。
生产环境大批量 DELETE / UPDATE 前必须备份。
```

---

## 19. 导出和导入数据

```sql
-- 将后续查询结果输出到文件
\o /tmp/users.txt
```

```sql
-- 执行查询，结果会写入 /tmp/users.txt
SELECT * FROM app.users;
```

```sql
-- 关闭输出到文件，恢复正常输出
\o
```

```sql
-- 导出整张表为 CSV
\copy app.users TO '/tmp/users.csv' CSV HEADER
```

```sql
-- 导出查询结果为 CSV
-- 注意：\copy 是 psql 元命令，必须写在一行；需要多行时改用 SQL 的 COPY (...) TO STDOUT 配 \g
\copy (SELECT id, username, email FROM app.users) TO '/tmp/users.csv' CSV HEADER
```

```sql
-- 从 CSV 导入数据到指定字段
\copy app.users(username, email) FROM '/tmp/users.csv' CSV HEADER
```

---

## 20. 显示格式设置

```sql
-- 开启扩展显示，适合查看字段很多的结果
\x
```

```sql
-- 关闭扩展显示
\x off
```

```sql
-- 显示 SQL 执行时间
\timing
```

```sql
-- 关闭 SQL 执行时间显示
\timing off
```

```sql
-- 设置 NULL 值显示样式
\pset null '[NULL]'
```

```sql
-- 只显示数据，不显示表头
\t
```

```sql
-- 恢复显示表头
\t off
```

---

## 21. 帮助命令

```sql
-- 查看 psql 内置命令帮助
\?
```

```sql
-- 查看 SQL 语法帮助
\h
```

```sql
-- 查看 CREATE TABLE 语法帮助
\h CREATE TABLE
```

```sql
-- 查看 ALTER TABLE 语法帮助
\h ALTER TABLE
```

```sql
-- 查看 CREATE INDEX 语法帮助
\h CREATE INDEX
```

```sql
-- 查看 GRANT 语法帮助
\h GRANT
```

---

## 22. 常用系统查询

```sql
-- 查看 PostgreSQL 版本
SELECT version();
```

```sql
-- 查看当前时间
SELECT now();
```

```sql
-- 查看当前时区
SHOW timezone;
```

```sql
-- 查看当前数据库连接数量
SELECT count(*)
FROM pg_stat_activity;
```

```sql
-- 查看当前连接详情
SELECT
  pid,
  datname,
  usename,
  client_addr,
  state,
  query
FROM pg_stat_activity;
```

```sql
-- 查看当前正在执行的 SQL
SELECT
  pid,
  usename,
  datname,
  state,
  query,
  now() - query_start AS running_time
FROM pg_stat_activity
WHERE state <> 'idle';
```

---

## 23. 字符集和排序规则

```sql
-- 查看所有数据库的字符集和排序规则
SELECT
  datname,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate,
  datctype
FROM pg_database;
```

```sql
-- 查看当前数据库的字符集和排序规则
SELECT
  datname,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate,
  datctype
FROM pg_database
WHERE datname = current_database();
```

```sql
-- 查看客户端编码
SHOW client_encoding;
```

```sql
-- 设置当前会话客户端编码为 UTF8
SET client_encoding = 'UTF8';
```

```sql
-- 查看可用 collation
SELECT
  collname,
  collcollate,
  collctype
FROM pg_collation
ORDER BY collname;
```

---

## 24. 时区操作

```sql
-- 查看当前时区
SHOW timezone;
```

```sql
-- 查看当前时间
SELECT now();
```

```sql
-- 查看 UTC 时间
SELECT now() AT TIME ZONE 'UTC';
```

```sql
-- 查看东京时间
SELECT now() AT TIME ZONE 'Asia/Tokyo';
```

```sql
-- 设置当前会话时区为 UTC
SET timezone TO 'UTC';
```

```sql
-- 设置数据库默认时区为 UTC
ALTER DATABASE g_api SET timezone TO 'UTC';
```

```sql
-- 设置用户默认时区为 UTC
ALTER ROLE g_api_user SET timezone TO 'UTC';
```

---

## 25. 创建和删除数据库

```sql
-- 创建正式数据库，指定 UTF8、排序规则和 template0
-- 注意：CREATE DATABASE 的 WITH 选项之间用空格分隔，不能加逗号，否则报语法错误
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

```sql
-- 创建普通数据库，默认使用 template1
CREATE DATABASE g_api;
```

```sql
-- 查看当前有哪些连接正在使用 g_api 数据库
SELECT
  pid,
  usename,
  client_addr,
  state,
  query
FROM pg_stat_activity
WHERE datname = 'g_api';
```

```sql
-- 强制断开 g_api 数据库连接，删除数据库前常用
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'g_api'
  AND pid <> pg_backend_pid();
```

```sql
-- 删除数据库
DROP DATABASE g_api;
```

---

## 26. template0 / template1 查看

```sql
-- 查看 template0 和 template1 配置
SELECT
  datname,
  datistemplate,
  datallowconn,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate,
  datctype
FROM pg_database
WHERE datname IN ('template0', 'template1');
```

```sql
-- 查看所有模板数据库
SELECT
  datname,
  datistemplate,
  datallowconn
FROM pg_database
WHERE datistemplate = true;
```

```sql
-- 使用 template0 创建干净数据库
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

```sql
-- 使用 template1 创建数据库，CREATE DATABASE 默认就是 template1
CREATE DATABASE test_db TEMPLATE template1;
```

---

## 27. 备份命令

```bash
# 备份单个数据库，推荐 custom 格式
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -F c -f g_api_$(date +%F_%H%M%S).dump
```

```bash
# 备份为普通 SQL 文件
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -f g_api_$(date +%F_%H%M%S).sql
```

```bash
# 压缩备份数据库
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -F c | gzip > g_api_$(date +%F_%H%M%S).dump.gz
```

```bash
# 只备份 app schema
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -n app -F c -f g_api_app_$(date +%F_%H%M%S).dump
```

```bash
# 只备份 app.users 表
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -t app.users -F c -f g_api_app_users_$(date +%F_%H%M%S).dump
```

```bash
# 备份角色和全局对象
pg_dumpall -h 127.0.0.1 -U postgres --globals-only > globals_$(date +%F_%H%M%S).sql
```

```bash
# 备份所有数据库
pg_dumpall -h 127.0.0.1 -U postgres > all_databases_$(date +%F_%H%M%S).sql
```

---

## 28. 恢复命令

```sql
-- 创建恢复库
CREATE DATABASE g_api_restore
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

```bash
# 恢复 custom 格式备份到恢复库
pg_restore -h 127.0.0.1 -p 5432 -U postgres -d g_api_restore g_api_2026-07-05_020000.dump
```

```bash
# 恢复前清理已有对象
pg_restore -h 127.0.0.1 -U postgres -d g_api_restore --clean --if-exists g_api_2026-07-05_020000.dump
```

```bash
# 恢复 SQL 文件
psql -h 127.0.0.1 -U postgres -d g_api_restore -f g_api_2026-07-05_020000.sql
```

```bash
# 恢复角色和全局对象
psql -h 127.0.0.1 -U postgres -f globals_2026-07-05_020000.sql
```

```bash
# 从完整备份中恢复指定表
pg_restore -h 127.0.0.1 -U postgres -d g_api_restore -t app.users g_api_2026-07-05_020000.dump
```

---

## 29. 恢复后检查

```bash
# 连接恢复库
psql -h 127.0.0.1 -U postgres -d g_api_restore
```

```sql
-- 查看 schema
\dn
```

```sql
-- 查看 app schema 下的表
\dt app.*
```

```sql
-- 查看 app.users 表结构
\d app.users
```

```sql
-- 查看 app.users 数据量
SELECT count(*) FROM app.users;
```

```sql
-- 查看 app.users 时间范围
SELECT min(created_at), max(created_at)
FROM app.users;
```

```sql
-- 查看 app schema 下的索引
\di app.*
```

```sql
-- 查看 app schema 下的序列
\ds app.*
```

```sql
-- 查看 app schema 下对象权限
\dp app.*
```

---

## 30. 常用组合示例

```sql
-- 新增字段
ALTER TABLE app.users
ADD COLUMN last_login_at TIMESTAMPTZ;
```

```sql
-- 给新字段加索引，生产大表推荐 CONCURRENTLY
CREATE INDEX CONCURRENTLY idx_users_last_login_at
ON app.users(last_login_at);
```

```sql
-- 查询 180 天未登录用户
SELECT id, email, last_login_at
FROM app.users
WHERE last_login_at < now() - interval '180 days';
```

```sql
-- 批量禁用 180 天未登录用户
UPDATE app.users
SET
  status = 'disabled',
  updated_at = now()
WHERE last_login_at < now() - interval '180 days';
```

```sql
-- 删除 30 天前的请求日志
DELETE FROM app.request_logs
WHERE created_at < now() - interval '30 days';
```

---

## 31. 最常用 psql 快捷命令

```sql
-- 查看数据库
\l
```

```sql
-- 切换数据库
\c g_api
```

```sql
-- 查看 schema
\dn
```

```sql
-- 查看 app schema 下的表
\dt app.*
```

```sql
-- 查看表结构
\d app.users
```

```sql
-- 查看表详细结构
\d+ app.users
```

```sql
-- 查看索引
\di app.*
```

```sql
-- 查看序列
\ds app.*
```

```sql
-- 查看用户
\du
```

```sql
-- 查看权限
\dp app.*
```

```sql
-- 查看连接信息
\conninfo
```

```sql
-- 开启执行时间
\timing
```

```sql
-- 开启扩展显示
\x
```

```sql
-- 查看帮助
\?
```

```sql
-- 退出
\q
```

---

## 32. MySQL 对比速查

```sql
-- MySQL: SHOW DATABASES;
-- PostgreSQL:
\l
```

```sql
-- MySQL: USE g_api;
-- PostgreSQL:
\c g_api
```

```sql
-- MySQL: SHOW TABLES;
-- PostgreSQL:
\dt
```

```sql
-- MySQL: DESC users;
-- PostgreSQL:
\d users
```

```sql
-- MySQL: SHOW INDEX FROM users;
-- PostgreSQL:
\d users
```

```sql
-- MySQL: SELECT DATABASE();
-- PostgreSQL:
SELECT current_database();
```

```sql
-- MySQL: SELECT USER();
-- PostgreSQL:
SELECT current_user;
```

```sql
-- MySQL: exit;
-- PostgreSQL:
\q
```

