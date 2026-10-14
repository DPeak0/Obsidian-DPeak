---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 15 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：应对**单表数据量突破千万级**后的物理存储与查询瓶颈。掌握 MySQL 8.0 表分区的核心概念与底层逻辑，熟练配置企业中最常用的 **RANGE 范围分区**处理时间序列日志，了解 HASH/LIST 场景，并掌握分区表的动态维护技术。

## 一、表分区核心概念：突破千万级单表存储瓶颈

当单张 InnoDB 表的数据量**超过 1000 万行**时，即使拥有完美的 B+Tree 索引，深达三四层的索引树也会导致**严重的磁盘 I/O 负担**。

表分区是在**物理层面上**将一张大表的数据和索引**拆分成多个更小的、可独立管理的文件片段**，但**在逻辑层面上，开发人员面对的依然是一张完整的表**。

| 特性 | 说明 |
| :--- | :--- |
| **物理隔离** | 不同的分区可以**跨磁盘分布**，有效分散系统的底层存储 I/O 压力 |
| **查询裁剪 (Partition Pruning)** | MySQL 优化器会**自动过滤掉不包含目标数据的分区**，仅在特定分区内进行搜索，极大提升检索速度 |
| **无缝衔接** | 对应用层完全透明，前端业务代码的 CRUD 语句**无需进行任何修改** |

**分区 vs 分表 vs 分库**

| 方案 | 层面 | 对应用 | 说明 |
| :--- | :--- | :--- | :--- |
| **分区** | **物理文件** | **完全透明** | 还是**一张表**，逻辑不变 |
| **分表** | 表结构 | **要改代码** | 拆成 `user_01`、`user_02`... |
| **分库** | 数据库实例 | **要改代码 + 中间件** | 拆到不同 MySQL 实例 |

## 二、RANGE 范围分区实战：时间序列数据的切割

RANGE 分区是企业数据中心**应用最广的模式**，极其适合**按日期或自增 ID 来归档历史流水、操作日志**等海量时间序列数据。

```sql
-- 创建一张以年份范围进行分区的系统访问日志表
CREATE TABLE system_logs (
    log_id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    client_ip VARCHAR(50) NOT NULL,
    access_time DATETIME NOT NULL,
    PRIMARY KEY (log_id, access_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY RANGE (YEAR(access_time)) (
    PARTITION p_2024 VALUES LESS THAN (2025),
    PARTITION p_2025 VALUES LESS THAN (2026),
    PARTITION p_2026 VALUES LESS THAN (2027),
    PARTITION p_max VALUES LESS THAN MAXVALUE
);
```

> [!important] 分区表的一个硬性约束（记住）
> **在 MySQL 分区表中，分区键必须包含在表的主键或唯一约束之中。**
> （如上例**必须将 `access_time` 纳入复合主键**）

### 用递归 CTE 批量插入 100 条数据

用 MySQL 8.0 的**递归 CTE** 一次性插入 100 条数据，并让它们**均匀分布在四个分区中**（每个分区 25 条）。

```sql
INSERT INTO system_logs (client_ip, access_time)
WITH RECURSIVE seq AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 100
)
SELECT
    CONCAT('192.168.1.', n) AS client_ip,
    CASE
        WHEN n <= 25 THEN DATE_ADD('2024-01-01', INTERVAL n DAY)
        WHEN n <= 50 THEN DATE_ADD('2025-01-01', INTERVAL (n - 25) DAY)
        WHEN n <= 75 THEN DATE_ADD('2026-01-01', INTERVAL (n - 50) DAY)
        ELSE DATE_ADD('2027-01-01', INTERVAL (n - 75) DAY)
    END AS access_time
FROM seq;
```

**说明**：

- `n` 从 1 到 100
- 前 25 条 → 2024 年，进入 `p_2024`
- 第 26–50 条 → 2025 年，进入 `p_2025`
- 第 51–75 条 → 2026 年，进入 `p_2026`
- 第 76–100 条 → 2027 年，进入 **`p_max`**（因为 `p_max` 是 `LESS THAN MAXVALUE`，包含 2027 及以后）
- `log_id` 是自增列，不需要指定，MySQL 会自动生成，且与 `access_time` 组合满足主键唯一

### 验证各分区数据量

