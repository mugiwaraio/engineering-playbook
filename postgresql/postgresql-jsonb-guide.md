[← 返回索引](./index.md)

# PostgreSQL JSONB 使用规范

## 1. JSONB 是什么

`JSONB` 是 PostgreSQL 中用于存储 JSON 数据的字段类型。

可以理解为：

```text
JSONB = 可以存 JSON 数据，并且支持查询、索引和过滤的字段类型
```

适合存放：

```text
1. 结构不固定的数据
2. 扩展字段
3. 配置信息
4. 标签信息
5. 第三方接口返回结果
6. 请求参数快照
7. 元数据 metadata
```

---

## 2. JSON 和 JSONB 的区别

PostgreSQL 里有两个类似类型：

```text
JSON
JSONB
```

| 类型    | 说明                      | 推荐程度  |
| ----- | ----------------------- | ----- |
| JSON  | 按原始文本格式保存 JSON          | 一般不推荐 |
| JSONB | 按二进制结构保存 JSON，支持索引和高效查询 | 推荐使用  |

正式项目中，默认推荐使用：

```sql
JSONB
```

不建议优先使用：

```sql
JSON
```

简单记忆：

```text
PostgreSQL 中需要存 JSON，优先使用 JSONB。
```

---

## 3. JSONB 适合的业务场景

JSONB 适合用于补充关系型表结构，而不是替代表结构。

适合场景：

```text
1. 用户扩展资料
2. 文章 metadata
3. 商品扩展属性
4. API 请求体快照
5. API 响应体快照
6. 系统配置
7. 标签数组
8. 第三方平台返回结果
9. 不固定字段
10. 灰度配置、实验配置
```

示例：

```text
用户基础字段固定：
id、username、email、created_at

用户扩展字段不固定：
年龄、城市、偏好、标签、来源渠道
```

可以设计成：

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  username TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  profile JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 4. JSONB 不适合的场景

不要把所有字段都塞进 JSONB。

以下字段建议单独建列：

```text
1. 经常用于 WHERE 查询的字段
2. 经常用于 ORDER BY 排序的字段
3. 经常用于 JOIN 的字段
4. 有强约束要求的字段
5. 核心业务字段
6. 金额字段
7. 状态字段
8. 用户 ID
9. 订单 ID
10. 创建时间
```

不推荐设计：

```sql
CREATE TABLE app.orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  data JSONB
);
```

不推荐把订单核心字段都放在 JSONB 中：

```json
{
  "user_id": 1001,
  "amount": 99.99,
  "status": "paid",
  "created_at": "2026-07-05"
}
```

推荐设计：

