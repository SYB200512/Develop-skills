# MySQL基础与进阶

## 一、MySQL基础

##### 1.数据库基本概念

- 数据库：存放数据的仓库；DBMS：数据库管理系统（MySQL）
- 库 (database) → 表 (table) → 行 (row / 记录)、列 (column / 字段)
- SQL：结构化查询语言，分为 4 大类
  1. DDL 数据定义：`CREATE / ALTER / DROP` 库、表、索引
  2. DML 数据操作：`INSERT / UPDATE / DELETE`
  3. DQL 数据查询：`SELECT`（最常用）
  4. DCL 数据控制：`GRANT / REVOKE` 用户权限

##### 2.基础数据类型

1. `TINYINT`：1 字节，范围 (-128~127)，适合状态、年龄
2. `INT`：4 字节，常用整数
3. `BIGINT`：8 字节，主键 ID 推荐使用
4. `DECIMAL(M,D)`：定点数，金额必须用，不会有浮点数精度丢失
5. `FLOAT/DOUBLE`：浮点数，不适合金额

1. 字符串类型

- `CHAR(n)`：定长字符串，最大 255 字符，适合手机号、身份证
- `VARCHAR(n)`：变长字符串，节省空间，最常用
- `TEXT`：大文本，不能建索引

1. 时间类型

- `DATE`：日期 `yyyy-MM-dd`
- `DATETIME`：日期时间，范围大，不受时区影响，推荐
- `TIMESTAMP`：时间戳，受时区影响

##### 3.表约束（建表时使用）

- `PRIMARY KEY` 主键：唯一且非空，一张表只能一个主键，一般自增 ID
- `AUTO_INCREMENT` 自增，只能用于主键 / 唯一整数
- `NOT NULL` 非空约束，字段不能为 NULL
- `UNIQUE` 唯一约束：值不能重复，可以多个 NULL
- `DEFAULT` 默认值
- `COMMENT` 注释
- `FOREIGN KEY` 外键：保证引用完整性，生产环境一般不使用，影响并发性能

##### 4.命令集合

DDL 库操作（数据库层面）

```sql
-- 查看所有数据库
SHOW DATABASES;

-- 创建数据库
CREATE DATABASE IF NOT EXISTS test_db DEFAULT CHARACTER SET utf8mb4;

-- 查看数据库建库语句
SHOW CREATE DATABASE test_db;

-- 使用数据库（选中库，后续操作都在这个库）
USE test_db;

-- 删除数据库
DROP DATABASE IF EXISTS test_db;

```

 DDL 表操作

```sql
-- 查看当前库所有表
SHOW TABLES;

-- 创建表
CREATE TABLE IF NOT EXISTS `user` (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
  name VARCHAR(50) NOT NULL COMMENT '姓名',
  age INT DEFAULT NULL COMMENT '年龄',
  email VARCHAR(100) UNIQUE COMMENT '邮箱',
  create_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

-- 查看表结构
DESC `user`;
-- 查看建表语句
SHOW CREATE TABLE `user`;

-- 修改表：新增字段
ALTER TABLE `user` ADD COLUMN phone VARCHAR(20) AFTER email;
-- 修改字段类型
ALTER TABLE `user` MODIFY COLUMN age TINYINT;
-- 修改字段名
ALTER TABLE `user` CHANGE COLUMN phone tel VARCHAR(20);
-- 删除字段
ALTER TABLE `user` DROP COLUMN tel;
-- 修改表名
ALTER TABLE `user` RENAME TO `t_user`;

-- 删除表
DROP TABLE IF EXISTS `t_user`;
```

##### 5.DML：增删改（INSERT / UPDATE / DELETE）

###### **INSERT 新增数据**

```sql
-- 单行插入
INSERT INTO `user` (name, age, email) VALUES ('张三', 20, 'zs@xxx.com');

-- 多行插入（推荐，效率更高）
INSERT INTO `user` (name, age, email) 
VALUES 
('李四',22,'ls@xxx.com'),
('王五',25,'ww@xxx.com');

-- 只插入部分字段，其余使用默认值
INSERT INTO `user` (name) VALUES ('赵六');
```

> 注意：字段列表和 values 值数量、类型一一对应。

######  UPDATE 修改数据

