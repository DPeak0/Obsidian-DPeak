---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 6 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：掌握日常开发中最常用的 MySQL 内置函数，涵盖**字符串清洗、数学计算及日期时间转换**。学会在数据库端直接处理数据格式，从而**降低应用层（如 Java/Python/PHP）的代码压力与数据传输量**。

## 一、字符串处理函数：数据清洗与格式化

在实际业务中，用户输入的数据往往包含多余空格或大小写不一致，借助字符串函数可以**直接在数据库中清洗**。

### 1. CONCAT()：字符串拼接

```sql
-- 基础拼接测试
SELECT CONCAT('php', 'linux');

-- 实战案例：将查询结果自定义格式化输出（模拟生成报表）
SELECT CONCAT('id:', id) AS id, CONCAT('user:', username) AS user, CONCAT('pass:', password) AS pass FROM user;
```

> [!warning] CONCAT 遇到 NULL 会返回 NULL
> ```sql
> SELECT CONCAT('a', NULL);        -- 结果 NULL
> SELECT CONCAT_WS('-', 'a', NULL, 'b');  -- 结果 'a-b'（CONCAT_WS 跳过 NULL）
> ```

### 2. 大小写转换：LCASE() 与 UCASE()

```sql
-- 将大写字符串转换为小写
SELECT LCASE('PHP IS VERY MUCH!');

-- 实战：在查询时直接将用户名转换为大写展示
SELECT id, UCASE(username) FROM user;
```

| 函数 | 作用 | 等价写法 |
| :--- | :--- | :--- |
| `LCASE()` | 转小写 | **`LOWER()`** |
| `UCASE()` | 转大写 | **`UPPER()`** |

### 3. 长度与截取：LENGTH() 与 SUBSTRING()

```sql
-- 获取字符串长度（字节数）
SELECT LENGTH('linux');

-- 从第 6 个字符开始，截取 5 个字符长度
SELECT SUBSTRING('/usr/local/src', 6, 5);
```

> [!important] LENGTH vs CHAR_LENGTH（易错点）
> | 函数 | 返回 | 示例 |
> | :--- | :--- | :--- |
> | **`LENGTH()`** | **字节数** | `LENGTH('中文')` → **6**（utf8mb4 下 3 字节/字） |
> | **`CHAR_LENGTH()`** | **字符数** | `CHAR_LENGTH('中文')` → **2** |
>
> **要数字符个数，用 `CHAR_LENGTH()`，不是 `LENGTH()`。**

**SUBSTRING 的等价写法与负数**

```sql
SELECT SUBSTRING('abcdef', 2, 3);   -- 'bcd'
SELECT SUBSTR('abcdef', 2, 3);      -- 等价
SELECT SUBSTRING('abcdef', -3);     -- 'def'（负数从右往左数）
SELECT LEFT('abcdef', 3);           -- 'abc'
SELECT RIGHT('abcdef', 3);          -- 'def'
```

### 4. 去除多余空格：LTRIM()、RTRIM() 与 SPACE()

```sql
-- 去除左边空格
SELECT LENGTH(LTRIM('   linux'));

-- 去除右边空格
SELECT LENGTH(RTRIM('linux   '));

-- 拼接字段并在中间生成 20 个空格分隔符
SELECT CONCAT('linux', SPACE(20), 'php');
```

| 函数 | 作用 |
| :--- | :--- |
| `LTRIM()` | 去**左边**空格 |
| `RTRIM()` | 去**右边**空格 |
| **`TRIM()`** | **去两边空格**（也最常用） |
| `SPACE(n)` | 生成 **n 个空格** |

### 5. 字符串重复与替换：REPEAT() 与 REPLACE()

```sql
-- 重复 20 个横线
SELECT CONCAT(REPEAT('-', 20), 'linux');

-- 批量替换文本内容测试
SELECT REPLACE('linux and java', 'linux', 'php');
```

**`REPLACE(原字符串, 被替换内容, 新内容)`**

### 字符串函数速查表

| 函数 | 作用 |
| :--- | :--- |
| **`CONCAT(a,b)`** | 拼接（遇 NULL 返回 NULL） |
| **`CONCAT_WS(sep,a,b)`** | 带分隔符拼接（跳过 NULL） |
| `LOWER` / `UPPER` | 转小/大写 |
| **`LENGTH`** vs **`CHAR_LENGTH`** | **字节数** vs **字符数** |
| `SUBSTRING` / `SUBSTR` | 截取 |
| `LEFT` / `RIGHT` | 左/右截取 |
| `TRIM` / `LTRIM` / `RTRIM` | 去空格 |
| `REPEAT` | 重复 |
| **`REPLACE`** | 替换 |
| `INSTR` / `LOCATE` | 查找子串位置 |
| `LPAD` / `RPAD` | 左/右填充 |

## 二、数学与随机函数：精准计算

### 1. 进位与舍弃：CEILING() 与 FLOOR()

```sql
SELECT CEILING(10.5);  -- 结果为 11（向上取整）
SELECT FLOOR(10.5);    -- 结果为 10（向下取整）
```

| 函数 | 作用 |
| :--- | :--- |
| `CEILING()` / `CEIL()` | **向上取整** |
| `FLOOR()` | **向下取整** |
| **`ROUND(x, n)`** | **四舍五入**，保留 n 位小数 |

