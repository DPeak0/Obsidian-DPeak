---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 3 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：掌握 MySQL 核心数据类型的选择原理，深刻理解字段约束属性（尤其是 `unsigned` 与 `auto_increment`），了解 MySQL 8.0 字符集的底层优化，并熟练运用 `ALTER TABLE` 语句对表结构进行灵活的动态调整。

## 一、常用数据类型详解

在设计数据库时，选择合适的数据类型，既能保证最大长度或最大数值满足业务需求，又能**最大限度地节省占用空间**。

### 1. 数值型

| 类型 | 有符号范围 | 无符号 (unsigned) 范围 | 典型用途 |
| :--- | :--- | :--- | :--- |
| **tinyint** | -128 ~ +127 | **0 ~ 255** | 状态、年龄等小数值 |
| **int** | -2147483648 ~ 2147483647（约 ±21 亿） | **0 ~ 4294967295**（约 42 亿） | **最常用的主键类型** |
| **bigint** | -9223372036854775808 ~ 9223372036854775807 | 0 ~ 18446744073709551615 | **分布式 ID、超高频流水号** |

### 2. 字符串型

| 类型 | 特点 | 适用 |
| :--- | :--- | :--- |
| **char(M)** | **定长**字符串，最大长度 **255**。因为长度固定，**查询速度极快** | 手机号、**MD5 加密后的密码**等固定长度数据 |
| **varchar(M)** | **变长**字符串，最大长度理论值 **65535**。根据实际内容占用空间，**更为节省硬盘** | **目前使用最广的字符串类型** |
| **text 系列** | 用于存储大段文本文章 | 见下表 |

| text 类型 | 容量 |
| :--- | :--- |
| `tinytext` | 0 ~ 255 字节 |
| `text` | 0 ~ 65535 字节 |
| `mediumtext` | 0 ~ 16777215 字节（约 16MB） |
| `longtext` | 0 ~ 4294967295 字节（约 4GB） |

**char vs varchar 怎么选（记住）**

| 场景 | 选择 |
| :--- | :--- |
| 长度**固定**（手机号、身份证、MD5、UUID） | **`char`**（定长，无额外长度开销，更快） |
| 长度**变化大**（用户名、地址、备注） | **`varchar`**（变长，省空间） |

⚠️ `varchar` 需要 **1-2 字节额外空间**记录实际长度，且更新变长字段可能产生页分裂。

### 3. 日期时间型

在企业开发中，关于时间的存储有**两种流派**：

| 流派 | 说明 |
| :--- | :--- |
| **int 时间戳** | 将时间转化为**秒数**（如 `1492086867`），以 `int` 类型存储。这种方式**跨时区极其方便，查询效率极高** |
| **datetime** | MySQL 原生日期时间类型（如 `'2026-10-09 22:00:00'`），**直观易读**，MySQL 8.0 对其底层也进行了大量优化 |

**补充：其他日期类型**

| 类型 | 范围 | 大小 | 特点 |
| :--- | :--- | :--- | :--- |
| `date` | 1000-01-01 ~ 9999-12-31 | 3 字节 | 只有日期 |
| `time` | -838:59:59 ~ 838:59:59 | 3 字节 | 只有时间 |
| `datetime` | 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59 | **8 字节** | **不受时区影响** |
| `timestamp` | 1970-01-01 ~ **2038-01-19** | **4 字节** | **随时区变化**，有 2038 问题 |
| `int` | 1970 ~ 2038（无符号到 2106） | 4 字节 | 最省空间，**推荐** |

## 二、数据字段属性设置（约束）

字段属性是对字段类型的**进一步约束**，保证数据的合法性。

| 属性 | 说明 | 示例 |
| :--- | :--- | :--- |
| **`unsigned`** | **无符号**。只能存储正数，可以将正数的**取值范围扩大一倍** | `create table t2(id int unsigned);` |
| **`zerofill`** | **0 填充**。如果数值长度不够，左侧自动补 0，主要用于显示格式化 | `int(5) zerofill` → `00001` |
| **`auto_increment`** | **自动增长**。通常配合**主键** (`primary key`) 使用，实现 ID 的自动递增分配 | `create table t9(id int unsigned auto_increment primary key, username varchar(50));` |
| **`null` / `not null`** | `null` 表示该字段允许为空；`not null` 表示强制要求该字段必须有值，**不允许为空** | `username varchar(50) not null` |
| **`default`** | **默认值**。如果在插入数据时不指定该字段的值，则使用默认值 | `age int not null default 18;` |

> [!warning] `auto_increment` 的三个限制
> 1. **必须建在索引上**（通常是主键）
> 2. **一张表只能有一个** `auto_increment` 列
> 3. **类型必须是整数**（`tinyint`/`int`/`bigint`）

**`zerofill` 会自动附带 `unsigned`**

