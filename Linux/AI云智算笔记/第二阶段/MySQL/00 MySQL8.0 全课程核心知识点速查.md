---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 00 MySQL 命令速查
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---
## 一、服务与登录（Linux 端）

| 命令 | 说明 |
| :--- | :--- |
| `systemctl start mysqld` | 启动 MySQL（**服务名是 mysqld**） |
| `systemctl stop mysqld` | 停止服务 |
| `systemctl restart mysqld` | 重启（**改完 .cnf 配置必须执行**） |
| `systemctl status mysqld` | 查看状态，`active (running)` 为正常 |
| `systemctl enable mysqld` | 设置开机自启 |
| `mysql -u root` | 免密登录（CentOS 8 dnf 安装版本的特性） |
| `mysql -u root -p` | 标准登录，回车后再输密码 |
| `mysql -u root -p'密码' -h 127.0.0.1 -P 3306` | 指定主机端口登录（走 TCP 而非 socket） |
| `mysql -u root -p -e "SHOW DATABASES;"` | **不进交互界面**，执行一条语句就退出 |
| `exit` / `\q` / `Ctrl+D` | 退出客户端 |
| `\c` | 输入到一半**清空当前行** |
| `\G` | 结果**由横表转纵向**（看长语句必用，结尾替代 `;`） |

```bash
# MySQL 8.0 默认只监听 socket，要远程访问必须放开端口
vi /etc/my.cnf.d/mysql-server.cnf
[mysqld]
port=3306
bind-address=0.0.0.0
systemctl restart mysqld
```

## 二、库操作（Database）

| 命令                                                                             | 说明                      |
| :----------------------------------------------------------------------------- | :---------------------- |
| `SHOW DATABASES;`                                                              | 列出所有库                   |
| `CREATE DATABASE 库名;`                                                          | 创建数据库                   |
| `CREATE DATABASE 库名 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;` | **建库推荐写法**，显式指定字符集防中文乱码 |
| `USE 库名;`                                                                      | 切换当前库                   |
| `SHOW CREATE DATABASE 库名\G`                                                    | 查看建库语句                  |
| `DROP DATABASE 库名;`                                                            | 删库（**连同库下所有表，不可逆**）     |

## 三、表操作（Table）

| 命令                                                           | 说明                                          |
| :----------------------------------------------------------- | :------------------------------------------ |
| `SHOW TABLES;`                                               | 列出当前库所有表                                    |
| `DESC 表名;`                                                   | 查看表结构（等价 `DESCRIBE`、`SHOW COLUMNS FROM 表名`） |
| `SHOW CREATE TABLE 表名\G`                                     | 查看**完整建表语句**（含引擎、字符集、分区）                    |
| `CREATE TABLE 表名 (…) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;` | 创建数据表                                       |
| `CREATE TABLE 新表 LIKE 旧表;`                                   | **只复制结构**（字段+索引，不含数据）                       |
| `INSERT INTO 新表 SELECT * FROM 旧表;`                           | **只复制数据**，配合上一条完成整表克隆                       |
| `CREATE TABLE 新表 AS SELECT * FROM 旧表;`                       | 结构+数据一步到位（**不带索引/自增**，慎用）                   |
| `RENAME TABLE 旧 TO 新;`                                       | 修改表名                                        |
| `DROP TABLE 表名;`                                             | 删表（数据一起删）                                   |
| `DROP TABLE IF EXISTS 表名;`                                   | 表不存在也不报错，批量脚本常用                             |
| `TRUNCATE TABLE 表名;`                                         | **清空整表并重置自增**，速度远快于 DELETE                  |
| `CHECK TABLE 表名;`                                            | 检查表是否损坏                                     |
| `OPTIMIZE TABLE 表名;`                                         | 回收碎片空间（MyISAM 有效）                           |


## 四、字段管理（ALTER TABLE）

| 命令 | 说明 |
| :--- | :--- |
| `ALTER TABLE 表 ADD 列 类型 约束;` | 添加字段（默认加在**最后**） |
| `ALTER TABLE 表 ADD 列 类型 AFTER 某列;` | 插到指定字段**之后** |
| `ALTER TABLE 表 ADD 列 类型 FIRST;` | 放到**第一列** |
| `ALTER TABLE 表 DROP 列;` | 删除字段 |
| `ALTER TABLE 表 MODIFY 列 新类型 约束;` | 改**类型/约束**，**不能改列名** |
| `ALTER TABLE 表 CHANGE 旧列 新列 类型 约束;` | **改列名 + 类型**一起改 |
| `ALTER TABLE 表 RENAME COLUMN 旧 TO 新;` | 只改列名（MySQL 8.0） |
| `ALTER TABLE 表 ADD PRIMARY KEY (列);` | 添加主键 |
| `ALTER TABLE 表 DROP PRIMARY KEY;` | 删除主键 |

