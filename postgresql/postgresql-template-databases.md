[← 返回索引](./index.md)

# PostgreSQL template0 和 template1 使用说明

## 1. template0 和 template1 是什么

PostgreSQL 创建数据库时，本质上是从一个已有的模板数据库复制出一个新数据库。

PostgreSQL 默认自带两个模板库：

```text
template0
template1
```

简单理解：

```text
template0 = 系统原始干净模板
template1 = 默认创建数据库使用的模板
```

默认执行：

```sql
CREATE DATABASE g_api;
```

大致等价于：

```sql
CREATE DATABASE g_api TEMPLATE template1;
```

如果指定：

```sql
CREATE DATABASE g_api TEMPLATE template0;
```

表示从 `template0` 这个干净模板创建数据库。

---

## 2. template0 和 template1 区别

| 模板库       | 说明                | 是否默认使用 | 是否建议修改            |
| --------- | ----------------- | ------ | ----------------- |
| template0 | PostgreSQL 原始干净模板 | 否      | 不建议修改             |
| template1 | 默认创建数据库使用的模板      | 是      | 可以修改，但正式项目不建议随意修改 |

简单记忆：

```text
template0：出厂干净母版
template1：默认工作母版
```

---

## 3. template0 的作用

`template0` 是 PostgreSQL 保留的干净模板，常用于创建明确指定字符集和排序规则的新数据库。

推荐场景：

```text
1. 创建正式生产数据库
2. 创建测试数据库
3. 创建开发数据库
4. 创建恢复数据库
5. 需要明确指定 ENCODING、LC_COLLATE、LC_CTYPE
```

推荐命令：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

说明：

```text
ENCODING：数据库字符编码
LC_COLLATE：字符串排序规则
LC_CTYPE：字符分类规则
TEMPLATE：指定从哪个模板创建数据库
```

---

## 4. template1 的作用

`template1` 是 PostgreSQL 默认使用的模板。

当你执行：

```sql
CREATE DATABASE g_api;
```

PostgreSQL 默认从 `template1` 复制数据库。

也可以显式指定：

```sql
CREATE DATABASE g_api TEMPLATE template1;
```

如果 `template1` 中安装了扩展、创建了 schema、函数或其他对象，以后通过默认方式创建的新数据库都会继承这些内容。

例如，如果你在 `template1` 里创建了：

```sql
CREATE EXTENSION pgcrypto;
```

那么以后执行：

```sql
CREATE DATABASE g_api;
```

新数据库就会自动带上 `pgcrypto` 扩展（从模板复制时扩展会一并继承，是确定行为，不是偶发）。

---

## 5. 正式项目推荐使用方式

正式项目推荐：

```text
创建数据库时使用 template0
不要污染 template0
不要随意修改 template1
业务结构通过 SQL 初始化脚本或 migration 工具管理
```

推荐流程：

```text
1. 使用 template0 创建干净数据库
2. 创建 app schema
3. 创建应用用户和权限
4. 执行 migration 脚本创建表、索引、函数
```

示例：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

然后执行：

```text
001_create_schema.sql
002_create_users.sql
003_create_tables.sql
004_create_indexes.sql
```

---

## 6. 什么时候使用 template0

推荐使用 `template0` 的情况：

```text
1. 正式项目新建数据库
2. 需要指定字符集
3. 需要指定排序规则
4. 创建恢复库
5. 希望数据库从干净模板开始
6. 不希望继承 template1 中已有对象
```

示例：创建生产库

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

示例：创建测试库

```sql
CREATE DATABASE g_api_test
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

示例：创建恢复库

```sql
CREATE DATABASE g_api_restore
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

---

## 7. 什么时候使用 template1

可以使用 `template1` 的情况：

```text
1. 临时测试数据库
2. 不关心字符集和排序规则
3. 希望继承 template1 中已有配置
4. 本地开发环境快速创建数据库
```

示例：

```sql
CREATE DATABASE test_db;
```

等价于：

```sql
CREATE DATABASE test_db TEMPLATE template1;
```

但正式项目不推荐依赖 `template1` 中的隐含对象。

原因：

```text
1. template1 可能被修改过
2. 新库继承内容不够直观
3. 多环境可能不一致
4. 不如 SQL 初始化脚本清晰
```

---

## 8. 为什么指定字符集时推荐 template0

PostgreSQL 创建数据库时，新数据库的字符集、排序规则通常要和模板数据库兼容。

如果使用默认 `template1`，而 `template1` 的字符集或排序规则和你指定的不一致，可能创建失败。

所以，当需要明确指定：

```text
ENCODING
LC_COLLATE
LC_CTYPE
```

建议使用：

```sql
TEMPLATE = template0
```

标准写法：

```sql
CREATE DATABASE 数据库名
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

---

## 9. 查看 template0 和 template1

查看模板库：

```sql
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

查看所有模板数据库：

```sql
SELECT
  datname,
  datistemplate,
  datallowconn,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate,
  datctype
FROM pg_database
WHERE datistemplate = true;
```

重点字段说明：

```text
datname：数据库名
datistemplate：是否为模板库
datallowconn：是否允许连接
encoding：字符集
datcollate：排序规则
datctype：字符分类规则
```

---

## 10. template0 是否可以连接

