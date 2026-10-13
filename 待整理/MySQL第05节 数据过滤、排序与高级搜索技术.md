---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 5 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：掌握如何从海量数据库中精准提炼所需数据。熟练运用 WHERE 条件过滤、模糊匹配与正则表达式构建搜索核心逻辑，并精通通过去重、排序及分页技术对查询结果集进行最终展示控制。

## 一、Where 核心条件过滤：精准定位数据

在没有任何条件限制时，`SELECT` 会引发**全表扫描**。为了提高检索效率，我们需要使用 `WHERE` 子句配合逻辑运算符和比较运算符来缩小查询范围。

### 1. 比较运算符 (>, <, >=, <=, =)

```sql
-- 查询 id 大于 3 的记录
SELECT * FROM user WHERE id>3;

-- 查询 id 大于等于 3 的记录
SELECT * FROM user WHERE id>=3;

-- 查询 id 小于 3 的记录
SELECT * FROM user WHERE id<3;

-- 查询 id 小于等于 3 的记录
SELECT * FROM user WHERE id<=3;
```

### 2. 逻辑运算符 (AND / OR)

```sql
-- 同时满足：id 大于 3 并且 id 小于 7
SELECT * FROM user WHERE id>3 AND id<7;

-- 满足其一即可：id 等于 3 或者等于 5 或者等于 6 或者等于 10
SELECT * FROM user WHERE id=3 OR id=5 OR id=6 OR id=10;
```

**不等号 `<>` 和 `!=`**：两者**完全等价**，都表示"不等于"。`<>` 是 SQL 标准写法。

> [!warning] AND 和 OR 混用要加括号
> **`AND` 优先级高于 `OR`**，不加括号极易出错：
> ```sql
> -- ❌ 本意是 (id=1 或 id=2) 且 age=18，实际变成了 id=1 或 (id=2 且 age=18)
> WHERE id=1 OR id=2 AND age=18
>
> -- ✅ 正确
> WHERE (id=1 OR id=2) AND age=18
> ```

## 二、范围查询与集合判断（优化 OR 的冗长）

当面对**连续区间判断**或**大量离散点判断**时，使用 `BETWEEN AND` 和 `IN` 会让 SQL 语句更具可读性和执行效率。

### 1. BETWEEN AND（连续区间）

相当于 `>= AND <=` 的简写形式。

```sql
-- 查询 id 在 3 到 7 之间的记录 (包含 3 和 7)
SELECT * FROM user WHERE id BETWEEN 3 AND 7;
```

**BETWEEN 是**闭区间****：`BETWEEN 3 AND 7` 等价于 `id >= 3 AND id <= 7` —— **两头都包含**。

⚠️ 用在日期上要注意：`BETWEEN '2026-10-01' AND '2026-10-31'` **不包含** `2026-10-31 00:00:01` 之后的数据。
建议用 `>= '2026-10-01' AND < '2026-11-01'`。

### 2. IN（离散集合）

用来替代**多个 OR 拼接**的等值查询。

```sql
-- 查询 id 存在于集合 (3,5,6,10) 中的记录
SELECT * FROM user WHERE id IN(3,5,6,10);
```

> [!warning] IN 里元素别太多
> `IN` 列表过长（几千个）会导致 SQL 解析慢、执行计划差。
> 大量值时**改用临时表 JOIN 或分批查询**。

## 三、空值判定：IS NULL 与 IS NOT NULL

在数据库中，**`NULL` 是一个特殊的存在，它代表"未知"或"缺失"，绝不能使用等号 (`=`) 去判断。**

```sql
-- 查询 password 字段值为 null 的行记录
SELECT * FROM user WHERE password IS NULL;

-- 查询 password 字段值不为 null 的行记录
SELECT * FROM user WHERE password IS NOT NULL;
```

> [!important] NULL 的三个反直觉特性（记住）
> | 写法 | 结果 | 原因 |
> | :--- | :--- | :--- |
> | `WHERE col = NULL` | ❌ **永远查不到** | NULL 与任何值比较结果都是 NULL（不是 true） |
> | `WHERE col <> NULL` | ❌ 永远查不到 | 同上 |
> | **`WHERE col IS NULL`** | ✅ 正确 | 唯一正确写法 |
> | `NULL + 1` / `CONCAT('a', NULL)` | **结果都是 NULL** | NULL 具有"传染性" |
> | `COUNT(col)` | **不统计 NULL 行** | 但 `COUNT(*)` 会统计 |
> | `SUM`/`AVG` | **忽略 NULL** | 注意分母会变小 |

## 四、网站搜索技术基石：模糊匹配与正则表达式

早期的网站搜索技术主要基于 `LIKE` 实现，进阶搜索可依赖标签或专业搜索引擎架构。

### 1. LIKE 模糊匹配

使用 **`%`** 作为通配符，代表**任意长度的任意字符**。

```sql
-- 查找用户名中包含 'user1' 的所有记录
SELECT * FROM user WHERE username LIKE '%user1%';
```

| 通配符 | 含义 |
| :--- | :--- |
| **`%`** | **任意长度**的任意字符（包括 0 个） |
| **`_`** | **恰好一个**任意字符 |

