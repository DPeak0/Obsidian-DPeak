---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 全课程速查
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

## 课程目录速览

| 节次 | 章节名称 | 核心内容 |
| :--- | :--- | :--- |
| **第 1 节** | CentOS 8.4 环境搭建与 MySQL 8.0 基础启停 | 本地 dnf 光盘源配置与免网安装、systemctl 服务管控、MySQL 8.0 安全初始化配置 |
| **第 2 节** | 数据库设计、存储引擎与表结构管理 (DDL) | InnoDB 底层原理（废弃 `.frm`）、库表生命周期管理、表复制技术 |
| **第 3 节** | 数据类型、字段属性与字符集规范 | 数值/字符串/日期核心类型、unsigned/auto_increment 约束、utf8mb4 字符集解析、ALTER TABLE 高级管理 |
| **第 4 节** | 数据操作语言 (DML) 与基础查询 (DQL) | 标准的增删改查语法、DELETE 与 TRUNCATE 的底层差异剖析 |
| **第 5 节** | 数据过滤、排序与高级搜索技术 | WHERE 核心条件、模糊匹配与正则表达式 (REGEXP)、结果集去重与 LIMIT 分页技术 |
| **第 6 节** | MySQL 8.0 内置函数与实战应用 | 高频字符串清洗、数学极值计算、时间戳与日期转换函数 |
| **第 7 节** | 数据统计、分组聚合 (Group By) 与多表理论 | 五大聚合函数、GROUP BY 分组统计实战、多表关联理论与"笛卡尔积"性能灾难演示 |
| **第 8 节** | 多表嵌套子查询与表连接 (Join) 核心技术 | IN 嵌套查询、内连接 (INNER JOIN)、左/右外连接实战、视图 (VIEW) 虚拟化技术 |
| **第 9 节** | 索引管理 (Index) 与 SQL 执行效率分析 | 索引的分类与创建、复合索引最左前缀原则、使用 EXPLAIN 排查四大高频索引失效场景 |
| **第 10 节** | MySQL 服务器监控、慢查询优化与安全急救 | 全局状态监控 (SHOW STATUS)、慢查询日志配置、跳过授权表极限营救 MySQL 8.0 root 密码 |
| **第 11 节** | 核心事务机制 (Transaction) 与并发控制 | ACID 四大特性、事务开启/提交/回滚实战、四大隔离级别与 MySQL 8.0 默认隔离级别 (RR) |
| **第 12 节** | 触发器 (Trigger) 与存储过程 (Stored Procedure) | 触发器语法与 NEW/OLD 关键字、订单插入自动扣减库存实战；存储过程 IF/WHILE 控制结构、批量造数脚本封装 |
| **第 13 节** | MySQL 8.0 用户角色 (Role) 与细粒度权限管控 | 用户创建、密码过期策略、远程登录授权；角色创建、权限打包分配与激活机制；GRANT/REVOKE 表级细粒度权限 |
| **第 14 节** | Binlog 日志机制与数据备份恢复体系 | Binlog 开启与三大模式、mysqldump 全量备份与单表导出、基于时间点 (Point-in-Time) 的误删数据闪回 |
| **第 15 节** | 表分区技术 (Partitioning) 与海量数据架构展望 | RANGE 范围分区实战、HASH/LIST 分区简介、分区表维护 (ADD/DROP PARTITION) |

## MySQL 8.0 关键文件路径（CentOS 8）

| 路径 | 内容 |
| :--- | :--- |
| **`/etc/my.cnf.d/mysql-server.cnf`** | **主配置文件** |
| **`/var/lib/mysql/`** | **数据目录**（`.ibd`、Binlog 都在这里） |
| `/var/log/mysql/` | 慢查询日志（需自己创建） |
| `/var/log/mysqld.log` | 错误日志 |

## 各节核心命令速查

