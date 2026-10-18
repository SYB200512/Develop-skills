# Elasticsearch 学习文档

> 本文档根据学习目录整理，共 13 章，从安装到增删改查，再到高级搜索，循序渐进。
> 文档中的 Go 示例基于 `github.com/elastic/go-elasticsearch/v8`（TypedClient 风格），与本项目（go-blog）的 ES 使用方式一致。

---

## 目录

1. [ES 安装](#1-es-安装)
2. [ES 连接](#2-es-连接)
3. [ES 设置密码](#3-es-设置密码)
4. [Mapping 与创建索引](#4-mapping-与创建索引)
5. [ES 索引删除](#5-es-索引删除)
6. [ES 文档添加](#6-es-文档添加)
7. [根据 id 删除](#7-根据-id-删除)
8. [文档批量删除](#8-文档批量删除)
9. [ES 批量添加](#9-es-批量添加)
10. [ES 列表查询](#10-es-列表查询)
11. [模糊匹配与精确匹配](#11-模糊匹配与精确匹配)
12. [嵌套字段的搜索](#12-嵌套字段的搜索)
13. [ES 文档更新](#13-es-文档更新)

---

## 0. Elasticsearch 是什么（先看这里）

Elasticsearch（简称 ES）是一个**开源的分布式搜索与分析引擎**，底层使用 Lucene 实现。

一句话理解：**ES 就是一个"能搜索的数据库"**。

和传统数据库（MySQL）对比：

| 对比项 | MySQL | Elasticsearch |
|--------|-------|---------------|
| 数据组织 | 数据库 → 表 → 行 | 索引 → 文档 |
| 数据格式 | 行（二维表） | JSON 文档 |
| 查询方式 | SQL | RESTful API + JSON 查询体 |
| 擅长 | 事务、关系、复杂报表 | 全文搜索、模糊匹配、海量数据聚合 |
| 端口 | 3306 | 9200（HTTP） |

核心概念映射：

| MySQL | Elasticsearch |
|-------|---------------|
| 数据库 Database | 索引 Index |
| 表 Table | 类型 Type（7.x 后废弃，一个索引默认一种类型 `_doc`） |
| 行 Row | 文档 Document |
| 列 Column | 字段 Field |
| 表结构定义 DDL | 映射 Mapping |

**为什么本项目（博客）要用 ES？**
因为博客文章有大量文本内容（标题、正文），用户需要**全文搜索**（输入关键词找出相关文章）。MySQL 的 `LIKE '%关键词%'` 无法做相关性排序、分词匹配，而 ES 天生为全文搜索设计。本项目把文章同步到 ES 的 `go-blog_article_index` 索引中，供前端搜索使用。

> 术语速记：
> - **索引（Index）**：一类文档的集合，相当于 MySQL 的"表"
> - **文档（Document）**：一条数据，是 JSON 对象，相当于"一行"
> - **映射（Mapping）**：定义字段的类型和分析方式，相当于"表结构"
> - **分片（Shard）**：一个索引的数据被拆成多份存到多个节点，实现分布式
> - **副本（Replica）**：分片的备份，防止节点宕机丢数据，同时提高读性能

---

## 1. ES 安装

### 1.1 版本选择

推荐使用 **7.x 系列**（如 7.17.x）或 **8.x 系列**。
- 7.x：生态成熟、教程多、兼容性好，**新手指路推荐**
- 8.x：默认开启安全认证，安装后需要处理密码，上手略麻烦

> 注意：不同大版本的 API 有差异，Go 客户端版本要和 ES 服务端大版本保持一致（本项目用的是 v8 客户端，就搭配 ES 8.x）。

### 1.2 安装方式

#### 方式一：Docker 安装（最推荐，新手友好）

```bash
# 拉取镜像
docker pull elasticsearch:7.17.10

# 创建数据挂载目录（数据持久化，容器删了数据还在）
mkdir -p /data/elasticsearch

# 启动容器
docker run -d \
  --name es \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \   # 单节点模式，学习必备
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" # 限制内存（默认吃满机器内存）
  -v /data/elasticsearch:/usr/share/elasticsearch/data \
  elasticsearch:7.17.10
```

> `discovery.type=single-node` 必须加，否则单节点会因无法组集群而启动失败。

#### 方式二：解压安装（Windows / Linux）

1. 官网下载安装包：https://www.elastic.co/cn/downloads/elasticsearch
2. 解压到任意目录（注意：**路径不要有中文和空格**）
3. 启动：
   - Linux/Mac：`bin/elasticsearch`
   - Windows：双击 `bin/elasticsearch.bat`

### 1.3 验证安装是否成功

启动完成后，浏览器或命令行访问：

```bash
curl http://localhost:9200
```

返回类似：

```json
{
  "name": "node-1",
  "cluster_name": "elasticsearch",
  "version": { "number": "7.17.10" }
}
```

看到版本号，说明安装成功。

### 1.4 目录结构简介（解压版）

```
elasticsearch-7.17.10/
├── bin/        # 可执行文件（启动、插件管理）
├── config/     # 配置文件（elasticsearch.yml、jvm.options）
├── data/       # 数据目录（索引数据存这里）
├── logs/       # 日志目录
└── plugins/    # 插件目录（如中文分词器 ik）
```

---

## 2. ES 连接

ES 对外提供的是 **RESTful API**，默认端口 **9200**。所谓"连接"，其实就是发 HTTP 请求。

### 2.1 用浏览器 / curl 连接

```bash
# 查看集群健康状态
curl http://localhost:9200/_cluster/health

# 查看所有索引
curl http://localhost:9200/_cat/indices?v
```

返回绿色（green）说明集群健康。

### 2.2 用 Go 客户端连接（本项目方式）

本项目在 `initialize/es.go` 中封装了连接函数 `ConnectES()`：

```go
package initialize

import (
	"os"
	"server/global"

	"github.com/elastic/elastic-transport-go/v8/elastictransport"
	"github.com/elastic/go-elasticsearch/v8"
	"go.uber.org/zap"
)

func ConnectES() *elasticsearch.TypedClient {
	esCfg := global.Config.ES
	cfg := elasticsearch.Config{
		Addresses: []string{esCfg.URL},      // 例如 http://localhost:9200
		Username:  esCfg.Username,           // 用户名
		Password:  esCfg.Password,           // 密码
	}
	if esCfg.IsConsolePrint {               // 可选：把请求/响应打印到控制台，方便调试
		cfg.Logger = &elastictransport.ColorLogger{
			Output:             os.Stdout,
			EnableRequestBody:  true,
			EnableResponseBody: true,
		}
	}
	client, err := elasticsearch.NewTypedClient(cfg)
	if err != nil {
		global.Log.Error("Failed to create Elasticsearch client", zap.Error(err))
		os.Exit(1)
	}
	return client
}
```

要点：
- 连接 ES 不需要"握手"过程，创建客户端后就能直接用
- 连接信息从配置文件读取（本项目在 `config.yaml` 的 `es` 段），密码等敏感信息建议用环境变量注入
- `ColorLogger` 打印请求体/响应体，是排查问题的大杀器

---

## 3. ES 设置密码

ES 默认没有认证，任何人知道地址就能读写数据。生产环境必须开启安全认证。

### 3.1 修改配置文件

编辑 `config/elasticsearch.yml`，开启安全认证：

```yaml
# 开启安全认证（内置用户体系）
xpack.security.enabled: true
```

> 8.x 版本默认已开启；7.x 需要手动开启。

### 3.2 生成内置用户密码

重启 ES 后，执行：

```bash
# 交互式设置所有内置用户的密码
bin/elasticsearch-setup-passwords interactive

# 或自动生成随机密码（打印在控制台，记得保存）
bin/elasticsearch-setup-passwords auto
```

内置用户：
- `elastic`：超级管理员（最常用）
- `kibana_system`：Kibana 使用
- `logstash_system`：Logstash 使用

### 3.3 用密码连接

带密码访问：

```bash
curl -u elastic:你的密码 http://localhost:9200
```

Go 客户端中在 Config 里填上用户名密码即可（见第 2 章代码），配置文件示例：

```yaml
es:
  url: http://localhost:9200
  username: elastic
  password: 你的密码
  is_console_print: true
  index_prefix: go-blog
```

> 安全提醒：密码不要硬编码在代码或配置文件里，应通过环境变量注入（本项目安全规范）。

---

## 4. Mapping 与创建索引

### 4.1 什么是 Mapping

**Mapping（映射）就是 ES 的"表结构定义"**，它规定：
- 每个字段叫什么名字
- 每个字段是什么类型（text、keyword、integer、date……）
- 文本字段要不要分词、用什么分词器

### 4.2 常用字段类型

| 类型 | 说明 | 典型用途 |
|------|------|----------|
| `text` | 文本，**会分词**，用于全文搜索 | 文章标题、正文 |
| `keyword` | 关键字，**不分词**，整体匹配、精确查找 | 分类、标签、状态 |
| `integer` / `long` | 整数 | 浏览量、评论数 |
| `float` / `double` | 小数 | 价格、评分 |
| `boolean` | 布尔值 | 是否置顶 |
| `date` | 日期，可自定义格式 | 创建时间 |
| `object` | JSON 对象 | 嵌套信息 |
| `nested` | 对象数组（每个对象独立检索） | 评论列表 |

> **text vs keyword 是新手最容易搞混的：**
> - `text`：存储时会被分词器拆成一个个词，搜索时用"词"去匹配 → **模糊匹配、全文搜索用这个**
> - `keyword`：存储时整体保留原值，搜索时必须整体精确相等 → **精确匹配、范围、排序、聚合用这个**

### 4.3 REST 方式创建索引 + Mapping

```json
PUT http://localhost:9200/my_blog

{
  "mappings": {
    "properties": {
      "title":    { "type": "text" },
      "category": { "type": "keyword" },
      "views":    { "type": "integer" },
      "created_at": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" }
    }
  }
}
```

### 4.4 Go 方式创建索引（本项目方式）

本项目在 `model/elasticsearch/article.go` 定义了文章模型和 Mapping：

```go
// 文章模型（对应一条文档）
type Article struct {
	CreatedAt string   `json:"created_at"`
	UpdatedAt string   `json:"updated_at"`
	Cover     string   `json:"cover"`
	Title     string   `json:"title"`
	Keyword   string   `json:"keyword"`
	Category  string   `json:"category"`
	Tags      []string `json:"tags"`
	Abstract  string   `json:"abstract"`
	Content   string   `json:"content"`
	Views     int      `json:"views"`
	Comments  int      `json:"comments"`
	Likes     int      `json:"likes"`
}

// 索引名：所有索引统一加项目名前缀，避免不同项目互相冲突
func ArticleIndex() string {
	return global.Config.ES.IndexPrefix + "_article_index"
}

// Mapping 定义
func ArticleMapping() *types.TypeMapping {
	return &types.TypeMapping{
		Properties: map[string]types.Property{
			"created_at": types.DateProperty{Format: func(s string) *string { return &s }("yyyy-MM-dd HH:mm:ss")},
			"updated_at": types.DateProperty{Format: func(s string) *string { return &s }("yyyy-MM-dd HH:mm:ss")},
			"cover":      types.TextProperty{},
			"title":      types.TextProperty{},
			"keyword":    types.KeywordProperty{},
			"category":   types.KeywordProperty{},
			"tags":       []types.KeywordProperty{},
			"abstract":   types.TextProperty{},
			"content":    types.TextProperty{},
			"views":      types.IntegerNumberProperty{},
			"comments":   types.IntegerNumberProperty{},
			"likes":      types.IntegerNumberProperty{},
		},
	}
}
```

创建索引（本项目在 `service/es_index.go`）：

```go
// IndexCreate 创建索引
func (esService *EsService) IndexCreate(indexName string, mapping *types.TypeMapping) error {
	_, err := global.EsClient.Indices.Create(indexName).Mappings(mapping).Do(context.TODO())
	return err
}
```

调用：

```go
err := service.ServiceGroupApp.EsService.IndexCreate(
	elasticsearch.ArticleIndex(),  // "go-blog_article_index"
	elasticsearch.ArticleMapping(),
)
```

> **创建索引的注意点：**
> - Mapping 创建后**不可修改字段类型**（只能新增字段），所以一开始就要设计好
> - 不指定 Mapping 也可以（ES 自动推断类型），但生产上必须显式定义，防止类型推断错误

---

## 5. ES 索引删除

删除索引会**连索引带数据全部删掉，不可恢复**，务必小心。

### 5.1 REST 方式

```bash
# 删除单个索引
curl -X DELETE http://localhost:9200/my_blog

# 删除多个索引（逗号分隔）
curl -X DELETE http://localhost:9200/index_a,index_b

# 通配符删除（慎用）
curl -X DELETE http://localhost:9200/index_*

# 返回确认
# { "acknowledged": true }
```

### 5.2 Go 方式（本项目方式）

```go
// IndexDelete 删除索引
func (esService *EsService) IndexDelete(indexName string) error {
	_, err := global.EsClient.Indices.Delete(indexName).Do(context.TODO())
	return err
}
```

### 5.3 检查索引是否存在

删除前先检查，避免误删或重复操作：

```go
// IndexExists 检查索引是否存在
func (esService *EsService) IndexExists(indexName string) (bool, error) {
	return global.EsClient.Indices.Exists(indexName).Do(context.TODO())
}
```

本项目 `flag/flag_es.go` 的用法：先检查索引是否存在 → 存在则询问是否删了重建 → 确认后删除再创建。这种"先查再删再建"的模式很适合初始化场景。

---

## 6. ES 文档添加

文档（Document）就是一条 JSON 数据。添加文档有"指定 id"和"自动生成 id"两种方式。

### 6.1 指定 id 添加（可覆盖）

```bash
curl -X PUT http://localhost:9200/my_blog/_doc/1 \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Go 入门指南",
    "category": "编程",
    "views": 100
  }'
```

- PUT + 指定 id：**如果 id 已存在，会整体覆盖**
- 返回结果里有 `"_id": "1"` 和版本号 `"_version"`

### 6.2 自动生成 id 添加

```bash
curl -X POST http://localhost:9200/my_blog/_doc \
  -H "Content-Type: application/json" \
  -d '{
    "title": "ES 学习笔记",
    "category": "技术",
    "views": 20
  }'
```

POST 不指定 id，ES 会自动生成（如 `_id: "abc123..."`）。

### 6.3 Go 方式添加

```go
article := elasticsearch.Article{
	Title:    "Go 入门指南",
	Category: "编程",
	Views:    100,
}

res, err := global.EsClient.Index(elasticsearch.ArticleIndex()).
	Id("1").                  // 指定文档 id
	Document(article).        // 传入要存储的结构体
	Do(context.TODO())
if err != nil {
	global.Log.Error("添加文档失败", zap.Error(err))
}
```

> **新增内容后必须把数据也同步到 ES 吗？**
> 是的。ES 和 MySQL 是两套独立存储，本项目采用"双写"策略：写 MySQL 的同时调用 ES API 把文档写入索引，前端搜索走 ES。

---

## 7. 根据 id 删除

删除单条文档，`_doc` 后跟文档 id。

### 7.1 REST 方式

```bash
curl -X DELETE http://localhost:9200/my_blog/_doc/1
```

返回：

```json
{
  "_index": "my_blog",
  "_id": "1",
  "result": "deleted"
}
```

如果文档不存在，`result` 字段是 `"not_found"`（不会报错）。

### 7.2 Go 方式

```go
res, err := global.EsClient.Delete(elasticsearch.ArticleIndex()).
	Id("1").
	Do(context.TODO())
if err != nil {
	global.Log.Error("删除文档失败", zap.Error(err))
}
```

### 7.3 批量场景的删除思路

真实项目中删除文章时，往往需要"按条件删除"（比如把某个用户的所有文章删掉）。此时可以先查出符合条件的文档 id 列表，再逐个或批量删除（批量删除见第 8 章）。

---

## 8. 文档批量删除

用 `_bulk` 接口一次性提交多条删除操作，效率远高于循环单删。

### 8.1 REST 方式

```bash
curl -X POST http://localhost:9200/_bulk \
  -H "Content-Type: application/json" \
  -d '
{ "delete": { "_index": "my_blog", "_id": "1" } }
{ "delete": { "_index": "my_blog", "_id": "2" } }
{ "delete": { "_index": "my_blog", "_id": "3" } }
'
```

要点：
- 请求体是**多行 NDJSON 格式**（每两行一组操作）
- 第一行写操作类型和位置，第二行写数据（delete 操作**没有**第二行，因为删数据不需要内容）
- 每行末尾必须有换行符

### 8.2 Go 方式（批量删除）

```go
bulkReq := global.EsClient.Bulk().Index(elasticsearch.ArticleIndex())

// 追加多条删除操作
bulkReq.Delete("1")
bulkReq.Delete("2")
bulkReq.Delete("3")

res, err := bulkReq.Do(context.TODO())
if err != nil {
	global.Log.Error("批量删除失败", zap.Error(err))
}
```

### 8.3 按查询删除（Delete By Query）

如果不关心具体 id，想"把符合某条件的所有文档删掉"，用 `delete_by_query`：

```bash
curl -X POST http://localhost:9200/my_blog/_delete_by_query \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "term": { "category": "垃圾广告" }
    }
  }'
```

Go 方式：

```go
_, err := global.EsClient.DeleteByQuery(elasticsearch.ArticleIndex()).
	Query(&types.Query{
		Term: map[string]types.TermQuery{
			"category": { Value: "垃圾广告" },
		},
	}).
	Do(context.TODO())
```

> **bulk 批量接口是通用的**，不仅能删，还能批量增（第 9 章）、批量改（第 13 章），语法结构完全一致，只是第一行的操作类型不同（`index` / `create` / `update` / `delete`）。

---

## 9. ES 批量添加

一次提交多条文档，大幅提升写入性能（插入 1 万条文档时，bulk 比单条循环快几个数量级）。

### 9.1 REST 方式

```bash
curl -X POST http://localhost:9200/_bulk \
  -H "Content-Type: application/json" \
  -d '
{ "index": { "_index": "my_blog", "_id": "1" } }
{ "title": "文章一", "category": "技术", "views": 10 }
{ "index": { "_index": "my_blog", "_id": "2" } }
{ "title": "文章二", "category": "生活", "views": 20 }
'
```

- 操作类型 `index`：如果 id 已存在会**覆盖**
- 操作类型 `create`：如果 id 已存在会**报错**（用来保证不覆盖）

### 9.2 Go 方式（批量添加）

```go
bulkReq := global.EsClient.Bulk()

// 每调一次 Index(...) 追加一条 index 操作
bulkReq.Index(elasticsearch.ArticleIndex()).Id("1").Document(article1)
bulkReq.Index(elasticsearch.ArticleIndex()).Id("2").Document(article2)
bulkReq.Index(elasticsearch.ArticleIndex()).Id("3").Document(article3)

res, err := bulkReq.Do(context.TODO())
if err != nil {
	global.Log.Error("批量添加失败", zap.Error(err))
}
```

配合循环批量插入大量数据：

```go
bulkReq := global.EsClient.Bulk()
for i, article := range articles {
	bulkReq.Index(elasticsearch.ArticleIndex()).
		Id(strconv.Itoa(i+1)).
		Document(article)
}
_, err := bulkReq.Do(context.TODO())
```

> 本项目 `flag/flag_es_import.go` 就是典型的批量导入场景：从 MySQL 查出全部文章，循环构造 bulk 请求一次性导入 ES。

---

## 10. ES 列表查询

ES 的查询统一用 `GET /索引/_search` + JSON 查询体，核心是 `query` 字段。

### 10.1 查询全部（match_all）

```bash
curl -X GET http://localhost:9200/my_blog/_search \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match_all": {} },
    "from": 0,
    "size": 10
  }'
```

- `from`：跳过多少条（页码偏移量）
- `size`：取多少条（每页条数）

**分页公式**：第 N 页 → `from = (N-1) * size`

### 10.2 Go 方式查询列表

```go
res, err := global.EsClient.Search().
	Index(elasticsearch.ArticleIndex()).
	Query(&types.Query{
		MatchAll: &types.MatchAllQuery{},
	}).
	From(0).      // 页码偏移
	Size(10).     // 每页条数
	Do(context.TODO())
if err != nil {
	global.Log.Error("查询失败", zap.Error(err))
}

// 解析结果
var list []elasticsearch.Article
for _, hit := range res.Hits.Hits {
	var article elasticsearch.Article
	if err := json.Unmarshal(hit.Source_, &article); err != nil {
		continue
	}
	list = append(list, article)
}

// 总数
total := res.Hits.Total.Value
```

要点：
- `res.Hits.Hits` 是文档列表，每条 `hit` 里 `Source_` 是原始 JSON（json.RawMessage），要 `json.Unmarshal` 成结构体
- `res.Hits.Total.Value` 是命中的总条数，用于前端分页

### 10.3 按条件过滤 + 排序

```go
res, err := global.EsClient.Search().
	Index(elasticsearch.ArticleIndex()).
	Query(&types.Query{
		Bool: &types.BoolQuery{
			Filter: []types.Query{
				{ Term: map[string]types.TermQuery{
					"category": { Value: "技术" },   // 精确过滤：分类
				}},
			},
		},
	}).
	Sort("created_at", "desc").   // 按创建时间倒序（最新的在前）
	From(0).
	Size(10).
	Do(context.TODO())
```

> 排序建议只用 `keyword` / `integer` / `date` 类型字段。`text` 字段默认不能直接排序（会报错），因为它是分词的。

---

## 11. 模糊匹配与精确匹配

这是 ES 学习的核心难点，核心在于理解 **text（分词）与 keyword（不分词）** 的区别。

### 11.1 精确匹配：term 查询

`term` 查询**不做分词**，拿整个值去匹配字段值，适用于 `keyword` 类型。

```json
{
  "query": {
    "term": { "category": "技术" }
  }
}
```

Go 方式：

```go
Query: &types.Query{
	Term: map[string]types.TermQuery{
		"category": { Value: "技术" },
	},
},
```

> `term` 用在 `text` 字段上会"查不到"，因为 text 被分词存储了，存的是一个个词条，不是完整句子。

### 11.2 模糊匹配：match 查询

`match` 查询**会对搜索词分词**，再和字段的分词结果匹配，适用于 `text` 字段。

```json
{
  "query": {
    "match": { "title": "Go 入门" }
  }
}
```

Go 方式：

```go
Query: &types.Query{
	Match: map[string]types.MatchQuery{
		"title": { Query: "Go 入门" },
	},
},
```

`"Go 入门"` 会被分词成 `["go", "入门"]`，只要标题里包含其中任意一个词就能匹配，而且按相关度打分排序。

### 11.3 模糊程度控制：fuzziness

`match` 允许拼写错误/近似匹配，通过 `fuzziness` 控制：

```go
Match: map[string]types.MatchQuery{
	"title": {
		Query:     "Golng",      // 拼错了，写成 Golng
		Fuzziness: "AUTO",       // 自动允许少量字符差异
	},
},
```

这样即使搜索词有拼写错误，也能匹配到 "Golang"。

### 11.4 通配符匹配：wildcard

`wildcard` 支持 `*`（任意多个字符）和 `?`（单个字符）：

```go
Query: &types.Query{
	Wildcard: map[string]types.WildcardQuery{
		"keyword": { Value: "Go*" },   // 匹配 Go 开头的关键字
	},
},
```

### 11.5 对比总结

| 查询 | 是否分词 | 适用字段 | 典型场景 |
|------|---------|----------|----------|
| `term` | 否，整体匹配 | keyword | 分类筛选、标签、状态精确匹配 |
| `match` | 是，分词匹配 | text | 标题/正文全文搜索 |
| `fuzzy` / fuzziness | 是，容忍拼写错误 | text | 容错搜索 |
| `wildcard` | 否，通配符 | keyword | 前缀/模糊规则匹配 |
| `match_phrase` | 是，要求词序连续 | text | 精确短语搜索（如"中国 人民"） |

> **记法**：keyword 字段 = 整体比较（精确）；text 字段 = 拆词比较（模糊）。选错类型是搜索"查不到/查得乱"的头号原因。

---

## 12. 嵌套字段的搜索

### 12.1 什么是嵌套字段

当文档里有一个**对象数组**字段时（如文章的评论列表），这个字段就是嵌套字段。

```json
{
  "title": "Go 入门",
  "comments": [
    { "user": "小明", "content": "写得好" },
    { "user": "小红", "content": "学到了" }
  ]
}
```

### 12.2 object 与 nested 的区别（关键）

- **object（默认）**：数组里的每个对象会被**扁平化**打散，导致跨对象错配
- **nested**：每个对象**独立存储**，能正确做"对象内条件同时满足"的搜索

> 错误示例（object 类型）：想找"小明 写的『写得好』"，如果小红也写过『写得好』，object 类型会把小明和『写得好』错误地配在一起，返回本不该命中的文档。

### 12.3 定义 nested 类型的 Mapping

```json
{
  "mappings": {
    "properties": {
      "comments": {
        "type": "nested",
        "properties": {
          "user":    { "type": "keyword" },
          "content": { "type": "text" }
        }
      }
    }
  }
}
```

### 12.4 嵌套查询（nested query）

```json
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "term":  { "comments.user": "小明" } },
            { "match": { "comments.content": "写得好" } }
          ]
        }
      }
    }
  }
}
```

Go 方式：

```go
Query: &types.Query{
	Nested: &types.NestedQuery{
		Path: "comments",      // 嵌套字段名
		Query: &types.Query{
			Bool: &types.BoolQuery{
				Must: []types.Query{
					{ Term: map[string]types.TermQuery{
						"comments.user": { Value: "小明" },
					}},
					{ Match: map[string]types.MatchQuery{
						"comments.content": { Query: "写得好" },
					}},
				},
			},
		},
	},
},
```

注意：nested 查询内部字段要写成 **`comments.user`、`comments.content`** 这种带路径的完整名称。

---

## 13. ES 文档更新

更新支持**整体替换**和**局部更新**两种。

### 13.1 整体替换（用 PUT 覆盖）

```bash
curl -X PUT http://localhost:9200/my_blog/_doc/1 \
  -H "Content-Type: application/json" \
  -d '{
    "title": "新的标题",
    "category": "技术",
    "views": 200
  }'
```

> 整体替换 = 先删旧文档再写新文档，**未指定的字段会丢失**。

### 13.2 局部更新（update，只改指定字段）

```bash
curl -X POST http://localhost:9200/my_blog/_update/1 \
  -H "Content-Type: application/json" \
  -d '{
    "doc": { "views": 201 }
  }'
```

只更新 `views` 字段，其他字段原样保留。

### 13.3 用脚本更新（自增等计算场景）

```bash
curl -X POST http://localhost:9200/my_blog/_update/1 \
  -H "Content-Type: application/json" \
  -d '{
    "script": {
      "source": "ctx._source.views += 1"
    }
  }'
```

浏览量 +1 这种场景用脚本最合适（先查再改会有并发问题）。

### 13.4 Go 方式更新

局部更新：

```go
_, err := global.EsClient.Update(elasticsearch.ArticleIndex(), "1").
	Doc(map[string]interface{}{
		"views": 201,
	}).
	Do(context.TODO())
```

脚本自增：

```go
_, err := global.EsClient.Update(elasticsearch.ArticleIndex(), "1").
	Script(&types.Script{
		Source: "ctx._source.views += 1",
	}).
	Do(context.TODO())
```

批量更新（bulk + update）：

```go
bulkReq := global.EsClient.Bulk().Index(elasticsearch.ArticleIndex())
bulkReq.Update("1").Doc(map[string]interface{}{"views": 301})
bulkReq.Update("2").Doc(map[string]interface{}{"views": 302})
res, err := bulkReq.Do(context.TODO())
```

### 13.5 Upsert（存在则更新，不存在则插入）

```go
_, err := global.EsClient.Update(elasticsearch.ArticleIndex(), "1").
	Doc(map[string]interface{}{"views": 10}).
	Upsert(map[string]interface{}{
		"title":    "新文章",
		"category": "技术",
		"views":    10,
	}).
	Do(context.TODO())
```

> **项目实践建议**：本项目中文章更新时，需要同时更新 MySQL 和 ES。可以先更新 MySQL 拿到最新数据，再整体覆盖写入 ES（用 `Index` + 指定 id），保证两边数据一致。

---

## 附录 A：13 章速查表

| 章节 | 操作 | REST 接口 |
|------|------|-----------|
| 1 | 安装 | 启动服务，访问 `GET /` |
| 2 | 连接 | `GET /_cluster/health` |
| 3 | 设置密码 | `elasticsearch-setup-passwords` 命令 |
| 4 | 创建索引 + Mapping | `PUT /索引` |
| 5 | 删除索引 | `DELETE /索引` |
| 6 | 添加文档 | `PUT/POST /索引/_doc[/id]` |
| 7 | 按 id 删除 | `DELETE /索引/_doc/id` |
| 8 | 批量删除 | `POST /_bulk`（delete 操作） |
| 9 | 批量添加 | `POST /_bulk`（index 操作） |
| 10 | 列表查询 | `GET /索引/_search`（match_all） |
| 11 | 模糊/精确匹配 | `term` / `match` / `wildcard` |
| 12 | 嵌套字段搜索 | `nested` query |
| 13 | 文档更新 | `PUT` 或 `POST /索引/_update/id` |

## 附录 B：本项目的 ES 相关文件索引

| 文件 | 作用 |
|------|------|
| `initialize/es.go` | 创建 ES 客户端连接 |
| `config/conf_es.go` | ES 配置结构体（URL、账号、前缀等） |
| `model/elasticsearch/article.go` | 文章模型 + 索引名 + Mapping 定义 |
| `service/es_index.go` | 索引的创建/删除/存在性检查 |
| `flag/flag_es.go` | 命令行创建索引入口 |
| `flag/flag_es_import.go` | MySQL 数据导入 ES |
| `flag/flag_es_export.go` | ES 数据导出 |

> 常用命令备忘：
> ```bash
> # 创建/重建文章索引（本项目）
> go run main.go -es
> ```