```sql
CREATE TABLE app.orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL,
  amount NUMERIC(10,2) NOT NULL,
  status TEXT NOT NULL,
  extra JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

原则：

```text
核心字段单独建列
扩展字段放 JSONB
```

---

## 5. JSONB 建表示例

## 5.1 用户扩展信息

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  username TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  profile JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

插入示例：

```sql
INSERT INTO app.users (username, email, profile)
VALUES (
  'tom',
  'tom@example.com',
  '{"age": 18, "city": "Tokyo", "vip": true, "tags": ["developer", "aws"]}'
);
```

---

## 5.2 文章标签和元数据

```sql
CREATE TABLE app.articles (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title TEXT NOT NULL,
  tags JSONB DEFAULT '[]',
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

插入示例：

```sql
INSERT INTO app.articles (title, tags, metadata)
VALUES (
  'PostgreSQL 入门',
  '["postgresql", "database", "backend"]',
  '{"author": "Jesse", "views": 1000, "published": true}'
);
```

说明：

```text
tags 是 JSON 数组
metadata 是 JSON 对象
```

---

## 5.3 API 请求日志

```sql
CREATE TABLE app.api_requests (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL,
  path TEXT NOT NULL,
  method TEXT NOT NULL,
  status_code INT,
  cost_ms INT,
  request_body JSONB DEFAULT '{}',
  response_body JSONB DEFAULT '{}',
  extra JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

说明：

```text
user_id、path、method、status_code、cost_ms、created_at 是核心字段
request_body、response_body、extra 是扩展 JSONB 字段
```

---

## 6. JSONB 查询语法

## 6.1 查询整个 JSONB 字段

```sql
SELECT title, metadata
FROM app.articles;
```

---

## 6.2 查询 JSONB 中某个字段

使用 `->>` 取文本值：

```sql
SELECT
  title,
  metadata->>'author' AS author
FROM app.articles;
```

常用记法：

```text
->   取出来还是 JSON
->>  取出来是文本
```

示例：

```sql
SELECT
  metadata->'author' AS author_json,
  metadata->>'author' AS author_text
FROM app.articles;
```

区别：

```text
metadata->'author'   返回 JSON 值
metadata->>'author'  返回文本值
```

---

## 6.3 查询 JSONB 数字字段

因为 `->>` 取出来是文本，如果要按数字比较，需要转换类型。

```sql
SELECT *
FROM app.articles
WHERE (metadata->>'views')::int > 500;
```

说明：

```text
metadata->>'views'  取出 views
::int               转成整数
> 500               做数字比较
```

---

## 6.4 查询 JSONB 布尔字段

```sql
SELECT *
FROM app.articles
WHERE (metadata->>'published')::boolean = true;
```

或者使用包含查询：

```sql
SELECT *
FROM app.articles
WHERE metadata @> '{"published": true}';
```

---

## 7. JSONB 条件查询

## 7.1 查询某个 key 的值

查询 `metadata` 中 `author = Jesse` 的数据：

```sql
SELECT *
FROM app.articles
WHERE metadata->>'author' = 'Jesse';
```

---

## 7.2 判断是否包含某个 JSON 内容

`@>` 表示包含。

```sql
SELECT *
FROM app.articles
WHERE metadata @> '{"published": true}';
```

含义：

```text
查询 metadata 里面包含 published = true 的文章
```

---

## 7.3 查询 JSONB 数组是否包含某个元素

```sql
SELECT *
FROM app.articles
WHERE tags @> '["postgresql"]';
```

如果 `tags` 内容是：

```json
["postgresql", "database", "backend"]
```

这条 SQL 可以命中。

---

## 7.4 查询 JSONB 是否存在某个 key

```sql
SELECT *
FROM app.articles
WHERE metadata ? 'author';
```

含义：

```text
查询 metadata 中存在 author 这个 key 的数据
```

---

## 7.5 查询 JSONB 是否存在任意一个 key

```sql
SELECT *
FROM app.articles
WHERE metadata ?| array['author', 'source'];
```

含义：

```text
metadata 中存在 author 或 source 任意一个 key 即可
```

---

## 7.6 查询 JSONB 是否同时存在多个 key

```sql
SELECT *
FROM app.articles
WHERE metadata ?& array['author', 'views'];
```

含义：

```text
metadata 中必须同时存在 author 和 views
```

---

## 8. JSONB 更新语法

## 8.1 整体更新 JSONB 字段

```sql
UPDATE app.articles
SET metadata = '{"author": "Jesse", "views": 2000, "published": true}'
WHERE id = 1;
```

说明：

```text
这种方式会整体替换 metadata 字段。
```

---

## 8.2 只更新 JSONB 中某个字段

使用 `jsonb_set`。

```sql
UPDATE app.articles
SET metadata = jsonb_set(metadata, '{views}', '2000')
WHERE id = 1;
```

说明：

```text
metadata      要更新的 JSONB 字段
{views}       要更新的 key
'2000'        新值，必须是 JSONB 格式
```

---

## 8.3 更新字符串字段

如果更新 JSONB 中的字符串，要注意双引号。

```sql
UPDATE app.articles
SET metadata = jsonb_set(metadata, '{author}', '"Tom"')
WHERE id = 1;
```

注意：

```text
'"Tom"' 是 JSONB 字符串格式
```

---

## 8.4 更新嵌套字段

假设 metadata 内容：

```json
{
  "seo": {
    "title": "Old Title",
    "keywords": ["postgresql", "database"]
  }
}
```

更新 `seo.title`：

```sql
UPDATE app.articles
SET metadata = jsonb_set(metadata, '{seo,title}', '"New Title"')
WHERE id = 1;
```

---

## 8.5 新增 JSONB 字段

使用 `||` 合并 JSONB。

```sql
UPDATE app.articles
SET metadata = metadata || '{"source": "markdown"}'
WHERE id = 1;
```

原来：

```json
{
  "author": "Jesse",
  "views": 1000
}
```

更新后：

```json
{
  "author": "Jesse",
  "views": 1000,
  "source": "markdown"
}
```

---

## 8.6 删除 JSONB 中的字段

```sql
UPDATE app.articles
SET metadata = metadata - 'source'
WHERE id = 1;
```

含义：

```text
删除 metadata 中的 source 这个 key。
```

---

## 8.7 删除嵌套字段

```sql
UPDATE app.articles
SET metadata = metadata #- '{seo,title}'
WHERE id = 1;
```

含义：

```text
删除 metadata.seo.title 字段。
```

---

## 9. JSONB 索引规范

如果 JSONB 字段需要经常查询，建议加索引。

## 9.1 给整个 JSONB 字段加 GIN 索引

```sql
CREATE INDEX idx_articles_metadata
ON app.articles
USING GIN (metadata);
```

适合查询：

```sql
SELECT *
FROM app.articles
WHERE metadata @> '{"published": true}';
```

---

## 9.2 给 JSONB 数组字段加 GIN 索引

```sql
CREATE INDEX idx_articles_tags
ON app.articles
USING GIN (tags);
```

适合查询：

```sql
SELECT *
FROM app.articles
WHERE tags @> '["postgresql"]';
```

---

## 9.3 给 JSONB 中某个字段加表达式索引

如果经常按 author 查询：

```sql
CREATE INDEX idx_articles_author
ON app.articles ((metadata->>'author'));
```

适合查询：

```sql
SELECT *
FROM app.articles
WHERE metadata->>'author' = 'Jesse';
```

---

## 9.4 给 JSONB 中数字字段加表达式索引

如果经常按 views 查询：

```sql
CREATE INDEX idx_articles_views
ON app.articles (((metadata->>'views')::int));
```

适合查询：

```sql
SELECT *
FROM app.articles
WHERE (metadata->>'views')::int > 1000;
```

---

## 10. JSONB 与 MySQL JSON 对比

| 场景         | MySQL        | PostgreSQL     |
| ---------- | ------------ | -------------- |
| 存 JSON     | JSON         | JSONB          |
| 查询 JSON 字段 | JSON_EXTRACT | -> / ->>       |
| JSON 包含查询  | 支持，但写法相对复杂   | @> 很常用         |
| JSON 索引    | 相对麻烦         | GIN 索引方便       |
| 标签数组查询     | 不太方便         | JSONB + GIN 方便 |
| 推荐程度       | 适合基础 JSON 存储 | 更适合复杂 JSON 查询  |

MySQL 查询：

```sql
SELECT *
FROM articles
WHERE JSON_EXTRACT(metadata, '$.author') = 'Jesse';
```

PostgreSQL 查询：

```sql
SELECT *
FROM app.articles
WHERE metadata->>'author' = 'Jesse';
```

---

## 11. 正式项目 JSONB 设计原则

正式项目建议遵循：

```text
核心字段：单独建列
扩展字段：使用 JSONB
经常查询的 JSONB 字段：单独建索引
不确定是否长期稳定的字段：先放 JSONB
稳定后高频使用的字段：再升级为正式列
```

---

## 12. 推荐字段命名

常见 JSONB 字段命名：

| 字段名           | 使用场景   |
| ------------- | ------ |
| extra         | 通用扩展字段 |
| metadata      | 元数据    |
| profile       | 用户扩展资料 |
| attributes    | 商品属性   |
| settings      | 配置项    |
| request_body  | 请求体    |
| response_body | 响应体    |
| payload       | 消息内容   |
| tags          | 标签数组   |

示例：

```sql
CREATE TABLE app.products (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL,
  price NUMERIC(10,2) NOT NULL,
  attributes JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 13. 推荐默认值

JSON 对象类型：

```sql
extra JSONB DEFAULT '{}'
```

JSON 数组类型：

```sql
tags JSONB DEFAULT '[]'
```

推荐：

```sql
CREATE TABLE app.articles (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title TEXT NOT NULL,
  tags JSONB DEFAULT '[]',
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 14. 使用注意事项

## 14.1 不要滥用 JSONB

不推荐：

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  data JSONB
);
```

推荐：

```sql
CREATE TABLE app.users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  username TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  profile JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 14.2 高频查询字段不要长期藏在 JSONB 里

如果某个 JSONB 字段经常被查询，例如：

```sql
WHERE metadata->>'status' = 'active'
```

长期来看更推荐升级为正式字段：

```sql
status TEXT NOT NULL
```

JSONB 更适合扩展字段，不适合承载所有核心查询字段。

---

## 14.3 JSONB 字段要控制大小

不要把过大的内容长期塞进 JSONB。

例如：

```text
超大响应体
大文件内容
完整 HTML
完整日志全文
```

更推荐：

```text
大内容放对象存储，例如 S3
数据库里只保存 URL、key、摘要信息
```

---

## 14.4 JSONB 仍然需要数据规范

虽然 JSONB 很灵活，但仍然建议约定字段结构。

例如：

```json
{
  "source": "github",
  "trace_id": "abc123",
  "client_ip": "1.1.1.1"
}
```

不要同一个字段里混乱存储：

```json
{
  "source": "github"
}
```

另一条数据却写成：

```json
{
  "from": "github"
}
```

字段命名要统一。

---

## 15. 最常用 SQL 速查

## 15.1 创建 JSONB 字段

```sql
CREATE TABLE app.articles (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title TEXT NOT NULL,
  tags JSONB DEFAULT '[]',
  metadata JSONB DEFAULT '{}'
);
```

---

## 15.2 插入 JSONB

```sql
INSERT INTO app.articles (title, tags, metadata)
VALUES (
  'PostgreSQL 入门',
  '["postgresql", "database"]',
  '{"author": "Jesse", "views": 1000}'
);
```

---

## 15.3 读取 JSONB 字段

```sql
SELECT metadata->>'author'
FROM app.articles;
```

---

## 15.4 JSONB 条件查询

```sql
SELECT *
FROM app.articles
WHERE metadata->>'author' = 'Jesse';
```

---

## 15.5 JSONB 包含查询

```sql
SELECT *
FROM app.articles
WHERE metadata @> '{"published": true}';
```

---

## 15.6 JSONB 数组包含查询

```sql
SELECT *
FROM app.articles
WHERE tags @> '["postgresql"]';
```

---

## 15.7 更新 JSONB 字段

```sql
UPDATE app.articles
SET metadata = jsonb_set(metadata, '{views}', '2000')
WHERE id = 1;
```

---

## 15.8 新增 JSONB 字段

```sql
UPDATE app.articles
SET metadata = metadata || '{"source": "markdown"}'
WHERE id = 1;
```

---

## 15.9 删除 JSONB 字段

```sql
UPDATE app.articles
SET metadata = metadata - 'source'
WHERE id = 1;
```

---

## 15.10 创建 JSONB GIN 索引

```sql
CREATE INDEX idx_articles_metadata
ON app.articles
USING GIN (metadata);
```

---

## 16. 最小记忆版

```text
JSONB = PostgreSQL 中推荐使用的 JSON 存储类型
```

正式项目记住：

```text
1. 核心字段不要放 JSONB
2. 不固定字段可以放 JSONB
3. 标签、配置、metadata 适合放 JSONB
4. 高频查询的 JSONB 字段要加索引
5. JSONB 查询常用 ->、->>、@>
6. JSONB 是补充表设计，不是替代表设计
```

三个最常用语法：

```sql
-- 取字段文本值
metadata->>'author'

-- 判断 JSON 对象包含
metadata @> '{"published": true}'

-- 判断 JSON 数组包含
tags @> '["postgresql"]'
```

---

## 17. 最终结论

JSONB 不是用来替代表设计的，而是用来补充表设计。

推荐设计方式：

```text
核心字段单独建列
扩展字段使用 JSONB
高频 JSONB 查询字段加索引
长期稳定且高频使用的 JSONB 字段升级为正式列
```

一句话总结：

```text
JSONB 适合存不固定、扩展性强、半结构化的数据；不适合存核心业务主字段。
```

