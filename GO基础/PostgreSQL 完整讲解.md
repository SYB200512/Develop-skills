# PostgreSQL 完整讲解

## 一、PostgreSQL 是什么

PostgreSQL（简称 PG），是一款**开源、企业级关系型数据库**。 MySQL 侧重易用、简单，互联网业务广泛； PostgreSQL 功能更强大，标准 SQL 支持更好，内置很多高级数据类型和能力，适合复杂查询、GIS 地理数据、JSON 业务、树形结构等场景。

> 默认端口：`5432`（MySQL 是 3306）

## 二、PG 核心优势（重点特性）

### 1. JSONB（最常用，面试高频）

PG 有两种 json 类型：

1. `json`：纯文本存储，读取时每次解析，**不能建索引**，性能差
2. `jsonb`：**二进制 JSON 存储**，写入时就解析、结构化存储，**支持建立索引，可以直接在 JSON 内部字段做查询、过滤**。

 示例表：

```sql
CREATE TABLE items (
    id BIGSERIAL PRIMARY KEY,
    data JSONB NOT NULL
);

-- 插入JSON数据
INSERT INTO items(data) VALUES ('{"name":"张三","type":"user","age":20}');

-- -> 取jsonb对象，返回jsonb类型
-- ->> 取出值，返回文本字符串（最常用）
SELECT data->>'name' FROM items WHERE data @> '{"type":"user"}';
```

- `data @> '{"type":"user"}'`：`@>` 是**包含运算符**，判断 jsonb 字段是否包含这个键值对，可以走索引！
- 可以给 jsonb 字段建 GIN 索引，大幅提升 JSON 内部字段查询性能：

```sql
CREATE INDEX idx_items_data ON items USING GIN(data);
```

> MySQL 也支持 JSON 类型，但 PG 的 JSONB 查询能力、索引支持要强很多。

### 2. 内置全文检索 TSVector / TSQuery

不需要额外部署 Elasticsearch，PG 原生支持全文搜索、分词、权重。

```sql
-- to_tsvector：把文本转为分词向量
-- to_tsquery：构建搜索词
SELECT title FROM article
WHERE to_tsvector('simple', title) @@ to_tsquery('PG');
```

适合轻量搜索场景，省去 ES 运维成本。

### 3. 原生数组、范围类型

MySQL 没有原生数组，只能用字符串逗号分隔存储（要自己拆分，很麻烦）。 PG 直接支持数组类型：

```sql
CREATE TABLE tags (
    id int primary key,
    tag_list text[] -- 字符串数组
);
INSERT INTO tags VALUES (1, ARRAY['Go','Gin','Postgres']);
-- 查询数组包含某个元素
SELECT * FROM tags WHERE 'Go' = ANY(tag_list);
```

范围类型：`int4range`、`daterange`，方便存储区间（时间范围、数字区间），支持范围重叠判断。

### 4. CTE WITH 表达式 + 递归 CTE

CTE 公用表表达式，可以把一段子查询单独抽出来，可读性强。 **递归 CTE**，用来遍历树形结构：组织架构、评论回复树、分类树，MySQL8.0 才支持，PG 很早就原生支持。

