---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 10 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：从开发者的视角切换至 **DBA 的上帝视角**。掌握服务器整体运行状态的监控方法，学会配置并捕捉**慢查询日志（Slow Query Log）**，理解表空间的检查与回收机制，并最终掌握在 CentOS 8 生产环境下，**越过授权表极限营救 MySQL 8.0 root 密码**的企业级急救预案。

## 一、服务器读写频次状态检查

要诊断数据库的性能瓶颈，首先需要了解**自本次启动以来服务器的宏观压力**。我们可以通过 `SHOW STATUS` 命令检查各类操作的使用频次。

### 1. 基础增删改查频次监控

```sql
-- 检查本次启动以来的更新操作频次
SHOW STATUS LIKE "%Com_update%";

-- 检查插入操作频次
SHOW STATUS LIKE "%Com_insert%";

-- 检查查询操作频次
SHOW STATUS LIKE "%Com_select%";

-- 检查删除操作频次
SHOW STATUS LIKE "%Com_delete%";
```

### 2. 核心引擎 (InnoDB) 行级监控

更能真实反映数据库底层压力的，是 **InnoDB 引擎自启动以来的底层行数影响状态**。

```sql
SHOW STATUS LIKE "%InnoDB_rows%";
```

**重点关注结果中的**：

| 指标 | 含义 |
| :--- | :--- |
| **`Innodb_rows_read`** | **读取行数** |
| **`Innodb_rows_updated`** | **更新行数** |
| **`Innodb_rows_inserted`** | **插入行数** |
| **`Innodb_rows_deleted`** | **删除行数** |

**为什么 `Innodb_rows_read` 特别重要**：它反映**真实扫描了多少行**。

**如果 `Innodb_rows_read` 远大于实际返回的行数，说明存在大量无效扫描 → 索引没用好。**

计算"读放大倍数"（理想值接近 1）：
```sql
SELECT
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_rows_read') AS rows_read;
```

## 二、索引使用频次监控

在上节课学习了利用 `EXPLAIN` 排查单条 SQL 后，我们还需要**从全局视角评估系统建立的索引是否真正发挥了作用**。

```sql
SHOW STATUS LIKE "%Handler_read%";
```

| 指标 | 含义 | 期望 |
| :--- | :--- | :--- |
| **`Handler_read_key`** | **通过索引被读取的次数** | **这个值越大越好**，说明你的查询**大量命中了索引** |
| **`Handler_read_rnd_next`** | **在数据文件中进行全表扫描读取下一行的请求数** | **这个值越小越好**，如果该值很高，说明系统**存在大量未走索引的全表扫描语句，急需优化** |

> [!important] 核心判断方法（记住）
> **看 `Handler_read_key` 和 `Handler_read_rnd_next` 的比值。**
>
> - `Handler_read_key` 高、`Handler_read_rnd_next` 低 → **索引用得健康**
> - `Handler_read_rnd_next` 持续高涨 → **有大量全表扫描，必须排查慢查询**

### 其他关键状态指标

| 指标 | 含义 |
| :--- | :--- |
| `Threads_connected` | 当前连接数 |
| `Threads_running` | 正在运行的线程数（**持续高说明有阻塞**） |
| `Innodb_buffer_pool_read_requests` | 缓冲池读请求数 |
| `Innodb_buffer_pool_reads` | **从磁盘物理读的次数**（越少越好） |
| `Table_locks_waited` | 表锁等待次数 |
| `Innodb_row_lock_waits` | **行锁等待次数**（高说明有锁竞争） |

## 三、慢查询日志 (Slow Query Log) 实战配置

找出那些**拖垮服务器的"毒瘤" SQL**，慢查询日志是第一阵地。

### 1. 检查慢查询状态与次数

```sql
-- 查看慢查询日志是否开启，以及日志文件的存储路径
SHOW VARIABLES LIKE "%quer%";

-- 查看自系统启动以来，慢查询发生的总次数
SHOW STATUS LIKE "%quer%";
```

### 2. 开启并配置慢查询时间阈值（CentOS 8 环境）

**配置文件位置（不同系统不一样）**：在 Linux 环境下，配置文件的位置与原笔记中的 Windows (`my.ini`) 完全不同。

**CentOS 8 的 MySQL 配置文件是 `/etc/my.cnf.d/mysql-server.cnf`**

```bash
# 1. 退出 MySQL，使用 vi 编辑 CentOS 的 MySQL 核心配置文件
vi /etc/my.cnf.d/mysql-server.cnf
```

在 `[mysqld]` 模块下方添加以下配置，**将慢查询的时间阈值修改为 6 秒**：

```ini
slow_query_log=1
slow_query_log_file=/var/log/mysql/mysql-slow.log
long_query_time=6
```

```bash
# 2. 重启 MySQL 服务器使配置生效 (切记使用 systemctl)
systemctl restart mysqld
```

> [!warning] 日志目录可能不存在
> 如果 `/var/log/mysql/` 不存在，MySQL 启动会失败。先创建并授权：
> ```bash
> mkdir -p /var/log/mysql
> chown mysql:mysql /var/log/mysql
> ```

