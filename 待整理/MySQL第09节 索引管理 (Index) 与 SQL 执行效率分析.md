---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 9 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：透彻理解 MySQL 索引的底层分类与设计原则。熟练掌握不同类型索引的创建与销毁，学会使用 `EXPLAIN`（或 `DESC`）剖析 SQL 执行计划，并具备在复杂业务代码中排查"**索引失效**"等性能致命伤的能力。

## 一、MySQL 索引分类与基础管理

索引（Index）**相当于字典的目录**。在没有索引的情况下，MySQL 必须**从第一行扫描到最后一行（全表扫描）** 来寻找匹配的行；有了索引，MySQL 就能通过特定的数据结构（如 **B+Tree**）迅速定位数据位置。

### 1. 查看表索引

在命令终端中，建议在 SQL 语句末尾使用 **`\G`** 替代分号 `;`，这会将原本**横向显示的表格转换为纵向排版**，非常适合查看复杂的索引和执行计划信息。

```sql
-- 查看 user 表上存在的所有索引信息
SHOW INDEX FROM user\G
```

### 2. 普通索引 (Normal Index)

最基础的索引，没有任何限制，主要用于**加速普通字段的查询**。

```sql
-- 为 user 表的 age 字段添加名为 i_age 的普通索引
CREATE INDEX i_age ON user(age);

-- 删除名为 i_age 的索引
DROP INDEX i_age ON user;
```

### 3. 唯一索引 (Unique Index)

与普通索引类似，但**索引列的值必须唯一，允许有空值**。通常用于如**身份证号、手机号、唯一用户名**等字段。

```sql
-- 为 username 字段添加名为 u_username 的唯一索引（确保用户名不重复）
CREATE UNIQUE INDEX u_username ON user(username);

-- 删除唯一索引与普通索引语法一致
DROP INDEX u_username ON user;
```

### 索引类型速查

| 类型 | 关键字 | 特点 |
| :--- | :--- | :--- |
| **普通索引** | `INDEX` / `KEY` | 无限制，只加速查询 |
| **唯一索引** | `UNIQUE INDEX` | 值必须唯一，**允许多个 NULL** |
| **主键索引** | `PRIMARY KEY` | 唯一 + **非空**，一张表只有一个 |
| **复合索引** | `INDEX(a,b,c)` | 多列组合，遵循最左前缀 |
| **全文索引** | `FULLTEXT INDEX` | 用于大文本的分词搜索 |
| **空间索引** | `SPATIAL INDEX` | 用于地理数据（GIS） |

**补充：什么是"回表"和"覆盖索引"**

- **回表**：通过二级索引找到主键后，**再回主键索引（聚簇索引）取整行数据** —— 多一次查找
- **覆盖索引**：查询的字段**都在索引里**，不用回表 —— **性能最好**

```sql
-- 假设有索引 idx_username(username)
SELECT id, username FROM user WHERE username='x';  -- 覆盖索引（id 在主键索引里）
SELECT * FROM user WHERE username='x';              -- 需要回表
```

## 二、复合索引与"最左前缀"原则

在实际业务中，我们经常需要在多个字段上同时过滤数据（如按班级和年龄综合筛选），此时可以将多个列组合起来创建一个**复合索引**。

**创建复合索引代码示例**：

```sql
CREATE INDEX i_class_age ON user(class_id, age);
```

> [!important] 最左前缀原则核心理论（记住）
> **对于创建的多列索引，只要查询的条件中用到最左边的列，索引一般就会被使用。**

| 场景 | 查询条件 | 是否走索引 | 说明 |
| :--- | :--- | :---: | :--- |
| **生效** | `WHERE class_id = 1` | ✅ | 用到最左列，走索引 |
| **生效** | `WHERE class_id = 1 AND age = 19` | ✅ | 按序完全匹配，走**完整复合索引** |
| **失效** | `WHERE age = 19` | ❌ | **跳过了最左边的 `class_id`，复合索引失效，触发全表扫描** |

**最左前缀的直观理解**：复合索引 `(class_id, age)` 就像电话簿**先按班级排、班级内再按年龄排**。

- 找"1 班的所有人" → 翻到 1 班那一段 ✅
- 找"1 班 19 岁的人" → 翻到 1 班再找 19 岁 ✅
- 找"所有 19 岁的人" → **全簿翻一遍，因为没按年龄整体排** ❌