```sql
-- 【重要】一定要加WHERE条件！否则全表更新！
UPDATE `user` SET age=21 WHERE id=1;

-- 多个字段同时修改
UPDATE `user` SET age=23, email='new@xxx.com' WHERE name='张三';
```

######  DELETE 删除数据

```sql
-- 根据条件删除，加WHERE！不加会清空整张表
DELETE FROM `user` WHERE id=1;
```

> DELETE vs TRUNCATE
>
> - DELETE：DML，删除行，可以带 where，支持事务回滚，自增 ID 不会重置
> - TRUNCATE：DDL，清空整张表，不能加 where，不支持事务，重置自增 ID，速度更快

#### 6.DQL：SELECT 查询（重中之重）

完整语法顺序：

```sql
SELECT [DISTINCT] 字段列表
FROM 表名
JOIN 关联表 ON 关联条件
WHERE 行过滤条件
GROUP BY 分组字段
HAVING 分组后的过滤
ORDER BY 排序字段 [ASC|DESC]
LIMIT 偏移量, 行数;
```

> 执行顺序（和书写顺序不一样！面试考点）： FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT

```sql
-- 查询全部字段（不推荐生产使用 select *）
SELECT * FROM `user`;

-- 查询指定字段，DISTINCT去重
SELECT DISTINCT name FROM `user`;

-- 别名 as，as可以省略
SELECT name AS username, age FROM `user`;

-- WHERE 条件查询
SELECT * FROM `user` WHERE age = 20;
SELECT * FROM `user` WHERE age > 18 AND age < 30;
SELECT * FROM `user` WHERE name LIKE '张%'; -- %任意字符，_单个字符
SELECT * FROM `user` WHERE age IN (18,20,22);
SELECT * FROM `user` WHERE age IS NULL; -- 判断null不能用 = NULL

-- 聚合函数：COUNT SUM AVG MAX MIN
SELECT COUNT(*) FROM `user`; --统计行数
SELECT AVG(age), MAX(age), MIN(age) FROM `user`;

-- GROUP BY 分组 + HAVING过滤分组结果
SELECT age, COUNT(*) cnt FROM `user` GROUP BY age HAVING cnt>1;

-- ORDER BY 排序 ASC升序（默认） DESC降序
SELECT * FROM `user` ORDER BY age DESC;

-- LIMIT 分页：offset从0开始
SELECT * FROM `user` LIMIT 0,10; --第1页，10条
SELECT * FROM `user` LIMIT 10,10; --第2页，10条
```

#### 多表连接查询

假设有 user (id,name) 和 order (id,user_id,title)

```sql
-- 内连接 INNER JOIN：只返回两边匹配上的数据
SELECT u.name,o.title FROM `user` u INNER JOIN `order` o ON u.id = o.user_id;

-- 左连接 LEFT JOIN：左边表全部保留，右边匹配不到填NULL
SELECT u.name,o.title FROM `user` u LEFT JOIN `order` o ON u.id = o.user_id;

-- 右连接 RIGHT JOIN：右边表全部保留，左边匹配不到填NULL
SELECT u.name,o.title FROM `user` u RIGHT JOIN `order` o ON u.id = o.user_id;
```

#### 7.TCL事务基础命令

```sql
-- 开启事务
START TRANSACTION;

-- 多条DML语句
INSERT INTO user(name) VALUES('小明');
UPDATE user SET age=19 WHERE name='小明';

-- 提交事务，持久化
COMMIT;

-- 回滚，放弃本次事务所有修改
ROLLBACK;
```

事务四大特性 ACID

- A 原子性：要么全部成功，要么全部失败
- C 一致性：事务前后数据满足业务约束
- I 隔离性：多个事务互相隔离
- D 持久性：提交之后永久保存

#### 8.账号权限

```0
-- 创建用户
CREATE USER 'test'@'localhost' IDENTIFIED BY '123456';
-- 授权
GRANT SELECT,INSERT ON test_db.* TO 'test'@'localhost';
-- 刷新权限
FLUSH PRIVILEGES;
-- 撤销权限
REVOKE INSERT ON test_db.* FROM 'test'@'localhost';
-- 删除用户
DROP USER 'test'@'localhost';
```

## 二、MySQL进阶

### 1. 存储引擎

```sql
-- 查看支持引擎
SHOW ENGINES;
```