```sql
SELECT 'p_2024' AS partition_name, COUNT(*) AS rows_count FROM system_logs PARTITION (p_2024)
UNION ALL
SELECT 'p_2025', COUNT(*) FROM system_logs PARTITION (p_2025)
UNION ALL
SELECT 'p_2026', COUNT(*) FROM system_logs PARTITION (p_2026)
UNION ALL
SELECT 'p_max', COUNT(*) FROM system_logs PARTITION (p_max);
```

### 查看单个分区的数据

```text
mysql> SELECT 'p_2025', COUNT(*) FROM system_logs PARTITION (p_2025);
+--------+----------+
| p_2025 | COUNT(*) |
+--------+----------+
| p_2025 |       25 |
+--------+----------+
1 row in set (0.01 sec)

mysql> SELECT * FROM system_logs PARTITION (p_2025);
+--------+--------------+---------------------+
| log_id | client_ip    | access_time         |
+--------+--------------+---------------------+
|     26 | 192.168.1.26 | 2025-01-02 00:00:00 |
|     27 | 192.168.1.27 | 2025-01-03 00:00:00 |
|     28 | 192.168.1.28 | 2025-01-04 00:00:00 |
|     29 | 192.168.1.29 | 2025-01-05 00:00:00 |
|     30 | 192.168.1.30 | 2025-01-06 00:00:00 |
|     31 | 192.168.1.31 | 2025-01-07 00:00:00 |
|     32 | 192.168.1.32 | 2025-01-08 00:00:00 |
|     33 | 192.168.1.33 | 2025-01-09 00:00:00 |
|     34 | 192.168.1.34 | 2025-01-10 00:00:00 |
|     35 | 192.168.1.35 | 2025-01-11 00:00:00 |
|     36 | 192.168.1.36 | 2025-01-12 00:00:00 |
|     37 | 192.168.1.37 | 2025-01-13 00:00:00 |
|     38 | 192.168.1.38 | 2025-01-14 00:00:00 |
|     39 | 192.168.1.39 | 2025-01-15 00:00:00 |
|     40 | 192.168.1.40 | 2025-01-16 00:00:00 |
|     41 | 192.168.1.41 | 2025-01-17 00:00:00 |
|     42 | 192.168.1.42 | 2025-01-18 00:00:00 |
|     43 | 192.168.1.43 | 2025-01-19 00:00:00 |
|     44 | 192.168.1.44 | 2025-01-20 00:00:00 |
|     45 | 192.168.1.45 | 2025-01-21 00:00:00 |
|     46 | 192.168.1.46 | 2025-01-22 00:00:00 |
|     47 | 192.168.1.47 | 2025-01-23 00:00:00 |
|     48 | 192.168.1.48 | 2025-01-24 00:00:00 |
|     49 | 192.168.1.49 | 2025-01-25 00:00:00 |
|     50 | 192.168.1.50 | 2025-01-26 00:00:00 |
+--------+--------------+---------------------+
25 rows in set (0.00 sec)
```

### 查看表的分区信息

```text
mysql> SHOW CREATE TABLE system_logs\G
*************************** 1. row ***************************
       Table: system_logs
Create Table: CREATE TABLE `system_logs` (
  `log_id` int unsigned NOT NULL AUTO_INCREMENT,
  `client_ip` varchar(50) NOT NULL,
  `access_time` datetime NOT NULL,
  PRIMARY KEY (`log_id`,`access_time`)
) ENGINE=InnoDB AUTO_INCREMENT=101 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
/*!50100 PARTITION BY RANGE (year(`access_time`))
(PARTITION p_2024 VALUES LESS THAN (2025) ENGINE = InnoDB,
 PARTITION p_2025 VALUES LESS THAN (2026) ENGINE = InnoDB,
 PARTITION p_2026 VALUES LESS THAN (2027) ENGINE = InnoDB,
 PARTITION p_max VALUES LESS THAN MAXVALUE ENGINE = InnoDB) */
1 row in set (0.00 sec)
```

**查分区信息的其他方式**

```sql
SELECT * FROM information_schema.PARTITIONS WHERE TABLE_NAME='system_logs'\G
SELECT PARTITION_NAME, TABLE_ROWS FROM information_schema.PARTITIONS
  WHERE TABLE_NAME='system_logs';
```

## 三、HASH 与 LIST 分区场景简介

面对无法按连续区间划分的数据，MySQL 提供了额外的分区策略应对特殊业务：