> [!warning] MODIFY 会覆盖原有属性
> `MODIFY` 必须把该列**所有属性写全**，否则没写的（如 `NOT NULL`、`DEFAULT`、`AUTO_INCREMENT`）会**被清掉**。

## 五、索引管理（INDEX）

| 命令                                 | 说明                             |
| :--------------------------------- | :----------------------------- |
| `CREATE INDEX 索引名 ON 表(列);`        | 建**普通索引**，无任何限制                |
| `CREATE UNIQUE INDEX 索引名 ON 表(列);` | 建**唯一索引**，值不可重复、**允许 NULL**    |
| `CREATE INDEX 索引名 ON 表(列1, 列2);`   | 建**复合索引**，遵循最左前缀               |
| `ALTER TABLE 表 ADD INDEX 索引名(列);`  | 建索引的另一种写法                      |
| `ALTER TABLE 表 ADD UNIQUE(列);`     | 加唯一约束                          |
| `DROP INDEX 索引名 ON 表;`             | 删除索引                           |
| `ALTER TABLE 表 DROP INDEX 索引名;`    | 删除索引（等价写法）                     |
| `SHOW INDEX FROM 表\G`              | 查看表上所有索引（**必须用 `\G`**，否则列挤成一团） |
|                                    |                                |

> [!warning] 复合索引最左前缀
> 索引 `(class_id, age)` 下：`WHERE class_id=1` ✅ ／ `WHERE class_id=1 AND age=19` ✅ ／ **`WHERE age=19` ❌ 跳过最左列 → 全表扫描**。
> 另外**范围查询会中断**：`WHERE a=1 AND b>2 AND c=3` 只用到 a、b，**c 用不上** —— 等值字段放前面，范围字段放最后。

## 六、数据操作（DML）

| 命令                                     | 说明                               |     |
| :------------------------------------- | :------------------------------- | --- |
| `INSERT INTO 表(列) VALUES (值);`         | 单行插入                             |     |
| `INSERT INTO 表(列) VALUES (值),(值),(值);` | **批量插入**，造测试数据最快方式               |     |
| `INSERT INTO 表 SET 列=值;`               | 单行的另一种写法                         |     |
| `UPDATE 表 SET 列=值 WHERE 条件;`           | 按条件改数据                           |     |
| `UPDATE 表 SET 列1=值1, 列2=值2 WHERE 条件;`  | 同时改多个字段                          |     |
| `DELETE FROM 表 WHERE 条件;`              | 按条件删行（**保留自增计数**、**可回滚**）        |     |
| `DELETE FROM 表;`                       | 删光数据，**自增不重置**                   |     |
| `TRUNCATE TABLE 表;`                    | 删光数据并**重置自增**，底层是"删表重建"，**不可回滚** |     |

> [!danger] 生产铁律
> **`UPDATE` / `DELETE` 必须带 `WHERE`**，否则整表数据被覆盖/清空 —— 头号事故。
> 稳妥习惯：**先 `SELECT` 确认范围 → 改写成 `DELETE` 执行 → 最后才改成 `UPDATE`**。

## 七、查询基础（DQL）

| 命令                                     | 说明                          |
| :------------------------------------- | :-------------------------- |
| `SELECT * FROM 表;`                     | 查全部字段（**性能差**，只取需要的列）       |
| `SELECT 列1, 列2 FROM 表;`                | 指定字段，降低 I/O 与网络传输           |
| `SELECT DISTINCT 列 FROM 表;`            | **去重**                      |
| `SELECT 列 AS 别名 FROM 表;`               | 起别名（`AS` 可省略：`SELECT 列 别名`） |
| `SELECT CONCAT('前缀', 列) AS 别名 FROM 表;` | 拼接后输出，报表格式化常用               |

## 八、条件过滤（WHERE）