> [!warning] 一个容易忽略的细节：范围查询会中断
> ```sql
> -- 索引 (a, b, c)
> WHERE a = 1 AND b > 2 AND c = 3
> ```
> `b > 2` 是**范围查询**，`c` 就用不上索引了（只能用到 a 和 b）。
>
> 所以复合索引的字段顺序：**等值字段放前面，范围字段放最后**。

## 三、使用 EXPLAIN / DESC 剖析 SQL 执行计划

要定位执行效率较低的 SQL 语句，我们可以使用 `explain` 或 `desc` 定位一条 SQL 语句的**影响行数及执行细节**。这是**高级 DBA 排查慢查询的核心利器**。

### 1. 基础用法

只需在原本的 `SELECT` 语句前加上 `DESC`（或 `EXPLAIN`）。

```sql
-- 分析按 username 检索时的执行计划
DESC SELECT * FROM user WHERE username='user8'\G
```

### 2. 核心输出字段解析

| 字段 | 含义 |
| :--- | :--- |
| **`type`** | **连接类型 —— 极其重要的指标！** |
| `possible_keys` | MySQL 预测在此次查询中**可能会用到**的索引 |
| **`key`** | MySQL 在查询中**实际决定使用**的索引。**如果为 `NULL`，则说明没走索引** |
| **`rows`** | MySQL 预估为了找到所需记录，**需要读取的行数**。这个数字**越小，查询越快** |

#### `type` 字段的等级（从好到差）

| type 值 | 含义 | 评价 |
| :--- | :--- | :--- |
| `system` | 表只有一行 | 最好 |
| **`const`** | 通过主键/唯一索引**一次命中** | 极好 |
| **`eq_ref`** | JOIN 时用主键/唯一索引关联 | 很好 |
| **`ref`** | 用**非唯一索引**等值查找 | 好 |
| **`range`** | 索引**范围扫描**（`BETWEEN`、`>`、`LIKE 'x%'`） | 较好 |
| **`index`** | **全索引扫描**（只比全表快一点） | 差 |
| **`ALL`** | **全表扫描** | **最差，必须优化** |

> [!important] 记住这个判断口诀
> **看到 `type: ALL` 就说明是全表扫描，是性能最差的信号，必须优化。**
>
> 如果显示 **`index`、`range`、`ref` 或 `const`** 等，则说明**成功利用了索引**。

### 3. EXPLAIN 的其他实用字段

| 字段 | 说明 |
| :--- | :--- |
| **`Extra`** | **额外信息，含关键性能提示** |
| `key_len` | 实际使用的索引长度（可判断用了复合索引的几列） |
| `filtered` | 过滤后剩余行的百分比 |

**`Extra` 常见值**：

| 值 | 含义 |
| :--- | :--- |
| **`Using index`** | ✅ **覆盖索引**，不用回表，**很好** |
| `Using where` | 使用了 WHERE 过滤 |
| **`Using filesort`** | ⚠️ **额外排序，无法用索引排序，需要优化** |
| **`Using temporary`** | ⚠️ **用了临时表（常见于 GROUP BY / DISTINCT），需要优化** |
| `Using join buffer` | JOIN 时无索引，用了连接缓冲 |

**MySQL 8.0 的 EXPLAIN 升级**

```sql
EXPLAIN FORMAT=TREE SELECT ...;      -- 树形执行计划，更直观
EXPLAIN ANALYZE SELECT ...;          -- 实际执行并给出真实耗时
EXPLAIN FOR CONNECTION <id>;         -- 分析正在运行的连接
```

## 四、常见索引失效场景大排查

很多时候，**即便字段上建了索引，由于 SQL 写法不当，也会导致 MySQL 放弃索引转而进行全表扫描**。必须坚决避开以下"**四大坑**"：

### 1. LIKE 模糊匹配时 % 号前置

当使用 `like` 进行搜索时，**`%` 在前索引可能会失效**。因为**索引是通过前缀来进行字典序排序匹配的**。

```sql
-- 索引失效 (type: ALL)：无法知道前面是什么字符，只能全表扫
DESC SELECT * FROM user WHERE username LIKE '%linux'\G

-- 索引生效 (type: range)：以 linux 开头，可以使用 B+Tree 索引检索
DESC SELECT * FROM user WHERE username LIKE 'linux%'\G
```