| 分区类型 | 原理 | 适用场景 |
| :--- | :--- | :--- |
| **HASH 分区** | 通过对分区键（如 `user_id`）进行**哈希取模计算**，将数据**均匀打散**到预设数量的分区中 | 适合**数据分布极为随机的高并发写入**场景，可有效**缓解单点并发热点** |
| **LIST 分区** | 基于**枚举值列表**进行归类。例如，根据**省份代码**（如区域编号为 11、31、44）将全国业务数据在物理文件上进行**严密隔离** | 按**离散枚举值**归档 |

```sql
-- HASH 分区示例：按 user_id 散列到 4 个分区
CREATE TABLE t_hash (
    id INT NOT NULL,
    user_id INT NOT NULL,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH(user_id) PARTITIONS 4;

-- LIST 分区示例：按省份代码分区
CREATE TABLE t_list (
    id INT NOT NULL,
    province INT NOT NULL,
    PRIMARY KEY (id, province)
) PARTITION BY LIST(province) (
    PARTITION p_north VALUES IN (11, 12, 13),
    PARTITION p_east  VALUES IN (31, 32, 33),
    PARTITION p_south VALUES IN (44, 45, 46)
);
```

## 四、分区表的日常运维与架构展望

随着时间推移，DBA 团队需要对分区进行**动态扩容**或对**冷数据进行淘汰清理**。

### 1. 动态淘汰历史过期的分区

```sql
-- 其包含的数据也会被瞬间物理清除，资源回收效率远超 DELETE
ALTER TABLE system_logs DROP PARTITION p_2024;
```

> [!important] `DROP PARTITION` 是"秒删"（记住）
> **它是 DDL 操作，直接删除物理文件** —— 删除 1000 万行和一行的耗时几乎一样。
>
> ⚠️ **但也因此不可回滚、不写完整 Binlog（只能靠 row 事件）**。**删之前务必备份。**

### 2. 动态拆分或追加新的未来时间范围分区

```sql
-- 避免新数据无家可归
ALTER TABLE system_logs REORGANIZE PARTITION p_max INTO (
    PARTITION p_2027 VALUES LESS THAN (2028),
    PARTITION p_max VALUES LESS THAN MAXVALUE
);
```

### 3. 插入 p_max 年的数据

```sql
-- 插入10条p_max年的数据
INSERT INTO system_logs (client_ip, access_time)
WITH RECURSIVE seq AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 10
)
SELECT
    CONCAT('10.0.0.', n) AS client_ip,
    DATE_ADD('2028-01-01', INTERVAL n DAY) AS access_time
FROM seq;
```

### 4. 查看 p_2027 的数据

```text
mysql> select * from system_logs PARTITION (p_2027);
+--------+---------------+---------------------+
| log_id | client_ip     | access_time         |
+--------+---------------+---------------------+
|     76 | 192.168.1.76  | 2027-01-02 00:00:00 |
|     77 | 192.168.1.77  | 2027-01-03 00:00:00 |
|     78 | 192.168.1.78  | 2027-01-04 00:00:00 |
|     79 | 192.168.1.79  | 2027-01-05 00:00:00 |
|     80 | 192.168.1.80  | 2027-01-06 00:00:00 |
|     81 | 192.168.1.81  | 2027-01-07 00:00:00 |
|     82 | 192.168.1.82  | 2027-01-08 00:00:00 |
|     83 | 192.168.1.83  | 2027-01-09 00:00:00 |
|     84 | 192.168.1.84  | 2027-01-10 00:00:00 |
|     85 | 192.168.1.85  | 2027-01-11 00:00:00 |
|     86 | 192.168.1.86  | 2027-01-12 00:00:00 |
|     87 | 192.168.1.87  | 2027-01-13 00:00:00 |
|     88 | 192.168.1.88  | 2027-01-14 00:00:00 |
|     89 | 192.168.1.89  | 2027-01-15 00:00:00 |
|     90 | 192.168.1.90  | 2027-01-16 00:00:00 |
|     91 | 192.168.1.91  | 2027-01-17 00:00:00 |
|     92 | 192.168.1.92  | 2027-01-18 00:00:00 |
|     93 | 192.168.1.93  | 2027-01-19 00:00:00 |
|     94 | 192.168.1.94  | 2027-01-20 00:00:00 |
|     95 | 192.168.1.95  | 2027-01-21 00:00:00 |
|     96 | 192.168.1.96  | 2027-01-22 00:00:00 |
|     97 | 192.168.1.97  | 2027-01-23 00:00:00 |
|     98 | 192.168.1.98  | 2027-01-24 00:00:00 |
|     99 | 192.168.1.99  | 2027-01-25 00:00:00 |
|    100 | 192.168.1.100 | 2027-01-26 00:00:00 |
+--------+---------------+---------------------+
25 rows in set (0.00 sec)
```

