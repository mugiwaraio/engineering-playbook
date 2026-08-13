[← 返回索引](./index.md)

# PostgreSQL 字符集、大小写、时区、备份恢复命令速查

## 1. 查看数据库字符集

```sql
SELECT
  datname,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate,
  datctype
FROM pg_database;
```

查看当前数据库：

```sql
SELECT
  datname,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate,
  datctype
FROM pg_database
WHERE datname = current_database();
```

查看客户端编码：

```sql
SHOW client_encoding;
```

设置当前会话客户端编码：

```sql
SET client_encoding = 'UTF8';
```

---

## 2. 创建数据库时指定字符集

推荐：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

如果系统不支持 `C.UTF-8`，可用：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'en_US.UTF-8'
  LC_CTYPE = 'en_US.UTF-8'
  TEMPLATE = template0;
```

查看系统支持的 collation：

```sql
SELECT * FROM pg_collation;
```

简化查看：

```sql
SELECT collname, collcollate, collctype
FROM pg_collation
ORDER BY collname;
```

---

## 3. 查看当前时区

```sql
SHOW timezone;
```

查看当前时间：

```sql
SELECT now();
```

查看 UTC 时间：

```sql
SELECT now() AT TIME ZONE 'UTC';
```

查看指定时区时间：

```sql
SELECT now() AT TIME ZONE 'Asia/Tokyo';
```

```sql
SELECT now() AT TIME ZONE 'Asia/Shanghai';
```

---

## 4. 设置数据库时区

设置数据库默认时区为 UTC：

```sql
ALTER DATABASE g_api SET timezone TO 'UTC';
```

设置用户默认时区为 UTC：

```sql
ALTER ROLE g_api_user SET timezone TO 'UTC';
```

设置当前会话时区：

```sql
SET timezone TO 'UTC';
```

查看某个数据库配置（`ALTER DATABASE ... SET` 的配置存放在 `pg_db_role_setting`，
不在 `pg_database`；`pg_database.datconfig` 列自 PostgreSQL 9.0 起已移除。
`setrole = 0` 表示对该库所有角色生效的库级配置）：

```sql
SELECT d.datname, s.setconfig
FROM pg_db_role_setting s
JOIN pg_database d ON d.oid = s.setdatabase
WHERE d.datname = 'g_api' AND s.setrole = 0;
```

查看某个用户配置：

```sql
SELECT rolname, rolconfig
FROM pg_roles
WHERE rolname = 'g_api_user';
```

---

## 5. 时间字段推荐写法

建表推荐：

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  username TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

更新时手动维护 `updated_at`：

```sql
UPDATE app.users
SET
  username = 'tom',
  updated_at = now()
WHERE id = 1;
```

---

## 6. 查看表名、字段名大小写

查看 schema：

```sql
\dn
```

查看表：

```sql
\dt app.*
```

查看表结构：

```sql
\d app.users
```

查看所有表：

```sql
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

查看字段：

```sql
SELECT
  table_schema,
  table_name,
  column_name,
  data_type
FROM information_schema.columns
WHERE table_schema = 'app'
ORDER BY table_name, ordinal_position;
```

---

## 7. 大小写命名示例

推荐：

```sql
CREATE TABLE app.user_profiles (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL,
  display_name TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

不推荐：

```sql
CREATE TABLE app."UserProfiles" (
  "ID" BIGINT,
  "UserName" TEXT
);
```

如果用了双引号，查询也必须带双引号：

```sql
SELECT "ID", "UserName"
FROM app."UserProfiles";
```

---

## 8. 字符串大小写查询

大小写敏感查询：

```sql
SELECT *
FROM app.users
WHERE username = 'Tom';
```

大小写不敏感查询：

```sql
SELECT *
FROM app.users
WHERE username ILIKE 'tom';
```

使用 `lower()`：

```sql
SELECT *
FROM app.users
WHERE lower(username) = lower('Tom');
```

邮箱唯一索引推荐：

```sql
CREATE UNIQUE INDEX uk_users_email_lower
ON app.users (lower(email));
```

查询：

```sql
SELECT *
FROM app.users
WHERE lower(email) = lower('Tom@Example.com');
```

---

# 9. 备份数据库

## 9.1 备份单个数据库，推荐 custom 格式

```bash
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -F c -f g_api_$(date +%F_%H%M%S).dump
```

## 9.2 备份为 SQL 文件

```bash
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -f g_api_$(date +%F_%H%M%S).sql
```

## 9.3 压缩备份

custom 格式（`-F c`）默认已用 zlib 压缩，再叠加 gzip 收益很小，建议直接用内置压缩级别：

```bash
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -F c -Z 9 -f g_api_$(date +%F_%H%M%S).dump
```

## 9.4 备份指定 schema

```bash
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -n app -F c -f g_api_app_$(date +%F_%H%M%S).dump
```

## 9.5 备份指定表

```bash
pg_dump -h 127.0.0.1 -p 5432 -U postgres -d g_api -t app.users -F c -f g_api_app_users_$(date +%F_%H%M%S).dump
```

## 9.6 备份角色和全局对象

```bash
pg_dumpall -h 127.0.0.1 -U postgres --globals-only > globals_$(date +%F_%H%M%S).sql
```

## 9.7 备份所有数据库

```bash
pg_dumpall -h 127.0.0.1 -U postgres > all_databases_$(date +%F_%H%M%S).sql
```

---

# 10. 恢复数据库

## 10.1 创建恢复库

```sql
CREATE DATABASE g_api_restore
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