| 写法 | 说明 |
| :--- | :--- |
| `WHERE 列 > / < / >= / <= / = / != / <>` | 比较运算符（`<>` 与 `!=` 等价） |
| `WHERE 条件1 AND 条件2` | 与 |
| `WHERE 条件1 OR 条件2` | 或 |
| `WHERE 列 BETWEEN 3 AND 7` | **闭区间**，等价 `>=3 AND <=7` |
| `WHERE 列 IN (3,5,6,10)` | 离散集合，替代一长串 `OR` |
| `WHERE 列 IS NULL` / `IS NOT NULL` | 判空，**绝不能写 `= NULL`** |
| `WHERE 列 LIKE '%关键字%'` | 模糊匹配，`%` 任意长度、`_` 单个字符 |
| `WHERE 列 NOT LIKE 'user%'` | 反向模糊匹配 |
| `WHERE 列 REGEXP '^php'` | 正则（`^` 开头、`$` 结尾、`或` 用竖线分隔） |
| `WHERE 条件 HAVING 聚合条件` | **分组后**过滤，可使用聚合函数 |

## 九、排序 / 分页 / 结果集控制

| 命令 | 说明 |
| :--- | :--- |
| `ORDER BY 列` | 升序（**默认 ASC**，可省略） |
| `ORDER BY 列 ASC` | 升序，从小到大 |
| `ORDER BY 列 DESC` | 降序，常用于"最新在前" |
| `ORDER BY 列1, 列2 DESC` | 多字段排序 |
| `ORDER BY RAND()` | 随机排序（**大表杀手**），配合 LIMIT 抽奖 |
| `LIMIT 5` | 取前 5 条 |
| `LIMIT 0, 5` | 从第 0 条开始取 5 条（与上等价） |
| `LIMIT 3, 3` | **分页核心**：跳过前 3 条，取第 4-6 条 |
| `LIMIT 5 OFFSET 3` | 同上，`OFFSET` 写法更易读 |

## 十、聚合函数与分组（GROUP BY）

| 函数 | 说明 |
| :--- | :--- |
| `COUNT(*)` | 统计**所有行**（含 NULL 行），最常用 |
| `COUNT(列)` | 统计该列**非 NULL** 的行数 |
| `COUNT(DISTINCT 列)` | 统计去重后的数量 |
| `SUM(列)` / `AVG(列)` | 求和 / 平均值（仅数值型） |
| `MAX(列)` / `MIN(列)` | 最大 / 最小值 |

| 命令 | 说明 |
| :--- | :--- |
| `GROUP BY 列` | 按列分组，配合聚合函数统计 |
| `SELECT 列, COUNT(*) FROM t GROUP BY 列;` | **每组统计**的标准写法 |
| `… GROUP BY 列 HAVING COUNT(*) > 3;` | 分组后再筛（**WHERE 在分组前，HAVING 在分组后**） |
| `SELECT GROUP_CONCAT(列) FROM t GROUP BY 分组列;` | 把每组的值**拼成一行字符串** |

## 十一、多表查询 / 子查询 / JOIN

| 写法                                              | 说明                          |
| :---------------------------------------------- | :-------------------------- |
| `SELECT … FROM a, b WHERE a.id = b.aid;`        | 隐式连接，**必须加关联条件**，否则笛卡尔积     |
| `SELECT … FROM a INNER JOIN b ON a.id = b.aid;` | **内连接**，只返回两表匹配的行（**推荐写法**） |
| `SELECT … FROM a LEFT JOIN b ON a.id = b.aid;`  | **左外连接**，左表全保留、右表补 NULL     |
| `SELECT … FROM a RIGHT JOIN b ON a.id = b.aid;` | 右外连接，调换表位置即等价于左连接           |
| `SELECT … FROM a JOIN b ON … JOIN c ON …;`      | 多表连查                        |
| `SELECT … WHERE 列 IN (SELECT 列 FROM …);`        | **嵌套子查询**，子结果作为外层条件         |
| `SELECT … WHERE 列 = (SELECT …);`                | 标量子查询（子查询只返回一行一列）           |
| `SELECT … WHERE EXISTS (SELECT …);`             | 存在性判断，大数据量下比 `IN` 快         |

> [!warning] 统计"数量为 0 的组"
> `GROUP BY` 查不出 0 人的组。**用 `LEFT JOIN` + `COUNT(从表字段)`**（如 `COUNT(user.id)`），NULL 会计为 0。

## 十二、视图（VIEW）