### 5. 查看 p_max（即 2028 年）的数据

```text
mysql> select * from system_logs PARTITION (p_max);
+--------+-----------+---------------------+
| log_id | client_ip | access_time         |
+--------+-----------+---------------------+
|    101 | 10.0.0.1  | 2028-01-02 00:00:00 |
|    102 | 10.0.0.2  | 2028-01-03 00:00:00 |
|    103 | 10.0.0.3  | 2028-01-04 00:00:00 |
|    104 | 10.0.0.4  | 2028-01-05 00:00:00 |
|    105 | 10.0.0.5  | 2028-01-06 00:00:00 |
|    106 | 10.0.0.6  | 2028-01-07 00:00:00 |
|    107 | 10.0.0.7  | 2028-01-08 00:00:00 |
|    108 | 10.0.0.8  | 2028-01-09 00:00:00 |
|    109 | 10.0.0.9  | 2028-01-10 00:00:00 |
|    110 | 10.0.0.10 | 2028-01-11 00:00:00 |
+--------+-----------+---------------------+
10 rows in set (0.00 sec)
```

### 6. 查看最终分区情况

```text
mysql> SHOW CREATE TABLE system_logs\G
*************************** 1. row ***************************
       Table: system_logs
Create Table: CREATE TABLE `system_logs` (
  `log_id` int unsigned NOT NULL AUTO_INCREMENT,
  `client_ip` varchar(50) NOT NULL,
  `access_time` datetime NOT NULL,
  PRIMARY KEY (`log_id`,`access_time`)
) ENGINE=InnoDB AUTO_INCREMENT=111 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
/*!50100 PARTITION BY RANGE (year(`access_time`))
(PARTITION p_2025 VALUES LESS THAN (2026) ENGINE = InnoDB,
 PARTITION p_2026 VALUES LESS THAN (2027) ENGINE = InnoDB,
 PARTITION p_2027 VALUES LESS THAN (2028) ENGINE = InnoDB,
 PARTITION p_max VALUES LESS THAN MAXVALUE ENGINE = InnoDB) */
1 row in set (0.00 sec)
```

对比可见：**`p_2024` 已被 `DROP`，`p_max` 已被 `REORGANIZE` 拆出 `p_2027`**。

### 分区维护命令速查

| 命令 | 作用 |
| :--- | :--- |
| **`ALTER TABLE t DROP PARTITION p;`** | **删除分区**（秒删数据） |
| **`ALTER TABLE t REORGANIZE PARTITION p_max INTO (...);`** | **拆分/追加分区** |
| `ALTER TABLE t ADD PARTITION (PARTITION ...);` | **追加分区**（只能加在 **RANGE 最后**） |
| `ALTER TABLE t TRUNCATE PARTITION p;` | 清空分区数据（保留分区） |
| `ALTER TABLE t REMOVE PARTITIONING;` | **移除分区**（变回普通表，**会重建全表**） |
| `EXPLAIN SELECT ... ` | 看是否触发**分区裁剪** |

> [!important] 分区表的重要限制（记住）
> 1. **分区键必须在主键/唯一键中**（否则建表报错）
> 2. **一张表最多 8192 个分区**
> 3. **分区表不支持外键**
> 4. **分区键不能用某些函数**（如 `RANGE(YEAR(dt))` 可用，但**分区裁剪要求条件能推导出分区范围**）
> 5. **分区裁剪需要条件直接作用在分区键上**：
>    ```sql
>    -- ✅ 能裁剪，只扫 p_2026
>    WHERE access_time >= '2026-01-01' AND access_time < '2027-01-01'
>    -- ❌ 无法裁剪，扫所有分区
>    WHERE YEAR(access_time) = 2026
>    ```

## 五、架构展望

在现代 ICT 技术体系中，**单机 MySQL 表分区技术仅仅是解决数据膨胀的初步手段**。这套切分理念将作为坚实的底层支撑，无缝衔接至未来更高阶的**云原生基础架构**中：

- 无论是将**无状态的 MySQL 实例放入 Kubernetes 与 Docker 容器**中进行敏捷编排
- 还是将**底层数据文件桥接至 Ceph 分布式存储池**以实现**计算与存储的彻底分离**

**数据路由与切分的底层逻辑都一脉相承。**

---