1. **InnoDB（MySQL5.7/8.0 默认）** ✅ 支持事务、行锁、MVCC、崩溃恢复；聚簇索引结构；并发性能好。生产首选。
2. MyISAM：不支持事务，只有表锁，查询快，崩溃容易丢数据，新项目不用。
3. Memory：内存表，重启数据丢失。

### 2. 索引（B+Tree）

B+Tree 特点：全部数据放在叶子节点；叶子节点双向链表，范围查询优秀；非叶子节点只存索引键。 索引分类：

1. **聚簇索引（主键索引）**：叶子节点存储**完整一行数据**。一张表只能有一个。没有主键自动寻找唯一非空列，无则生成隐藏 rowid。

2. **二级索引（普通索引）**：叶子节点存主键值。查到主键后，去聚簇索引拿完整数据 → **回表**。

3. 联合索引

   ：

   ```sql
   CREATE INDEX idx_name_age ON user(name,age);
   ```

   ✅ 

   最左前缀原则

   ，从左向右匹配。

   > 注意：`where age=25 and name='Alice'` MySQL 优化器会自动调换条件顺序，索引依旧生效！

4. **覆盖索引**：查询需要的所有字段都存在索引内，**不需要回表**，性能最优。

索引失效常见场景：

1. 索引列使用函数 `where UPPER(name)='ABC'`
2. 隐式类型转换（字段字符串，传入数字）
3. `like '%关键词'` 前缀模糊匹配；`关键词%` 可以走索引
4. or 一侧没有索引
5. 联合索引中范围查询（> <）后面的字段，索引失效

验证索引：`EXPLAIN SELECT ...`，重点看 type、key、Extra。

索引设计原则：

- 选择性高（区分度好）的字段建索引；性别这类低区分度字段不适合建索引
- 联合索引：等值条件放前面，范围条件放后面
- 索引不是越多越好，索引会降低增删改性能（需要维护索引树）

## 3.MVCC多版本并发控制

##### 前提知识

| 隔离级别                           | 脏读  | 不可重复读 | 幻读     |
| ---------------------------------- | ----- | ---------- | -------- |
| READ UNCOMMITTED 读未提交          | ✅存在 | ✅存在      | ✅存在    |
| READ COMMITTED 读已提交 (RC)       | ❌无   | ✅存在      | ✅存在    |
| REPEATABLE READ 可重复读 (RR 默认) | ❌无   | ❌无        | 基本解决 |
| SERIALIZABLE 串行化                | ❌无   | ❌无        | ❌无      |

脏读：读到别的事务**未提交**的数据 不可重复读：同一个事务两次读取同一行，结果不同（别的事务修改提交） 幻读：同一个事务范围查询，行数变化（别的事务插入删除）





> InnoDB 实现快照读（普通 SELECT 不加锁），提升并发。每行有两个隐藏字段：
> `trx_id`：最后修改该行的事务 ID
> `roll_pointer`：指向 undo log 旧版本，串联数据版本链

- undo log：保存旧版本数据，用于事务回滚、MVCC 版本链
- redo log：WAL 预写日志，崩溃恢复，保证事务持久性

ReadView：事务读取数据时的一致性视图

- RC（读已提交）：每次 SELECT 生成新 ReadView，会出现不可重复读
- RR（可重复读，MySQL 默认）：事务第一次 SELECT 生成 ReadView，整个事务复用，解决不可重复读；配合间隙锁解决幻读

两种读：

1. **快照读**：普通`select`，不加锁，读取数据快照，MVCC 实现
2. **当前读**：`select ... for update` / `lock in share mode` /update/delete。加锁，读取最新数据。

![image-20260912222922644](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260912222922644.png)

## 4.MySQL日志体系

1. redo log：引擎层，崩溃恢复，循环写，WAL 机制![image-20260912225720868](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260912225720868.png)

2. undo log：引擎层，回滚、MVCC 版本链

   ![image-20260912223759712](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260912223759712.png)

3. binlog：Server 层，记录所有 DDL/DML，主从复制、数据恢复；binlog 三种格式：statement、row、mixed。![image-20260912230111565](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260912230111565.png)

> 两阶段提交：保证 redo log 和 binlog 数据一致性。

![image-20260912231406146](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260912231406146.png)