```sql
CREATE TABLE t (a INT(5) ZEROFILL);
SHOW CREATE TABLE t\G
-- 结果为：`a` int(5) unsigned zerofill
```
因为补零填充只对非负数有意义。

## 三、MySQL 8.0 全新默认字符集优势

过去 MySQL 中的 **`utf8` 其实是残缺的（最多支持 3 个字节）**，无法存储复杂的 **Emoji 表情和生僻字**。

在 **MySQL 8.0 中，官方做出了重大改变，将默认字符集全面升级为了真正的 UTF-8：`utf8mb4`**，并将默认的校验规则改为了更先进的 **`utf8mb4_0900_ai_ci`**。

这一改动意味着，你在 CentOS 8 下通过 dnf 安装的 MySQL 8.0，**无需再像旧版那样去繁琐修改 `my.cnf`，开箱即可完美支持所有生僻字与表情符号！**

### 字符集对比

| 字符集 | 最大字节 | 支持 Emoji | 说明 |
| :--- | :---: | :---: | :--- |
| `utf8`（MySQL 里的别名） | **3 字节** | ❌ | **是残缺的 UTF-8**，MySQL 8.0 中也叫 `utf8mb3` |
| **`utf8mb4`** | **4 字节** | ✅ | **真正的 UTF-8**，MySQL 8.0 默认 |

### 排序规则（Collation）解析

**`utf8mb4_0900_ai_ci`** 拆开看：

| 片段 | 含义 |
| :--- | :--- |
| `utf8mb4` | 字符集 |
| **`0900`** | 基于 **Unicode 9.0** 标准（MySQL 8.0 新增） |
| **`ai`** | **accent insensitive** —— **不区分重音**（`é` 和 `e` 视为相同） |
| **`ci`** | **case insensitive** —— **不区分大小写**（`A` 和 `a` 视为相同） |

**常见排序规则速查**

| 排序规则 | 区分大小写 | 区分重音 |
| :--- | :---: | :---: |
| `utf8mb4_0900_ai_ci` | ❌ | ❌ |
| `utf8mb4_0900_as_cs` | ✅ | ✅ |
| `utf8mb4_general_ci` | ❌ | ❌ | 旧版（5.7）默认，**排序不严谨** |
| `utf8mb4_bin` | ✅ | ✅ | 按**二进制**比较，最严格 |

## 四、字段管理高级操作 (ALTER TABLE)

在日常开发中，业务需求经常变更，我们需要使用 `ALTER TABLE` 动态修改表结构。

### 1. 添加字段 (ADD)

**默认添加在表的最末尾**：

```sql
-- 给 user 表添加一个 age 字段，类型为无符号微小整型，不能为空
ALTER TABLE user ADD age tinyint unsigned not null;
```

**通过 `after` 指定添加到某个字段之后**：

```sql
-- 添加一个 pass 字段，紧跟在 username 字段之后
ALTER TABLE user ADD pass varchar(30) not null AFTER username;
```

**通过 `first` 添加到表的最前面**：

```sql
-- 将新字段添加到第一列
ALTER TABLE user ADD pass2 varchar(30) not null FIRST;
```

### 2. 删除字段 (DROP)

```sql
-- 删除刚才创建的 pass2 字段
ALTER TABLE user DROP pass2;

-- 按照原笔记示例，删除 password 字段
ALTER TABLE user DROP password;
```

### 3. 修改字段 (MODIFY 与 CHANGE)

| 关键字 | 能力 |
| :--- | :--- |
| **`MODIFY`** | 只能修改字段的**类型和属性**，**不能改字段名** |
| **`CHANGE`** | 可以**同时修改字段名、类型和属性** |

**语法**：

```sql
-- MODIFY：ALTER TABLE 表名 MODIFY 字段名 类型 约束;
-- CHANGE：ALTER TABLE 表名 CHANGE 旧字段名 新字段名 类型 约束;
```

```sql
-- 将 username 的长度修改为 30，并加上非空约束
ALTER TABLE user MODIFY username varchar(30) not null;

-- 将 username 字段改名为 user，同时设置类型和约束
ALTER TABLE user CHANGE username user varchar(30) not null;
```

> [!warning] MODIFY 会覆盖原有属性
> `MODIFY` / `CHANGE` **不是"增量修改"，而是"重新定义"**。
>
> 如果原来字段有 `DEFAULT 18`，你 `MODIFY age tinyint` 时**不写 default**，那么**默认值就被清掉了**。
>
> **改字段前先用 `SHOW CREATE TABLE user\G` 看清原定义。**

**其他常用 ALTER 操作**

```sql
-- 改表名
ALTER TABLE user RENAME TO user2;

-- 改引擎
ALTER TABLE user ENGINE=InnoDB;

-- 改字符集
ALTER TABLE user CONVERT TO CHARACTER SET utf8mb4;

-- 加索引
ALTER TABLE user ADD INDEX i_age(age);

-- 加主键
ALTER TABLE user ADD PRIMARY KEY(id);
```

---
