# ClickHouse OLAP 完整讲解

## 一、什么是 OLAP

OLAP：联机分析处理（Online Analytical Processing）

> 核心：**海量数据做聚合、统计、多维分析**（求和、平均值、分组、TopN、分位数） 对应的是 OLTP（联机事务处理，MySQL）：**增删改查，单行业务交易**

OLTP：少量数据，频繁修改，查询少量行 OLAP：大量数据，几乎不改，查询大量行做计算

ClickHouse 是**列式存储的 OLAP 数据库**，由俄罗斯 Yandex 开源，主打高性能分析。

## 二、列式存储（核心原理）

### 行式存储 MySQL

```
id | name | age
1  | zhang| 20
2  | li   | 22
```

磁盘存储：`1,zhang,20,2,li,22` 查询：`select avg(age) from t`  要把 id、name、age 全部读出来，只用到 age 列，IO 浪费大。 优点：查完整一行、单行更新快；缺点：大表聚合慢。

### 列式存储 ClickHouse

磁盘分开存每一列： id: `1,2` name: `zhang,li` age: `20,22`

查询 `select avg(age) from t` 只读取 age 这一列，其他列完全不加载** 同列数据类型相同，压缩率极高，磁盘 IO 大幅降低。

> 列存缺点：单行更新 / 删除很差，不适合频繁修改。 ClickHouse 设计理念：**数据写入后基本不变，只追加**

## 三、ClickHouse 核心引擎：MergeTree（最重要）

MergeTree 是 ClickHouse 的**默认核心表引擎**，所有生产环境几乎都用它。 一句话理解： 数据写入后，分成很多小的数据块（part），后台异步合并小 part 成大 part，类似 LSM 树思想，但实现不一样。

特点：

1. **按排序键排序存储**（一般放时间戳），范围查询极快
2. **分区（Partition）**：通常按天分区，查询某一天数据直接跳过其他分区，裁剪数据
3. 支持主键索引、稀疏索引（不是 MySQL 那种稠密索引）
4. 支持 TTL：旧数据自动过期删除（日志 / 监控很常用）

> 衍生引擎：ReplacingMergeTree（去重）、SummingMergeTree（预聚合求和）

## 四、适用场景 & 不适合场景

 适合

1. **日志分析**：Nginx、服务访问日志、错误日志，高吞吐写入，统计 QPS、错误率、慢请求，每秒百万行写入很轻松
2. **时序监控数据**：服务器指标、业务埋点，按时间范围聚合，P95/P99 耗时统计，可以替代 InfluxDB
3. **用户行为分析**：埋点事件，多维分析，UV、PV、用户留存
4. 离线报表、大数据多维分析

 不适合

1. 高频单行 UPDATE / DELETE（MergeTree 不能实时更新，合并才会处理）
2. 高频点查询：根据主键查单条记录（适合 ES/MySQL）
3. 核心交易业务（订单、账户，OLTP 场景）

## 五、ClickHouse 的优点

1. **查询速度极快**：列式存储 + 压缩 + 分区裁剪，百亿数据聚合秒级返回
2. **写入吞吐高**：批量写入性能很强，支持每秒百万行
3. 支持标准 SQL，上手简单，不用学新查询语言
4. 资源占用低，同等数据量比很多 OLAP 引擎省磁盘
5. 内置大量分析函数：分位数、直方图、各类聚合函数

## 六、简单实操示例

### 建表（MergeTree）

```sql
CREATE TABLE access_log (
    time DateTime,
    ip String,
    path String,
    status Int32,
    cost Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(time) -- 按月分区
ORDER BY (time, path); -- 排序键
```

写入一条（生产一般批量写入）

```sql
INSERT INTO access_log VALUES (now(), '127.0.0.1', '/api/test', 200, 0.12);
```

统计接口 QPS，错误率

```sql
SELECT path, count(*) as qps, countIf(status!=200)/count(*) as err_rate
FROM access_log
WHERE time >= now() - INTERVAL 1 HOUR
GROUP BY path;
```

## 七、简单对比

表格

| 数据库        | 存储     | 定位                         |
| ------------- | -------- | ---------------------------- |
| MySQL         | 行存     | OLTP，业务交易，增删改查     |
| ClickHouse    | 列存     | OLAP，海量数据聚合分析       |
| Redis         | 内存 KV  | 缓存，热点数据               |
| Elasticsearch | 倒排索引 | 全文检索，日志检索，少量聚合 |