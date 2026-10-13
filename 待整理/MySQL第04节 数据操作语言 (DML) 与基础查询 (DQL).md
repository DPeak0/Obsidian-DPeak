---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 4 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：理清 SQL 语言的四大阵营，熟练掌握表数据的"增删改" (DML) 核心操作，深刻理解 `DELETE` 与 `TRUNCATE` 在**自增体系下的本质区别**，并开启数据查询语言 (DQL) 的大门，掌握特定字段的提取与别名技术。

## 一、SQL 语言四大分类回顾

在正式操作数据之前，必须在理论层面认清 SQL 语句的四大基本分类，这是数据库工程师沟通的"黑话"：

| 分类 | 全称 | 用途 | 核心命令 |
| :--- | :--- | :--- | :--- |
| **DCL** | Data **Control** Language | 管理数据库的**用户权限与事务控制** | `grant`（授权）、`commit`（提交事务）、`rollback`（回滚事务） |
| **DDL** | Data **Definition** Language | 定义或修改**数据库结构** | `create`、`drop`、`alter` |
| **DML** | Data **Manipulation** Language | 对表中的**核心数据进行操作** | `insert`（增）、`delete`（删）、`update`（改） |
| **DQL** | Data **Query** Language | 专用于从数据库中**提取数据** | `select`（查）—— **只有这一条** |

## 二、环境准备：构建标准的测试表

```sql
-- 若已存在旧版 user 表，先删除再重建，确保表结构为标准版本
DROP TABLE IF EXISTS user;

-- 创建标准的 user 表
CREATE TABLE user(
    id int unsigned auto_increment primary key,
    username varchar(50) not null,
    password varchar(50) not null
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 查看建好的表结构
DESC user;
```

## 三、数据写入：Insert 操作与批量测试数据构造

`INSERT` 语句用于向数据表中追加新的行记录。由于 `id` 字段设置了 `auto_increment`（自增），我们在写入时**无需手动指定 ID，MySQL 会自动分配**。

### 1. 单行插入（标准语法）

显式指定要插入的字段 `(username, password)`，然后提供对应的值。

```sql
-- 插入一条用户 user1，密码为 123
INSERT INTO user(username, password) VALUES ('user1', '123');
```

### 2. 批量插入（性能扩展技巧）

在企业实战中，**频繁执行单行 INSERT 效率极低**。MySQL 支持一条语句批量插入多行，这是**构造测试数据最快的方式**。

```sql
-- 一次性批量插入多条数据
INSERT INTO user(username, password) VALUES
    ('user2', '123'),
    ('user3', '123'),
    ('user4', '456'),
    ('user5', '789');
```

**为什么批量插入快得多**：单条 INSERT = **1 次网络往返 + 1 次事务提交（含刷盘）**。
批量 INSERT = **1 次网络往返 + 1 次事务提交**，装 N 条数据。

**网络延迟和日志刷盘次数被平摊了** —— 这是 DBA 造数/导数的标准手法。

## 四、数据修改：Update 条件更新与安全防范

`UPDATE` 用于修改表中已有的数据记录。

> [!danger] 生产环境铁律
> **在生产环境中执行 `UPDATE` 时，必须且绝对必须带有 `WHERE` 条件。如果没有条件，将会导致整张表的数据被覆盖！**

### 1. 单字段更新

```sql
-- 将 id 为 2 的用户的密码修改为 123
UPDATE user SET password='123' WHERE id=2;
```

### 2. 多字段同时更新

使用逗号分隔多个需要修改的 `字段=值` 对。

```sql
-- 将 id 为 3 的用户名修改为 user33，密码同步修改为 123
UPDATE user SET username='user33', password='123' WHERE id=3;
```

> [!warning] 安全习惯：先 SELECT 再 UPDATE
> ```sql
> -- 1. 先用同样的 WHERE 条件 SELECT 一遍，确认影响的行数
> SELECT * FROM user WHERE id=3;
>
> -- 2. 确认无误后再 UPDATE
> UPDATE user SET username='user33' WHERE id=3;
> ```
> 另外可以开启安全模式，**禁止无 WHERE 的 UPDATE/DELETE**：
> ```bash
> mysql -uroot -p --safe-updates
> # 或临时开关
> SET sql_safe_updates=1;
> ```

## 五、数据删除：Delete 与 Truncate 的本质区别