| 命令 | 说明 |
| :--- | :--- |
| `CREATE VIEW 视图名 AS SELECT …;` | 把复杂查询封装成**虚拟表** |
| `SELECT * FROM 视图名;` | 像查表一样查视图 |
| `SHOW FULL TABLES WHERE Table_type = 'VIEW';` | 区分显示表和视图 |
| `SHOW CREATE VIEW 视图名\G` | 查看视图定义 |
| `DROP VIEW 视图名;` | 删除视图（**不影响底层真实表**） |

> 视图**不存储数据**，只存 SQL 逻辑；底层表数据变了，视图结果**实时跟着变**。

## 十三、常用内置函数

### 字符串

| 函数 | 说明 |
| :--- | :--- |
| `CONCAT(a, b)` / `CONCAT_WS('分隔符', a, b)` | 拼接（带/不带分隔符） |
| `LCASE(s)` / `UCASE(s)` | 转小写 / 大写（等价 `LOWER` / `UPPER`） |
| `LENGTH(s)` | **字节数**（UTF-8 中文占 3 字节） |
| `CHAR_LENGTH(s)` | **字符个数** |
| `SUBSTRING(s, 起始位置, 长度)` | 截取，位置从 **1** 开始；负数表示倒着数 |
| `TRIM(s)` / `LTRIM(s)` / `RTRIM(s)` | 去空格（全部 / 左 / 右） |
| `REPLACE(s, 旧, 新)` | 替换子串 |
| `REPEAT(s, n)` / `SPACE(n)` | 重复 n 次 / 生成 n 个空格 |
| `INSTR(s, 子串)` / `LOCATE(子串, s)` | 查子串位置，找不到返回 0 |

### 数学

| 函数 | 说明 |
| :--- | :--- |
| `CEILING(x)` / `FLOOR(x)` | 向上 / 向下取整 |
| `ROUND(x, 位数)` | 四舍五入 |
| `TRUNCATE(x, 位数)` | 直接截断，**不四舍五入** |
| `ABS(x)` | 绝对值 |
| `RAND()` | 0~1 随机数 |
| `BIN(n)` / `SQRT(x)` | 转二进制 / 开平方 |

### 日期与时间

| 函数 | 说明 |
| :--- | :--- |
| `NOW()` / `SYSDATE()` | 当前日期+时间 `YYYY-MM-DD HH:MM:SS` |
| `CURDATE()` / `CURTIME()` | 只取当前日期 / 只取当前时间 |
| `UNIX_TIMESTAMP()` | 当前**时间戳**（1970-01-01 起的秒数） |
| `FROM_UNIXTIME(秒数)` | 时间戳 → **日期字符串** |
| `DATE_FORMAT(日期, '%Y-%m-%d')` | 自定义格式化输出 |
| `YEAR(d)` / `MONTH(d)` / `DAY(d)` / `WEEK(d)` | 取年 / 月 / 日 / 第几周 |
| `DATE(日期时间)` | 截掉时间部分 |
| `DATEDIFF(d1, d2)` | **相差天数**（d1 − d2） |
| `TIMEDIFF(t1, t2)` | 相差时间 |
| `DATE_ADD(日期, INTERVAL n DAY)` | 日期加减（`DAY`/`MONTH`/`YEAR`/`HOUR`…） |
| `STR_TO_DATE(s, 格式)` | 字符串 → 日期 |

## 十四、执行计划与 SQL 优化

| 命令 | 说明 |
| :--- | :--- |
| `EXPLAIN SELECT …;` | 查看执行计划（`DESC` 与之等价） |
| `EXPLAIN SELECT … \G` | 纵向输出，便于阅读 |

**EXPLAIN 关键列怎么看：**

| 列               | 判断标准                                                             |
| :-------------- | :--------------------------------------------------------------- |
| **`type`**      | **最重要**：`const` ＞ `ref` ＞ `range` ＞ `index` ＞ **`ALL`（全表扫描，最差）** |
| `possible_keys` | 可能用到的索引                                                          |
| **`key`**       | **实际用到的索引，为 `NULL` 说明没走索引**                                      |
| `key_len`       | 用到的索引长度，**越短越好**                                                 |
| `rows`          | 预估扫描行数，**越小越好**                                                  |
| `Extra`         | `Using index`（覆盖索引）✅ ／ `Using filesort` ❌ ／ `Using temporary` ❌  |


**四大索引失效场景：**