## 10.2 恢复 custom 格式备份

```bash
pg_restore -h 127.0.0.1 -p 5432 -U postgres -d g_api_restore g_api_2026-07-05_020000.dump
```

## 10.3 恢复前清理已有对象

```bash
pg_restore -h 127.0.0.1 -U postgres -d g_api_restore --clean --if-exists g_api_2026-07-05_020000.dump
```

## 10.4 恢复 SQL 文件

```bash
psql -h 127.0.0.1 -U postgres -d g_api_restore -f g_api_2026-07-05_020000.sql
```

## 10.5 恢复角色权限

```bash
psql -h 127.0.0.1 -U postgres -f globals_2026-07-05_020000.sql
```

## 10.6 恢复指定表

```bash
pg_restore -h 127.0.0.1 -U postgres -d g_api_restore -t app.users g_api_2026-07-05_020000.dump
```

---

# 11. 恢复后检查命令

连接恢复库：

```bash
psql -h 127.0.0.1 -U postgres -d g_api_restore
```

查看 schema：

```sql
\dn
```

查看表：

```sql
\dt app.*
```

查看表结构：

```sql
\d app.users
```

查看数据量：

```sql
SELECT count(*) FROM app.users;
```

查看时间范围：

```sql
SELECT min(created_at), max(created_at)
FROM app.users;
```

查看索引：

```sql
\di app.*
```

查看序列：

```sql
SELECT
  sequence_schema,
  sequence_name
FROM information_schema.sequences
WHERE sequence_schema = 'app';
```

查看权限：

```sql
\dp app.*
```

---

# 12. 发布前备份命令

发布前备份数据库：

```bash
pg_dump -h 127.0.0.1 -U postgres -d g_api -F c -f g_api_before_release_$(date +%F_%H%M%S).dump
```

备份角色：

```bash
pg_dumpall -h 127.0.0.1 -U postgres --globals-only > globals_before_release_$(date +%F_%H%M%S).sql
```

---

# 13. 常用目录和文件名

备份目录：

```bash
mkdir -p /data/backups/postgresql/g_api/daily
mkdir -p /data/backups/postgresql/g_api/weekly
mkdir -p /data/backups/postgresql/g_api/monthly
```

文件名示例：

```text
g_api_full_2026-07-05_020000.dump
g_api_app_2026-07-05_020000.dump
g_api_app_users_2026-07-05_020000.dump
globals_2026-07-05_020000.sql
```

---

# 14. 最小命令清单

查看字符集：

```sql
SELECT datname, pg_encoding_to_char(encoding), datcollate, datctype
FROM pg_database;
```

查看时区：

```sql
SHOW timezone;
```

设置数据库时区：

```sql
ALTER DATABASE g_api SET timezone TO 'UTC';
```

设置用户时区：

```sql
ALTER ROLE g_api_user SET timezone TO 'UTC';
```

备份数据库：

```bash
pg_dump -h 127.0.0.1 -U postgres -d g_api -F c -f g_api_$(date +%F_%H%M%S).dump
```

恢复数据库：

```bash
pg_restore -h 127.0.0.1 -U postgres -d g_api_restore g_api_2026-07-05_020000.dump
```

备份角色：

```bash
pg_dumpall -h 127.0.0.1 -U postgres --globals-only > globals_$(date +%F_%H%M%S).sql
```

查看表权限：

```sql
\dp app.*
```