```sql
-- 匹配 'user1'、'auser1'、'user1x' ...
WHERE username LIKE '%user1%'
-- 匹配 'user1a'、'user1b'（后面正好一个字符）
WHERE username LIKE 'user1_'
```

### 2. REGEXP 正则表达式（高级搜索）

当模糊匹配无法满足复杂的搜索模式时，可启用 MySQL 内置的**正则表达式引擎**。

```sql
-- 查找以 'php' 结尾的数据 ($ 代表结尾)
SELECT * FROM user WHERE username REGEXP 'php$';

-- 查找以 'php' 结尾或以 'linux' 结尾的数据
SELECT * FROM user WHERE username REGEXP 'php$' OR username REGEXP 'linux$';

-- 查找用户名中包含 'php' 或 'linux' 或 'user' 的数据 (| 代表或)
SELECT * FROM user WHERE username REGEXP 'php|linux|user';
```

### LIKE vs REGEXP 常用元字符对照

| 需求 | **LIKE** | **REGEXP** |
| :--- | :--- | :--- |
| 包含 | `'%abc%'` | `'abc'` |
| 开头 | `'abc%'` | `'^abc'` |
| 结尾 | `'%abc'` | `'abc$'` |
| 或 | ❌（要写多个 OR） | `'abc\|def'` |
| 一个任意字符 | `'_'` | `'.'` |
| 特定字符集 | ❌ | `'[a-z]'`、`'[0-9]'` |
| 重复次数 | ❌ | `'a{2,4}'`、`'a+'`、`'a*'` |

**补充：MySQL 8.0 的 REGEXP_REPLACE / REGEXP_LIKE**：除了 `REGEXP` 运算符，MySQL 8.0 还提供了函数式写法：
```sql
SELECT REGEXP_LIKE(username, '^user[0-9]+$');
SELECT REGEXP_REPLACE('abc123', '[0-9]', '#');
SELECT REGEXP_SUBSTR('abc123def', '[0-9]+');
```

## 五、结果集控制：去重、排序与分页

当数据被成功筛选出后，我们还需要对其**展示形态**进行最终加工。

### 1. DISTINCT 数据去重

用于去除查询结果指定字段中的重复值。

```sql
-- 查询密码，并将相同的密码只显示一次
SELECT DISTINCT password FROM user;
```

> [!warning] DISTINCT 作用在**所有列**的组合上
> ```sql
> SELECT DISTINCT a, b FROM t;
> ```
> 是"**a 和 b 的组合**去重"，不是"只对 a 去重"。

### 2. ORDER BY 结果集排序

排序分为**升序 (`asc`)**（从小到大）和**降序 (`desc`)**（从大到小）。**默认不写时为升序。**

```sql
-- 升序排列：按照 id 从小到大排序 (ASC 可省略)
SELECT * FROM user ORDER BY id;
SELECT * FROM user ORDER BY id ASC;

-- 降序排列：按照 id 从大到小排序 (常用于展示最新数据)
SELECT * FROM user ORDER BY id DESC;
```

**多字段排序**：

```sql
-- 先按 age 降序，age 相同时再按 id 升序
SELECT * FROM user ORDER BY age DESC, id ASC;
```

> [!warning] 排序 NULL 值的位置
> MySQL 中 **NULL 被认为是最小值**：
> - `ORDER BY col ASC` → **NULL 排在最前**
> - `ORDER BY col DESC` → **NULL 排在最后**
>
> 想固定控制：`ORDER BY col IS NULL, col`（把 NULL 放后面）

### 3. LIMIT 限定输出条数（分页基石）

`LIMIT` 专门用于提取指定区间的数据，这是**所有 Web 应用程序"分页功能"的底层核心**。

**语法**：`LIMIT offset, length`（从哪条索引开始，取几条数据，**索引从 0 起始**）

```sql
-- 获取表中的前 5 条数据 (简写与全写等价)
SELECT * FROM user LIMIT 5;
SELECT * FROM user LIMIT 0,5;

-- 核心分页算法演示：从第四条开始提取 3 条数据 (即跳过前3条，取第4-6条)
SELECT * FROM user LIMIT 3,3;
```

#### 分页算法通用公式

```sql
-- 第 page 页，每页 size 条
-- offset = (page - 1) * size
SELECT * FROM user LIMIT (page-1)*size, size;
```

**替代写法：`LIMIT n OFFSET m`**

```sql
SELECT * FROM user LIMIT 3 OFFSET 3;   -- 等价于 LIMIT 3,3
```
这种写法**可读性更好**，推荐用。

> [!warning] 深分页性能陷阱
> `LIMIT 1000000, 10` **不是"只扫 10 行"** —— MySQL 要**先找到并丢弃前 1000000 行**，非常慢。
>
> 优化方案（**记住这个技巧**）：
> ```sql
> -- ❌ 慢：先查 100 万行再丢掉
> SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
>
> -- ✅ 快：用游标/延迟关联
> SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
>
> -- ✅ 或延迟关联
> SELECT u.* FROM user u
>   INNER JOIN (SELECT id FROM user ORDER BY id LIMIT 1000000,10) t ON u.id=t.id;
> ```

---