| 错误写法 | 结果 | 正确写法 |
| :--- | :---: | :--- |
| `LIKE '%linux'` | ❌ 全表扫描 | `LIKE 'linux%'`（`%` 不能在最前） |
| `WHERE a='x' OR b=15`（b 无索引） | ❌ 索引失效 | 用 `UNION` 拆分 |
| `WHERE 字符串列 = 200` | ❌ **隐式类型转换** | `WHERE 字符串列 = '200'`（**加引号**） |
| `WHERE 列 IS NULL` | ❌ 8.0 下不走索引 | 改为具体值或重建数据 |

> [!warning] 索引列上别做运算
> `WHERE YEAR(日期列)=2026` 会让索引失效，应写成
> `WHERE 日期列 >= '2026-01-01' AND 日期列 < '2027-01-01'`。

## 十五、事务（Transaction）

| 命令 | 说明 |
| :--- | :--- |
| `START TRANSACTION;` | 开启事务（等价 `BEGIN`） |
| `COMMIT;` | **提交**，修改永久生效 |
| `ROLLBACK;` | **回滚**，撤销事务内所有修改 |
| `SAVEPOINT 名;` ／ `ROLLBACK TO 名;` | 设置 / 回滚到**保存点** |
| `SET autocommit = 0;` | 关闭自动提交（**每条 DML 独立提交**的开关） |
| `SET autocommit = 1;` | 恢复自动提交（**MySQL 默认值**） |
| `SELECT @@transaction_isolation;` | 查看隔离级别（8.0 默认 **RR 可重复读**） |
| `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` | 修改隔离级别 |

## 十六、触发器（TRIGGER）

```sql
DELIMITER //                                    -- 改定界符，防止 ; 提前截断
CREATE TRIGGER 触发器名
AFTER INSERT ON 订单表
FOR EACH ROW
BEGIN
    UPDATE 库存表 SET stock = stock - NEW.quantity
    WHERE item_id = NEW.item_id;                -- NEW=新数据，OLD=旧数据
END //
DELIMITER ;                                     -- 恢复标准定界符
```

| 命令 / 关键字 | 说明 |
| :--- | :--- |
| `SHOW TRIGGERS\G` | 查看所有触发器 |
| `DROP TRIGGER 触发器名;` | 删除触发器 |
| **`NEW.列`** | INSERT / UPDATE 后的**新值** |
| **`OLD.列`** | UPDATE / DELETE 前的**旧值** |
| `BEFORE` / `AFTER` | 动作**之前 / 之后**触发 |
| `INSERT` / `UPDATE` / `DELETE` | 三种触发时机 |

## 十七、存储过程（Stored Procedure）

```sql
DELIMITER //
CREATE PROCEDURE 过程名(IN num INT)
BEGIN
    DECLARE i INT DEFAULT 0;                     -- 声明局部变量
    WHILE i < num DO
        INSERT INTO t(username) VALUES (CONCAT('test_', i));
        SET i = i + 1;                           -- 计数器自增
    END WHILE;
END //
DELIMITER ;
```

| 命令 / 关键字 | 说明 |
| :--- | :--- |
| `CALL 过程名(参数);` | **调用**存储过程 |
| `SHOW PROCEDURE STATUS;` | 列出所有存储过程 |
| `SHOW CREATE PROCEDURE 过程名\G` | 查看定义 |
| `DROP PROCEDURE 过程名;` | 删除存储过程 |
| `IN` / `OUT` / `INOUT` | 参数方向：**传入 / 传出 / 传入又传出** |
| `IF … THEN … ELSE … END IF;` | 条件分支 |
| `WHILE … DO … END WHILE;` | 循环 |
| `SELECT … INTO 变量;` | 查询结果赋给变量 |

## 十八、用户 / 权限 / 角色

### 用户管理

| 命令 | 说明 |
| :--- | :--- |
| `CREATE USER '用户'@'主机' IDENTIFIED BY '密码';` | 创建用户，`主机` 限定**来源 IP** |
| `'u'@'localhost'` | 只能本机登录 |
| `'u'@'192.168.0.%'` | 允许某网段远程登录 |
| `'u'@'%'` | **允许任意 IP**（生产环境慎用） |
| `ALTER USER 'u'@'主机' IDENTIFIED BY '新密码';` | **改密码**（8.0 标准写法） |
| `ALTER USER 'u'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;` | 密码 **90 天过期**（等保要求） |
| `ALTER USER 'u'@'%' PASSWORD EXPIRE;` | **下次登录强制改密** |
| `DROP USER 'u'@'主机';` | 删除用户 |
| `SELECT user, host FROM mysql.user;` | 查看所有用户及可登录主机 |