当我们需要清理无用数据时，通常面临两种选择，它们在**底层机制上有极其关键的差异**。

### 1. DELETE 删除特定行

标准 DML 语句，配合 `WHERE` 逐行清理数据。

```sql
-- 精准删除 id 为 5 的那条用户记录
DELETE FROM user WHERE id=5;
```

### 2. 毁灭级清表：DELETE FROM vs TRUNCATE

当我们想要**彻底清空一张表的所有数据**时，这两种方式表现截然不同：

#### 使用 DELETE 清空表

```sql
DELETE FROM user;
```

**机制**：只清空表里的数据记录，但**不会清除自增计数器**。
如果下一条插入数据，**ID 会接着之前最大的 ID 继续自增**。

#### 使用 TRUNCATE 清空表

```sql
TRUNCATE user;
```

**机制**：既清空了表数据，**同时也会将自增计数器重置为 0**（从 1 重新开始）。
其底层逻辑是**直接把整张表摧毁，然后重新创建一张完全相同的新表**，**执行速度远高于 DELETE**。

### DELETE vs TRUNCATE vs DROP 对比

| 维度 | **DELETE** | **TRUNCATE** | **DROP** |
| :--- | :--- | :--- | :--- |
| **语句类型** | DML | DDL | DDL |
| **能否加 WHERE** | ✅ 能 | ❌ 不能 | ❌ 不能 |
| **删除范围** | 符合条件的行 | **全部行** | **整张表** |
| **自增计数器** | **不重置** | **重置为 1** | 表都没了 |
| **事务/回滚** | ✅ **可回滚** | ❌ **不可回滚**（隐式提交） | ❌ 不可回滚 |
| **触发器** | ✅ 触发 | ❌ 不触发 | ❌ 不触发 |
| **速度** | 慢（逐行删 + 写日志） | **快**（重建表） | 快 |
| **空间回收** | ❌ **不回收**（留空洞） | ✅ 回收 | ✅ 回收 |
| **表结构** | 保留 | **保留** | **删除** |

> [!important] 一句话记忆（记住）
> - **`DELETE`** = 一行行擦掉 → **可回滚，留着空位和自增号**
> - **`TRUNCATE`** = 把表**扔了重建** → **不可回滚，从 1 重新开始**
> - **`DROP`** = 表**连壳都没了**

> [!warning] TRUNCATE 的隐性风险
> 因为 `TRUNCATE` 是 DDL、会**隐式提交**，所以：
> 1. **不能回滚** —— 生产上误执行就没救了
> 2. **不写 Binlog 的 row 事件** —— 基于 Binlog 闪回恢复也救不回来
> 3. **有外键引用时会失败**
>
> 所以生产环境**尽量用 `DELETE` 或有 WHERE 的批量删**，`TRUNCATE` 只在确认要全清时用。

## 六、基础数据查询 (DQL)：Select 与别名技术

`SELECT` 是数据库中使用频率最高、也是语法最复杂的命令。我们先从最基础的检索开始。

### 1. 查询所有字段与记录

```sql
-- 使用 * 通配符代表所有字段
SELECT * FROM user;
```

### 2. 性能优化原则：选择特定的字段

**实战建议：查单列的速度要优于多列。在实际开发中严禁滥用 `SELECT *`**，应当只查询业务需要的指定字段，以**降低磁盘 I/O 和网络传输负担**。

```sql
-- 仅获取用户的 id 和 username，抛弃无用的 password 字段
SELECT id, username FROM user;
```

> [!warning] `SELECT *` 的四宗罪
> 1. **浪费 I/O** —— 读取不需要的字段（尤其 `text`/`blob` 大字段）
> 2. **浪费网络** —— 传输不需要的数据
> 3. **无法使用覆盖索引** —— 索引里没有的字段必须回表
> 4. **表结构变更后行为不可控** —— 加字段后应用可能解析错位

### 3. 给字段起别名 (AS 关键字)

当表字段名过长、或者多表查询字段名冲突时，我们可以通过别名**在返回结果中临时重命名字段**。

```sql
-- 标准写法：使用 AS 关键字，将 username 显示为 user，password 显示为 pass
SELECT username AS user, password AS pass FROM user;

-- 极简写法：AS 关键字可以省略，直接用空格隔开字段名和别名
SELECT username user, password pass FROM user;
```

**别名加引号可以含空格/中文**

```sql
SELECT username AS `用户 名` FROM user;
```

---