### 2. 极值运算：MAX() 与 MIN()

```sql
-- 获取 user 表中最大的 id 值
SELECT MAX(id) FROM user;

-- 获取 user 表中最小的 id 值
SELECT MIN(id) FROM user;
```

**MAX/MIN 也是**聚合函数****：它们作用于**整个结果集**，返回一个值。第 7 节会与 `GROUP BY` 配合使用。

### 3. 其他数学运算：BIN() 与 SQRT()

```sql
SELECT BIN(10);     -- 十进制转2进制 → '1010'
SELECT SQRT(100);   -- 开平方 → 10
```

### 4. 巧用 RAND() 提取随机行

`RAND()` 函数用于求随机数。在企业实战中，常配合 `ORDER BY` 和 `LIMIT` 实现"**猜你喜欢**"或"**随机抽奖**"功能。

```sql
-- 全表打乱排序（大表慎用，性能开销大）
SELECT * FROM user ORDER BY RAND();

-- 实战案例：从 user 表中随机抽取 3 名幸运用户
SELECT * FROM user ORDER BY RAND() LIMIT 3;
```

> [!danger] `ORDER BY RAND()` 是大表杀手
> 它会给**每一行生成一个随机数**再做全表排序 —— **复杂度 O(n log n)，且无法用索引**。
>
> 10 万行以上就会明显卡顿，百万行直接拖垮数据库。
>
> **大表随机取样的替代方案**：
> ```sql
> -- 方案1：按随机 ID 区间取（需要 ID 连续）
> SELECT * FROM user WHERE id >= (SELECT FLOOR(RAND()*(SELECT MAX(id) FROM user))) LIMIT 3;
>
> -- 方案2：随机偏移（ID 稀疏时可能有空洞，可多取几条）
> SELECT * FROM user LIMIT FLOOR(RAND()*100000), 3;
> ```

## 三、日期与时间函数：时间线穿梭

时间处理是 Web 开发的高频需求，掌握**时间戳（Timestamp）与标准日期格式（Date）的相互转化**至关重要。

### 1. 获取当前系统时间

```sql
SELECT CURDATE();          -- 当前日期 (YYYY-MM-DD)
SELECT CURTIME();          -- 当前时间 (HH:MM:SS)
SELECT NOW();              -- 当前日期和时间 (YYYY-MM-DD HH:MM:SS)
SELECT UNIX_TIMESTAMP();   -- 获取当前时间戳（距离 1970 年 1 月 1 日的秒数）
```

| 函数 | 返回 |
| :--- | :--- |
| `CURDATE()` / `CURRENT_DATE()` | 日期 |
| `CURTIME()` / `CURRENT_TIME()` | 时间 |
| **`NOW()`** / `CURRENT_TIMESTAMP()` | 日期 + 时间 |
| `UNIX_TIMESTAMP()` | **当前时间戳**（int） |
| `SYSDATE()` | 同 NOW，但返回**函数执行瞬间**的时间 |

### 2. 时间戳转换与日期解析

很多互联网公司的表结构设计喜欢使用 **`int` 类型存储时间戳**。但在报表展示时，需要使用 **`FROM_UNIXTIME()`** 将时间戳转日期。

```sql
-- 将时间戳解析为直观的日期格式
SELECT FROM_UNIXTIME(1492176896);
```

> [!important] 时间戳 ⇄ 日期 互转（记住）
> | 方向 | 函数 | 示例 |
> | :--- | :--- | :--- |
> | **时间戳 → 日期** | **`FROM_UNIXTIME()`** | `FROM_UNIXTIME(1492176896)` |
> | **日期 → 时间戳** | **`UNIX_TIMESTAMP()`** | `UNIX_TIMESTAMP('2026-10-09 22:00:00')` |

**FROM_UNIXTIME 可以格式化输出**

```sql
SELECT FROM_UNIXTIME(1492176896, '%Y年%m月%d日 %H:%i:%s');
```

### 3. 时间特征提取与差值计算

```sql
-- 提取具体日期是当年的第几周
SELECT WEEK('2026-1-8');

-- 提取特定日期的年份
SELECT YEAR('2026-4-14');

-- 计算两个日期之间相差的天数
SELECT DATEDIFF('2026-10-14', '2026-10-10');
```

| 函数 | 作用 |
| :--- | :--- |
| `YEAR()` / `MONTH()` / `DAY()` | 提取年 / 月 / 日 |
| `HOUR()` / `MINUTE()` / `SECOND()` | 提取时 / 分 / 秒 |
| **`WEEK()`** | 一年中的第几周 |
| **`DATEDIFF(d1, d2)`** | **两日期相差天数**（d1 - d2） |
| `TIMEDIFF(t1, t2)` | 两时间相差 |
| **`DATE_ADD(d, INTERVAL n unit)`** | 日期加 |
| **`DATE_SUB(d, INTERVAL n unit)`** | 日期减 |
| `DATE_FORMAT(d, fmt)` | 格式化日期 |

```sql
-- 日期加减示例
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);     -- 7 天后
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);   -- 1 个月前
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');
```

---