### 2. 滥用 OR 关键字

在使用 `or` 的情况下**两边的索引都有可能失效**。只有当 **`OR` 连接的每一个字段都有独立索引**时，MySQL 才会尝试进行**索引合并（Index Merge）**，否则直接退化为全表扫描。

```sql
-- 假设 username 有索引，但 age 没有索引，整个语句的索引直接失效
DESC SELECT * FROM user WHERE username='user7' OR age=15\G
```

**优化方案**：尽量使用 **`UNION`** 替代包含不同字段的复合 OR 查询。

```sql
SELECT * FROM user WHERE username='user7'
UNION
SELECT * FROM user WHERE age=15;
```

### 3. 数据类型不匹配引发的"隐式转换"

> [!danger] 这是开发中最容易犯的错（记住）
> **当条件中字段和值的类型不同时，该字段的索引可能会失效。**

```sql
-- 假设 username 定义为 varchar(50) 字符串类型
-- 索引失效 (type: ALL)：由于传入了整型 200，MySQL 会在底层对全表字段执行隐式数据转换函数，导致索引瘫痪
DESC SELECT * FROM user WHERE username=200\G

-- 索引生效 (type: ref)：类型严格匹配，必须加引号
DESC SELECT * FROM user WHERE username='200'\G
```

> [!important] 隐式转换的规则（记住）
> **MySQL 在字符串和数字比较时，会把字符串转成数字。**
>
> - `username = 200` → 每一行的 `username` 都被转成数字再比 → **索引失效**
> - `username = '200'` → 类型一致，**走索引**
>
> ⚠️ **反过来**（数字字段和字符串比）**是走索引的**，因为会转成数字：
> ```sql
> -- id 是 int，这两条都走索引
> WHERE id = '200'
> WHERE id = 200
> ```

### 4. IS NULL 判断会导致索引失效

当判断 null 值时会使用 `username` 这一列的索引，**在 MySQL 8.0 中，`IS NULL` 判断无法使用索引**。

```sql
DESC SELECT * FROM user WHERE username IS NULL\G
```

> [!warning] 关于 NULL 与索引的补充说明
> 严格来说：
> - **MySQL 的 B+Tree 索引是可以存储 NULL 值的**
> - 但**优化器常常认为"NULL 行占比小"，走索引反而要回表，不如直接全表扫**
> - 实际是否走索引取决于**优化器的成本估算**
>
> **规避建议**：字段尽量设为 `NOT NULL DEFAULT ''`（字符串）或 `NOT NULL DEFAULT 0`（数字），**从设计上避免 NULL**。

### 索引失效场景汇总表

| # | 场景 | 失效原因 | 优化方案 |
| :--- | :--- | :--- | :--- |
| 1 | `LIKE '%xxx'` | 无法用前缀排序定位 | 改用 `LIKE 'xxx%'` 或**全文索引/搜索引擎** |
| 2 | `OR` 连接非索引字段 | 无法用索引同时满足 | 改用 **`UNION`** |
| 3 | **隐式类型转换** | 全表做类型转换 | **类型严格匹配（加引号）** |
| 4 | `IS NULL` | 优化器估算不走索引 | 设计上**避免 NULL** |
| 5 | 索引列上做**运算/函数** | `WHERE YEAR(dt)=2026` | 改成 `WHERE dt >= '2026-01-01' AND dt < '2027-01-01'` |
| 6 | 复合索引**不满足最左前缀** | 跳过最左列 | 调整条件或索引顺序 |
| 7 | `!=` / `<>` 不等于 | 结果集通常太大 | 尽量用范围替代 |
| 8 | `NOT IN` / `NOT EXISTS` | 同 `!=` | 改 `LEFT JOIN ... IS NULL` |

> [!important] 第 5 条最容易忽略（记住）
> **索引列上做运算或函数，索引一定失效。**
> ```sql
> -- ❌ 失效
> WHERE YEAR(access_time) = 2026
> WHERE id + 1 = 100
>
> -- ✅ 生效（把运算移到值那边）
> WHERE access_time >= '2026-01-01' AND access_time < '2027-01-01'
> WHERE id = 99
> ```

---