```sql
-- mysql登录后可执行以下语句快速产生一条慢查询，方便测试
SELECT SLEEP(7);
```

**其他有用的慢查询参数**

| 参数 | 作用 |
| :--- | :--- |
| **`long_query_time=0.1`** | 生产环境建议调到 **0.1~1 秒**（6 秒太宽，抓不到问题） |
| `log_queries_not_using_indexes=1` | **记录所有没走索引的语句**（很有用，但量大） |
| `log_slow_admin_statements=1` | 记录慢的管理语句（ALTER 等） |
| `log_output=TABLE,FILE` | 同时输出到表和文件 |

也可以**动态开启不用重启**：
```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;
```

### 3. 分析慢查询日志

```bash
# 用 mysqldumpslow 汇总统计（按出现次数排序）
mysqldumpslow -s c -t 10 /var/log/mysql/mysql-slow.log

# 用 pt-query-digest 做专业分析（需装 percona-toolkit）
pt-query-digest /var/log/mysql/mysql-slow.log
```

## 四、表空间维护与检查

经过长时间的高频读写，数据表可能会产生**碎片**或出现**逻辑损坏**。

### 1. 检查表健康状态

用于检查表结构或视图是否存在损坏和报错。

```sql
CHECK TABLE user;
```

### 2. 优化表空间回收

随着数据的不断删除，表文件会产生**大量空洞**。

```sql
-- 尝试回收 user 表的碎片空间
OPTIMIZE TABLE user;
```

> [!important] InnoDB 的 `OPTIMIZE` 真相（面试考点，记住）
> 在 MySQL 8 中：
> - 如果是 **MyISAM 表**，此命令能**完美回收空间**
> - 但对于 **InnoDB 表**，**系统表空间 `ibdata1` 默认是无法直接缩容回收空间的**，通常需要通过**独立表空间 (`innodb_file_per_table`)** 机制结合**重建表**来解决
>
> **MySQL 8.0 中 `innodb_file_per_table` 默认为 ON**，所以 `OPTIMIZE TABLE` 对 InnoDB 实际是 **`ALTER TABLE ... FORCE`（重建表）**，**会锁表且耗时极长**。
>
> **生产环境替代方案**：
> ```bash
> pt-online-schema-change --alter "ENGINE=InnoDB" D=mydb,t=user --execute
> # 或 MySQL 8.0 原生算法（不锁写）
> ```
> ```sql
> ALTER TABLE user ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
> ```

## 五、紧急救援：MySQL 8.0 root 密码重置

在生产环境中，**密码丢失是最高级别的紧急事故**。在 CentOS 8 + MySQL 8 的体系下，我们必须掌握**跳过授权表的急救方案**。

### 1. 正常情况：密码没忘，仅作修改

```sql
-- MySQL 8.0 推荐的改密标准语法（需符合密码强度要求）
ALTER USER 'root'@'localhost' IDENTIFIED BY 'wei@12345';
```

### 2. 事故情况：密码彻底遗忘的破解预案

以下流程完全替代了原笔记中基于 Windows 和老版本 MySQL 的做法。

#### Step 1: 关闭 MySQL 服务

```bash
systemctl stop mysqld
```

#### Step 2: 修改配置文件，越过授权表启动母进程

打开 `/etc/my.cnf.d/mysql-server.cnf`，在 `[mysqld]` 下方添加一行：

```ini
skip-grant-tables
```

保存退出后，启动服务：

```bash
systemctl start mysqld
```

**`skip-grant-tables` 还可以顺手加 `skip-networking`**

```ini
skip-grant-tables
skip-networking
```
**`skip-networking` 会关闭 TCP 监听**，只留 socket —— 防止这期间**别人通过网络无密码连进来**（安全加固必备）。

#### Step 3: 无密码登录

```bash
mysql -uroot
```

#### Step 4: 刷新权限并强改密码

> [!important] MySQL 8.0 的关键差异（记住）
> **在 MySQL 8.0 中，即便跳过了授权表，也必须先刷新管理员权限，否则无法执行 `ALTER` 语句。**

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'wei@12345';
exit;
```

> [!warning] 如果 `ALTER USER` 还是报错
> 有些版本下需要先清空密码再改：
> ```sql
> FLUSH PRIVILEGES;
> ALTER USER 'root'@'localhost' IDENTIFIED BY '';      -- 先置空
> FLUSH PRIVILEGES;
> ALTER USER 'root'@'localhost' IDENTIFIED BY 'wei@12345';   -- 再设新密码
> ```

#### Step 5: 恢复安全配置并重启

再次打开 `/etc/mysql/my.cnf.d/mysql-server.cnf`，**删除或注释掉 `skip-grant-tables`**，然后最后一次重启服务。

```bash
systemctl restart mysqld
```

> [!danger] 千万别忘了删 skip-grant-tables（重要）
> **忘了删等于把数据库裸奔在网络上 —— 任何人不需要密码就能连进来。**
>
> 恢复后**必须验证**：
> ```bash
> mysql -uroot -p'wei@12345' -e "SELECT 1;"   # 需要密码才连得上才算成功
> ```

至此，系统恢复安全状态，新密码 `wei@12345` 生效！

---