### 权限（GRANT / REVOKE）

| 命令 | 说明 |
| :--- | :--- |
| `GRANT SELECT, INSERT, UPDATE, DELETE ON 库.* TO 'u'@'主机';` | 授予**库级**增删改查 |
| `GRANT SELECT ON 库.表 TO 'u'@'主机';` | 授予**单表只读**（最细粒度） |
| `GRANT ALL PRIVILEGES ON *.* TO 'u'@'主机' WITH GRANT OPTION;` | 全局所有权限（**root 才有**） |
| `REVOKE DELETE ON 库.* FROM 'u'@'主机';` | **回收**权限 |
| `SHOW GRANTS FOR 'u'@'主机';` | 查看某用户**现有权限** |
| `FLUSH PRIVILEGES;` | 重载权限表 |

> 新建用户**默认无任何权限**，连 `SELECT` 都不行，必须显式 `GRANT`。

### 角色（Role，MySQL 8.0）

| 命令                                      | 说明                   |
| :-------------------------------------- | :------------------- |
| `CREATE ROLE '角色1', '角色2';`             | 创建角色（**权限的集合**）      |
| `GRANT SELECT, INSERT ON 库.* TO '角色1';` | 把权限**打包**给角色         |
| `GRANT '角色1' TO 'u1'@'%', 'u2'@'%';`    | 把角色**批量shou'yu多个用户** |
| `SET DEFAULT ROLE ALL TO 'u1'@'%';`     | **手动激活**角色           |
| `SHOW GRANTS FOR 'u1'@'%';`             | 查看用户的角色与权限           |
| `DROP ROLE '角色1';`                      | 删除角色                 |

> [!danger] 角色分配后默认未激活 ⚠️
> 用户登录**依然没有任何权限**。二选一：
> 手动 `SET DEFAULT ROLE ALL TO '用户'@'主机';`，或在配置文件里**一劳永逸**：
> ```ini
> [mysqld]
> activate_all_roles_on_login = ON
> ```
> 改完记得 `systemctl restart mysqld`。

## 十九、状态监控与慢查询

| 命令 | 说明 |
| :--- | :--- |
| `SHOW STATUS LIKE '%Com_select%';` | 查询操作**次数** |
| `SHOW STATUS LIKE '%Com_insert%';` | 插入次数 |
| `SHOW STATUS LIKE '%Com_update%';` | 更新次数 |
| `SHOW STATUS LIKE '%Com_delete%';` | 删除次数 |
| `SHOW STATUS LIKE '%InnoDB_rows%';` | InnoDB **行级**读/写/删统计 |
| `SHOW STATUS LIKE '%Handler_read_key%';` | 走索引读取次数，**越大越好** |
| `SHOW STATUS LIKE '%Handler_read_rnd_next%';` | 全表扫描读取次数，**越小越好** |
| `SHOW VARIABLES LIKE '%quer%';` | 查看慢查询日志**是否开启**及路径 |
| `SHOW STATUS LIKE '%Slow_queries%';` | 慢查询**累计次数** |
| `SHOW PROCESSLIST;` | 当前连接与执行中的语句 |
| `SHOW FULL PROCESSLIST;` | 含完整 SQL 文本 |
| `SHOW VARIABLES LIKE 'max_connections';` | 查看参数当前值 |
| `SET GLOBAL 参数 = 值;` | **运行时改参数**（重启后失效） |

```bash
# 开启慢查询日志（CentOS 8）
vi /etc/my.cnf.d/mysql-server.cnf
[mysqld]
slow_query_log=1
slow_query_log_file=/var/log/mysql/mysql-slow.log
long_query_time=6          # 超过 6 秒记为慢查询
systemctl restart mysqld
# 快速造一条慢查询用于测试
```

```sql
SELECT SLEEP(7);
```

### root 密码彻底遗忘时的极限营救

```bash
systemctl stop mysqld
# 在 /etc/my.cnf.d/mysql-server.cnf 的 [mysqld] 下加一行
skip-grant-tables
systemctl start mysqld
mysql -uroot
```

```sql
FLUSH PRIVILEGES;                                -- 8.0 必须先刷新，否则改不了
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
exit;
```

> [!danger] 千万别忘了删 `skip-grant-tables`
> 改完密码后**必须删掉这一行并重启**，否则数据库处于**无密码裸奔**状态。