默认情况下，`template0` 通常不允许连接。

查看：

```sql
SELECT
  datname,
  datallowconn
FROM pg_database
WHERE datname IN ('template0', 'template1');
```

常见结果：

```text
template0 | false
template1 | true
```

不建议修改 `template0` 的连接状态，也不建议往 `template0` 中安装扩展、建表或写入对象。

---

## 11. template1 是否可以修改

`template1` 可以修改，但正式项目不建议随意修改。

例如，你可以连接 `template1`：

```bash
psql -U postgres -d template1
```

然后安装扩展：

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

以后默认创建的新库：

```sql
CREATE DATABASE test_db;
```

会继承 `pgcrypto`。

但是正式项目不推荐这么做，原因是：

```text
1. 隐式继承，不容易排查
2. 多环境 template1 可能不同
3. 数据库初始化过程不透明
4. 不利于版本管理
5. 不如 migration 脚本可控
```

更推荐：

```text
每个业务库创建后，显式执行初始化 SQL 或 migration。
```

---

## 12. C.UTF-8 不存在怎么办

不同操作系统和 PostgreSQL 安装环境支持的 locale 可能不同。

如果执行：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

报错，可以先查看可用 collation：

```sql
SELECT
  collname,
  collcollate,
  collctype
FROM pg_collation
ORDER BY collname;
```

如果没有 `C.UTF-8`，可以考虑使用：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'en_US.UTF-8'
  LC_CTYPE = 'en_US.UTF-8'
  TEMPLATE = template0;
```

也可以根据系统实际可用 locale 选择。

---

## 13. 是否建议自建 template

一般正式项目不建议自建业务 template。

更推荐：

```text
template0 + 初始化 SQL 脚本 + migration 工具
```

原因：

```text
1. template 修改不容易追踪
2. 新库继承了什么内容不够清晰
3. 多人协作容易产生隐性差异
4. 跨环境同步不如 SQL 脚本稳定
5. 业务结构频繁变化，不适合放进 template
```

可以考虑自建 template 的场景：

```text
1. 临时测试库
2. 沙箱库
3. 自动化测试库
4. 培训环境
5. 高频创建大量相同基础结构的开发库
```

但正式生产项目建议：

```text
不要依赖自建 template 管理业务表结构。
业务表结构应通过 migration 管理。
```

---

## 14. 使用注意事项

## 14.1 不要修改 template0

不建议：

```sql
UPDATE pg_database
SET datallowconn = true
WHERE datname = 'template0';
```

不建议在 `template0` 中创建：

```text
schema
table
function
extension
业务数据
```

---

## 14.2 不要随意污染 template1

不建议把项目专属内容放到 `template1`：

```text
项目 schema
业务表
业务函数
业务权限
测试数据
```

如果修改了 `template1`，后续所有默认新建数据库都会继承这些内容。

---

## 14.3 正式项目使用显式初始化脚本

推荐：

```text
CREATE DATABASE 使用 template0
数据库对象使用 migration 创建
权限使用授权脚本管理
```

示例：

```text
db/
├── init/
│   ├── 001_create_database.sql
│   ├── 002_create_schema.sql
│   └── 003_create_roles.sql
└── migration/
    ├── 001_create_users.sql
    ├── 002_create_api_keys.sql
    └── 003_create_indexes.sql
```

---

## 15. 常用命令速查

### 15.1 使用 template0 创建正式数据库

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

### 15.2 使用 template1 创建数据库

```sql
CREATE DATABASE test_db;
```

等价于：

```sql
CREATE DATABASE test_db TEMPLATE template1;
```

### 15.3 查看 template0 和 template1

```sql
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

### 15.4 查看所有模板库

```sql
SELECT
  datname,
  datistemplate,
  datallowconn
FROM pg_database
WHERE datistemplate = true;
```

### 15.5 查看可用 collation

```sql
SELECT
  collname,
  collcollate,
  collctype
FROM pg_collation
ORDER BY collname;
```

### 15.6 连接 template1

```bash
psql -U postgres -d template1
```

### 15.7 查看 template1 中的扩展

```sql
\dx
```

---

## 16. 推荐标准

正式项目统一推荐：

```text
1. template0 保持干净，不修改
2. template1 不放项目专属内容
3. 创建正式库时使用 template0
4. 字符集统一 UTF8
5. 排序规则统一指定
6. 业务 schema、表、索引用 migration 管理
7. 不依赖 template1 的隐式继承
8. 一般不自建业务 template
```

推荐创建数据库命令：

```sql
CREATE DATABASE g_api
WITH
  ENCODING = 'UTF8'
  LC_COLLATE = 'C.UTF-8'
  LC_CTYPE = 'C.UTF-8'
  TEMPLATE = template0;
```

---

## 17. 最小记忆版

```text
template0：
系统干净模板，正式项目创建数据库时推荐使用。

template1：
默认模板，不指定 TEMPLATE 时默认使用。

正式项目：
使用 template0 创建干净数据库，后续结构通过 SQL 脚本或 migration 管理。

临时测试：
可以直接 CREATE DATABASE test_db，默认使用 template1。
```

一句话总结：

```text
template0 用于干净、可控地创建数据库；template1 是默认模板，但正式项目不要依赖它做隐式初始化。
```