| 节 | 核心命令 |
| :--- | :--- |
| 1 | `dnf install mysql-server -y`、`systemctl start mysqld`、`mysql_secure_installation` |
| 2 | `CREATE DATABASE`、`CREATE TABLE`、`RENAME TABLE`、`CREATE TABLE x LIKE y` |
| 3 | `ALTER TABLE ... ADD/MODIFY/CHANGE/DROP` |
| 4 | `INSERT`、`UPDATE`、`DELETE`、`TRUNCATE` |
| 5 | `WHERE`、`BETWEEN`、`IN`、`LIKE`、`REGEXP`、`DISTINCT`、`ORDER BY`、`LIMIT` |
| 6 | `CONCAT`、`SUBSTRING`、`FROM_UNIXTIME`、`UNIX_TIMESTAMP`、`DATEDIFF` |
| 7 | `COUNT`、`SUM`、`AVG`、`MAX`、`MIN`、`GROUP BY`、`HAVING` |
| 8 | `INNER JOIN`、`LEFT JOIN`、`RIGHT JOIN`、`CREATE VIEW` |
| 9 | `CREATE INDEX`、`EXPLAIN`、`SHOW INDEX` |
| 10 | `SHOW STATUS`、`slow_query_log`、`CHECK TABLE`、`OPTIMIZE TABLE` |
| 11 | `START TRANSACTION`、`COMMIT`、`ROLLBACK`、`SET transaction_isolation` |
| 12 | `CREATE TRIGGER`、`CREATE PROCEDURE`、`DELIMITER`、`CALL` |
| 13 | `CREATE USER`、`GRANT`、`REVOKE`、`CREATE ROLE`、`SET DEFAULT ROLE` |
| 14 | `mysqldump`、`mysqlbinlog`、`SHOW BINARY LOGS` |
| 15 | `PARTITION BY RANGE`、`DROP PARTITION`、`REORGANIZE PARTITION` |

## 高频面试/考试点汇总

> [!important] 必须记住的十个点
> 1. **MySQL 8.0 废弃 `.frm`**，表结构存在 `.ibd`（InnoDB）/ `.sdi`（MyISAM）
> 2. **`utf8` 是残缺的（3 字节）**，**`utf8mb4` 才是真 UTF-8**
> 3. **`DELETE` 不重置自增，`TRUNCATE` 重置且不可回滚**
> 4. **`COUNT(*)` 统计所有行，`COUNT(col)` 跳过 NULL**
> 5. **`WHERE` 在分组前，`HAVING` 在分组后；`WHERE` 不能用聚合函数**
> 6. **统计"每组含 0"要用 `LEFT JOIN` + `COUNT(从表字段)`**
> 7. **复合索引最左前缀** —— 跳过最左列必然失效
> 8. **索引失效四大坑**：`LIKE '%x'`、`OR` 混非索引列、**隐式类型转换**、`IS NULL`
> 9. **MySQL 默认隔离级别是 RR**，且用**间隙锁**额外解决了幻读
> 10. **角色 (Role) 授予后默认不激活** —— 需要 `SET DEFAULT ROLE` 或 `activate_all_roles_on_login=ON`

## 常见坑汇总

> [!warning] 实操最容易踩的
> 1. **改完 `/etc/my.cnf.d/*.cnf` 忘了 `systemctl restart mysqld`**
> 2. **`UPDATE`/`DELETE` 忘加 `WHERE`** —— **生产头号事故**
> 3. **`TRUNCATE` 不可回滚** —— 想删干净先想清楚
> 4. **`SELECT *`** —— 拖慢查询、无法用覆盖索引
> 5. **`ORDER BY RAND()` 大表用** —— 直接拖垮数据库
> 6. **`LIMIT 1000000, 10` 深分页** —— 要先扫 100 万行
> 7. **`COUNT(*)` 统计含 0 的分组** —— 结果会多 1
> 8. **`NOT IN` 子查询含 NULL** —— 整个查询返回空
> 9. **字符串字段不加引号** —— 隐式转换导致索引失效
> 10. **索引列上做函数运算** —— `WHERE YEAR(dt)=2026` 索引失效
> 11. **`mysqldump` 不加 `--single-transaction`** —— 锁表导致业务假死
> 12. **忘了删 `skip-grant-tables`** —— 数据库无密码裸奔
> 13. **角色授予后没激活** —— 用户登录啥权限都没有
> 14. **`binlog_expire_logs_seconds` 设太短** —— 出事了日志已被清理，无法闪回
> 15. **`DROP PARTITION` 不可回滚** —— 删之前先备份
