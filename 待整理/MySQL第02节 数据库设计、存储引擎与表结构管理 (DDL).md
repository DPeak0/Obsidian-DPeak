---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 2 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：深入理解 MySQL 8.0 默认存储引擎 InnoDB 的底层文件结构，掌握数据库与数据表的基础 DDL 操作，并熟练使用表复制技术进行快速数据备份与迁移。

## 一、数据表引擎深度剖析：MySQL 8.0 存储结构的巨大变革

> [!important] MySQL 8.0 最大的变化（面试必考，记住）
> 在之前的 MySQL 5.7 版本中，不论是 InnoDB 还是 MyISAM 引擎，**表结构元数据都统一存储在 `.frm` 文件中**。
>
> 但在 **MySQL 8.0 中，由于引入了全新的事务性数据字典，`.frm` 文件被彻底废弃！**

### 1. InnoDB 引擎（MySQL 8.0 默认且最核心的引擎）

在 CentOS 8 系统中，MySQL 的默认数据目录位于 **`/var/lib/mysql/`**。

当创建一张 InnoDB 表时，底层文件结构如下：

| 文件 | 说明 |
| :--- | :--- |
| **表名.ibd** | **表空间文件**，包含该表的**表索引 + 表数据**。在 MySQL 8.0 中，它还**内置了表的序列化字典信息（SDI，替代了旧版的 `.frm`）** |
| **ibdata1** | **系统表空间文件**，所有数据库中的 InnoDB 表**共享此文件**，用于存储 **undo 日志、更改缓冲区**等底层数据 |

### 2. MyISAM 引擎（不支持事务，主要用于极高并发的只读业务）

| 文件 | 说明 |
| :--- | :--- |
| **表名.MYD** | (MY**D**ata) 专门用来存储**表数据** |
| **表名.MYI** | (MY**I**ndex) 专门用来存储**表索引** |
| **表名_xxx.sdi** | MySQL 8.0 新增，以 **JSON 格式**存储该表的**表结构元数据**（替代了旧版的 `.frm`） |

**SDI 是什么**

**SDI = Serialized Dictionary Information（序列化字典信息）**

MySQL 8.0 把表结构元数据**写进了表空间文件本身**（InnoDB 存在 `.ibd` 里，MyISAM 存在 `.sdi` 里）。

好处：**表结构可随表空间文件一起迁移/备份**，不再依赖外部字典文件。

查看 SDI：
```bash
ibd2sdi /var/lib/mysql/mydb/user.ibd
```

### 引擎对比

| 维度 | **InnoDB** | MyISAM |
| :--- | :--- | :--- |
| **事务** | ✅ 支持 | ❌ 不支持 |
| **行锁** | ✅ 支持 | ❌ 只有表锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 强 | ❌ 弱 |
| **全文索引** | ✅ 5.6+ 支持 | ✅ 支持 |
| 存储文件 | `.ibd`（+ `ibdata1`） | `.MYD` + `.MYI` + `.sdi` |
| 适用 | **读写混合、要求可靠** | **极高并发的只读业务** |

**怎么查表的引擎**

```sql
SHOW TABLE STATUS LIKE 'user'\G
SHOW CREATE TABLE user\G
```

## 二、数据库管理 (Database 管理)

我们在终端登录 MySQL (`mysql -uroot -p`) 后，开始执行数据库级的操作。

### 1. 查看数据库

```sql
SHOW DATABASES;
```

### 2. 创建数据库（极力推荐指定字符集）

虽然可以直接 `CREATE DATABASE test;`，但在真实企业开发中，**为了防止中文乱码，推荐显式指定 MySQL 8.0 的默认字符集 `utf8mb4`**。

```sql
-- 创建一个名为 mydb 的数据库，并指定字符集为 utf8mb4
CREATE DATABASE mydb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

### 3. 进入/切换数据库

要对表进行操作，**必须先指定当前操作的数据库**。

```sql
USE mydb;
-- Database changed
```

### 4. 删除数据库

⚠️ **危险操作，会清除该数据库下的所有表和数据。**

```sql
DROP DATABASE test;
```

## 三、数据表管理 (Table 管理)

进入 `mydb` 数据库后，我们开始管理数据表。

### 1. 创建表

创建一个标准的员工表（`user` 表），包含编号、用户名和密码。

```sql
CREATE TABLE user(
    id int,
    username varchar(50),
    password varchar(50)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

可以在建表语句最后使用 `ENGINE=` 显式指定引擎，不过 **MySQL 8.0 默认即为 InnoDB**。

### 2. 查看所有表与查看表结构

```sql
-- 查看当前数据库中的所有表
SHOW TABLES;

-- 查看 user 表的具体字段结构
DESC user;
```

### 3. 修改表名

在业务重构时经常用到修改表名功能。

```sql
-- 将 user 表重命名为 user2
RENAME TABLE user TO user2;

-- 演示完毕后，请将其改回 user，方便后续测试
RENAME TABLE user2 TO user;
```

### 4. 删除表

彻底摧毁一张表及其包含的所有数据。

```sql
-- 测试专用代码，执行后需重新运行上方的 CREATE TABLE 语句恢复表
DROP TABLE user;

-- 更稳妥的写法（表不存在也不报错）
DROP TABLE IF EXISTS user;
```

## 四、表复制技术

在进行危险的数据更新或测试前，DBA 经常需要快速备份一张表。分为"**仅复制结构**"和"**复制结构+数据**"两步。

### 1. 准备测试数据

```sql
INSERT INTO user(id, username, password) VALUES (1, 'user1', '123');
INSERT INTO user(id, username, password) VALUES (2, 'user2', '456');
```

### 2. 复制表结构（克隆空表）

使用 **`LIKE`** 关键字，可以完美克隆原表的**字段结构、索引等属性**，但**不包含数据**。

```sql
-- 创建一张名为 student 的新表，结构与 user 表完全一致
CREATE TABLE student LIKE user;

-- 此时查询 student 表是空的
SELECT * FROM student;
```

### 3. 复制表内容（灌入数据）

使用 **`INSERT INTO ... SELECT`** 语句，将 `user` 表的数据查出并直接灌入 `student` 表。

```sql
-- 将 user 表的所有数据插入到 student 表中
INSERT INTO student SELECT * FROM user;

-- 验证数据是否复制成功
SELECT * FROM student;
```

```text
+------+----------+----------+
| id   | username | password |
+------+----------+----------+
|    1 | user1    | 123      |
|    2 | user2    | 456      |
+------+----------+----------+
2 rows in set (0.00 sec)
```

**一步完成"结构+数据"的复制**：上面的两步可以用一条语句搞定：
```sql
CREATE TABLE student AS SELECT * FROM user;
```
⚠️ **但这种方式会丢失索引、主键、自增等属性**，只复制字段和数据。
所以**要完整备份还是用 `LIKE` + `INSERT ... SELECT` 两步走**。

**三种复制方式的区别**

| 语句 | 结构 | 索引/主键 | 数据 |
| :--- | :---: | :---: | :---: |
| `CREATE TABLE b LIKE a;` | ✅ | ✅ | ❌ |
| `CREATE TABLE b AS SELECT * FROM a;` | ✅ | ❌ | ✅ |
| `LIKE` + `INSERT ... SELECT` | ✅ | ✅ | ✅ |

---