## 二十、备份与恢复

### 服务端配置（Binlog）

```ini
# /etc/my.cnf.d/mysql-server.cnf 的 [mysqld] 下
log-bin=mysql-bin              # 开启 Binlog 并指定文件名前缀
binlog-format=ROW              # 推荐 ROW（8.0 默认，数据一致性最高）
server-id=1                    # 集群内必须全局唯一
binlog_expire_logs_seconds=604800   # 保留 7 天
```

**三种格式：**

| 格式 | 说明 |
| :--- | :--- |
| `STATEMENT` | 记 SQL 原文，体积小，但 `LIMIT`/`UUID()` 会导致**主从不一致** |
| **`ROW`** | 记每行改成了什么，**体积大但一致**，是**数据闪回的基础**（推荐） |
| `MIXED` | 系统自动切换，现已少用 |

### mysqldump 逻辑备份

| 命令 | 说明 |
| :--- | :--- |
| `mysqldump -uroot -p'pw' --single-transaction 库名 > x.sql` | 备份**单库**（**必须加 `--single-transaction`**，不锁表） |
| `mysqldump -uroot -p'pw' --single-transaction 库名 表1 表2 > x.sql` | 备份**指定几张表** |
| `mysqldump -uroot -p'pw' --single-transaction --databases 库1 库2 > x.sql` | 备份**多库**（含建库语句） |
| `mysqldump -uroot -p'pw' --all-databases > all.sql` | 备份**全部库** |
| `mysqldump -uroot -p'pw' --single-transaction --routines --triggers 库名 > x.sql` | 连**存储过程 + 触发器**一起导出 |
| `mysql -uroot -p'pw' 库名 < x.sql` | **导入**（恢复） |

### Binlog 增量恢复（时间点闪回）

| 命令 | 说明 |
| :--- | :--- |
| `SHOW BINARY LOGS;` | 列出所有 Binlog 文件 |
| `SHOW MASTER STATUS;` | 查看当前写入的 Binlog 及位置 |
| `mysqlbinlog -vv 文件` | 查看 Binlog 明细 |
| `mysqlbinlog --base64-output=decode-rows -vv 文件` | **以行格式**还原具体改动 |
| `mysqlbinlog --start-datetime='2026-10-10 02:00:00' --stop-datetime='2026-10-10 10:00:00' 文件 > inc.sql` | 按**时间**截取增量 |
| `mysqlbinlog --start-position=137745 --stop-position=138642 文件 > inc.sql` | 按 **Pos 号**像素级截取 |
| `mysql -uroot -p'pw' < inc.sql` | **播放**增量，完成数据拼接 |
| `PURGE BINARY LOGS TO 'mysql-bin.000005';` | 清理 5 之前的旧日志 |

> [!warning] 恢复的正确顺序
> 先导**全量备份** → 再放**增量 Binlog**（`--stop-position` 要卡在**误删语句之前**）。
> 灾难发生后**第一时间切断写入**，否则新产生的 Binlog 会干扰定位。

## 二十一、表分区（Partitioning）

```sql
CREATE TABLE 日志表 (
    log_id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    access_time DATETIME NOT NULL,
    PRIMARY KEY (log_id, access_time)      -- 分区键必须进主键
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY RANGE (YEAR(access_time)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_max VALUES LESS THAN MAXVALUE
);
```

| 命令 | 说明 |
| :--- | :--- |
| `SELECT * FROM 表 PARTITION (分区名);` | **只查某个分区** |
| `SELECT * FROM information_schema.PARTITIONS WHERE TABLE_NAME='表';` | 查看分区详情与行数 |
| `SHOW CREATE TABLE 表\G` | 查看分区定义 |
| `ALTER TABLE 表 ADD PARTITION (PARTITION p名 VALUES LESS THAN (值));` | **新增**分区 |
| `ALTER TABLE 表 DROP PARTITION 分区名;` | **删除分区**，数据**物理秒删**、不可回滚 |
| `ALTER TABLE 表 REORGANIZE PARTITION p_max INTO (…新分区…);` | **拆分 / 扩容**分区 |
| `ALTER TABLE 表 COALESCE PARTITION 2;` | 合并掉最后 2 个分区 |

**分区方式对比：**

| 方式 | 适用场景 |
| :--- | :--- |
| `RANGE` | 按**连续区间**（年份、ID 段）归档，**最常用** |
| `HASH` | 按哈希取模**均匀打散**，缓解单点热点 |
| `LIST` | 按**枚举值**（省份代码 11/31/44）物理隔离 |