```sql
WITH RECURSIVE tree AS (
    -- 锚点成员：根节点
    SELECT id,name,parent_id FROM category WHERE parent_id IS NULL
    UNION ALL
    -- 递归部分：不断查找子节点
    SELECT c.id,c.name,c.parent_id FROM category c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

### 5. 并行查询

PG 查询优化器会自动把大查询拆分成多个任务，**利用多核 CPU 并行执行**，复杂大 SQL 性能提升明显。 MySQL 并行查询能力弱很多。

### 6. 逻辑复制

PG 的逻辑复制：基于 WAL 日志，**只复制表数据（逻辑层面）**，可以做到：

- 跨版本 PG 数据同步
- 部分表同步（只同步指定几张表，不是全库）
- 数据实时迁移、数据订阅、异构数据库同步

> MySQL 主从是 binlog 逻辑复制；PG 同时支持物理流复制（底层块复制，灾备）+ 逻辑复制（业务数据同步）。

## 三、PG 基础语法小区别（和 MySQL 对比，容易踩坑）

1. 自增主键：PG 用 `SERIAL` / `BIGSERIAL`，或者推荐 `GENERATED AS IDENTITY`；MySQL 是 `AUTO_INCREMENT`

```sql
id BIGSERIAL PRIMARY KEY
```

1. 字符串：`VARCHAR(n)`、`TEXT`，`TEXT`无长度限制，PG 推荐直接用 TEXT，性能和 VARCHAR 差别很小

2. 引号：

   双引号用来引用表名 / 字段名，单引号是字符串常量

   （MySQL 反引号 ``）

   - `SELECT "name" FROM user WHERE name='张三'`

3. 分页：同样 `LIMIT offset,count`，和 MySQL 一致

4. 事务：默认隔离级别 `READ COMMITTED`，也支持 RR；MVCC 机制 PG 和 MySQL InnoDB 思路不一样（PG 的 MVCC 是行版本，没有 undo log）

> ⚠️ PG MVCC：更新数据不会原地修改，新增一条新版本行，旧版本保留；旧版本数据叫**死元组 (dead tuple)**，需要`VACUUM`清理，否则表会膨胀。这是 PG 最经典的知识点！

## 四、索引

PG 支持多种索引类型，不止 B-Tree：

1. B-Tree：默认，等值、范围查询（和 MySQL B+Tree 类似，但 PG 是 B 树）
2. GIN 索引：倒排索引，适合 jsonb、数组、全文检索
3. GIST 索引：地理信息、模糊搜索
4. BRIN 索引：超大表，数据有序场景，占用空间极小

## 五、Go 中使用 PostgreSQL：pgx（课程重点）

Go 标准库 `database/sql` 是抽象层，底层需要驱动。 `pgx/v5` 现在官方推荐，性能更好，原生支持 PG 特性（jsonb、数组等），替代老的 lib/pq 驱动。

> 包：`github.com/jackc/pgx/v5/pgxpool`，`pgxpool` 是连接池，生产环境必须用连接池，不要单连接。

### 极简示例代码

```sql
package main

import (
	"context"
	"fmt"
	"github.com/jackc/pgx/v5/pgxpool"
)

func main() {
	// 连接串 postgres://用户:密码@地址:端口/数据库名
	dsn := "postgres://user:pass@localhost:5432/testdb"
	// 创建连接池
	pool, err := pgxpool.New(context.Background(), dsn)
	if err != nil {
		panic(err)
	}
	defer pool.Close() // 程序退出关闭池

	// 测试连通性
	err = pool.Ping(context.Background())
	if err != nil {
		panic(err)
	}

	// 查询jsonb示例
	rows, err := pool.Query(context.Background(),
		`SELECT data->>'name' FROM items WHERE data @> '{"type":"user"}'`)
	if err != nil {
		panic(err)
	}
	defer rows.Close()

	// 遍历结果
	for rows.Next() {
		var name string
		err = rows.Scan(&name)
		if err != nil {
			panic(err)
		}
		fmt.Println(name)
	}
}
```

要点：

1. 优先使用 `pgxpool` 连接池，复用连接，减少建立连接开销
2. 所有操作都需要传入 `context.Context`，可以做超时控制
3. 原生支持 PG 的 jsonb、数组类型，直接映射 Go 结构体

## 六、PG vs MySQL 快速对比（面试）

表格

| 特性      | MySQL(InnoDB)            | PostgreSQL                              |
| --------- | ------------------------ | --------------------------------------- |
| JSON      | JSON 类型，索引能力弱    | JSONB 二进制，GIN 索引强大              |
| 树形查询  | MySQL8.0 支持递归 CTE    | 很早就支持递归 CTE                      |
| 数组类型  | 无原生数组               | 原生数组、范围类型                      |
| 全文检索  | 内置弱，一般用 ES        | 内置 TSVector 全文检索                  |
| MVCC 实现 | undo log 多版本          | 新增行版本，VACUUM 清理死元组           |
| 并行查询  | 弱                       | 强大，自动多核并行                      |
| 适合场景  | 简单业务、高并发简单读写 | 复杂查询、GIS、JSON、树形数据、数据分析 |