## 二十二、SHOW 命令总速查

| 命令 | 说明 |
| :--- | :--- |
| `SHOW DATABASES;` | 所有库 |
| `SHOW TABLES;` | 当前库所有表 |
| `SHOW TABLES FROM 库名;` | 指定库的表 |
| `SHOW COLUMNS FROM 表名;` | 表结构（=`DESC`） |
| `SHOW CREATE TABLE 表名\G` | 建表语句 |
| `SHOW INDEX FROM 表名\G` | 索引信息 |
| `SHOW CREATE VIEW 视图名\G` | 视图定义 |
| `SHOW TRIGGERS\G` | 触发器 |
| `SHOW PROCEDURE STATUS;` / `SHOW FUNCTION STATUS;` | 存储过程 / 函数 |
| `SHOW VARIABLES LIKE '模式';` | **配置参数**（`%` 通配） |
| `SHOW STATUS LIKE '模式';` | **运行状态统计** |
| `SHOW PROCESSLIST;` | 当前连接与执行中的语句 |
| `SHOW ENGINES;` | 支持的存储引擎 |
| `SHOW CHARACTER SET;` / `SHOW COLLATION;` | 字符集与排序规则 |
| `SHOW GRANTS FOR '用户'@'主机';` | 用户权限 |
| `SHOW BINARY LOGS;` | Binlog 文件列表 |
| `SHOW WARNINGS;` | 上一条语句的**警告**（排错必看） |
| `SHOW ERRORS;` | 上一条语句的错误 |
| `SELECT VERSION();` | 查版本 |
| `SELECT DATABASE();` | 查**当前所在库** |
| `SELECT USER();` / `SELECT CURRENT_USER();` | 查当前登录用户 |

## 二十三、CentOS 8 常用文件路径

| 路径 | 内容 |
| :--- | :--- |
| **`/etc/my.cnf.d/mysql-server.cnf`** | **主配置文件**（改完必须重启） |
| **`/var/lib/mysql/`** | **数据目录**（`.ibd`、Binlog 都在这里） |
| `/etc/yum.repos.d/*.repo` | yum / dnf 源配置 |
| `/var/log/mysql/mysql-slow.log` | 慢查询日志 |
| `/var/log/mysqld.log` | 错误日志 |
| `/backup/mysql/` | 备份存放目录（自建） |

## 二十四、15 节核心命令索引

| 节 | 核心命令 |
| :--- | :--- |
| 01 | `dnf install mysql-server -y`、`systemctl start mysqld`、`mysql_secure_installation` |
| 02 | `CREATE DATABASE`、`CREATE TABLE`、`RENAME TABLE`、`CREATE TABLE x LIKE y` |
| 03 | `ALTER TABLE … ADD / MODIFY / CHANGE / DROP` |
| 04 | `INSERT`、`UPDATE`、`DELETE`、`TRUNCATE` |
| 05 | `WHERE`、`BETWEEN`、`IN`、`LIKE`、`REGEXP`、`DISTINCT`、`ORDER BY`、`LIMIT` |
| 06 | `CONCAT`、`SUBSTRING`、`FROM_UNIXTIME`、`UNIX_TIMESTAMP`、`DATEDIFF` |
| 07 | `COUNT`、`SUM`、`AVG`、`MAX`、`MIN`、`GROUP BY`、`HAVING` |
| 08 | `INNER JOIN`、`LEFT JOIN`、`RIGHT JOIN`、`CREATE VIEW` |
| 09 | `CREATE INDEX`、`EXPLAIN`、`SHOW INDEX` |
| 10 | `SHOW STATUS`、`slow_query_log`、`CHECK TABLE`、`OPTIMIZE TABLE` |
| 11 | `START TRANSACTION`、`COMMIT`、`ROLLBACK`、`SET transaction_isolation` |
| 12 | `CREATE TRIGGER`、`CREATE PROCEDURE`、`DELIMITER`、`CALL` |
| 13 | `CREATE USER`、`GRANT`、`REVOKE`、`CREATE ROLE`、`SET DEFAULT ROLE` |
| 14 | `mysqldump`、`mysqlbinlog`、`SHOW BINARY LOGS` |
| 15 | `PARTITION BY RANGE`、`DROP PARTITION`、`REORGANIZE PARTITION` |
