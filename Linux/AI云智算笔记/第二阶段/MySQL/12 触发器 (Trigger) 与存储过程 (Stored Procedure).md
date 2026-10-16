---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 12 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：打破以往只在业务代码（如 Java/Python）中处理逻辑的思维局限。掌握**触发器**，实现表级数据的自动化联动防漏；精通**存储过程**，将复杂的批量操作和统计逻辑直接封装在数据库底层执行，**大幅度降低网络传输开销**，打造真正贴合企业级高并发场景的强力后盾。

## 一、触发器 (Trigger)：数据的自动化守卫

触发器是一种**特殊的存储过程，它不需要手动调用**。当在一张表上发生**特定的 DML 操作（INSERT, UPDATE, DELETE）**时，触发器会**自动"扣动扳机"** 执行预设的 SQL 语句。

### 1. 核心原理与关键字

触发器执行逻辑依赖**两个极为重要的内部对象**：

| 关键字 | 含义 | 常用于 |
| :--- | :--- | :--- |
| **`NEW`** | 代表引发触发器执行的**新录入数据** | **INSERT 和 UPDATE** |
| **`OLD`** | 代表触发器执行前**原有的老数据** | **UPDATE 和 DELETE** |

**NEW / OLD 可用性矩阵（记住）**

| 触发事件 | NEW | OLD |
| :--- | :---: | :---: |
| **INSERT** | ✅ | ❌ |
| **UPDATE** | ✅ | ✅ |
| **DELETE** | ❌ | ✅ |

### 2. 实战演练：订单插入联动扣减库存

在**电商秒杀场景**中，如果在应用程序里先写订单再更新库存，**一旦网络抖动会导致两步操作脱节**。使用触发器可以保证这两步在**数据库底层瞬间连贯完成**。

#### 准备测试表

```sql
-- 创建库存表
CREATE TABLE inventory (
    item_id INT UNSIGNED PRIMARY KEY,
    item_name VARCHAR(50),
    stock INT UNSIGNED NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 初始化一件商品，库存 100 件
INSERT INTO inventory(item_id, item_name, stock) VALUES (1, 'MySQL核心技术指南', 100);

-- 创建订单记录表
CREATE TABLE orders (
    order_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    item_id INT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL,
    order_time DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 创建核心触发器

> [!important] `DELIMITER` 的作用（关键，记住）
> 在 MySQL 命令行中编写**包含多条语句的代码块**时，**必须先使用 `DELIMITER` 将结束符临时从 `;` 改为其他符号（如 `//`）**，防止 MySQL 提前阻断语句编译。

```sql
-- 更改定界符
DELIMITER //

-- 创建触发器：在 orders 表完成插入操作之后自动触发
CREATE TRIGGER tr_after_insert_order
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- 核心逻辑：利用 NEW 关键字获取刚刚插入的商品ID和购买数量，去扣减库存表
    UPDATE inventory
    SET stock = stock - NEW.quantity
    WHERE item_id = NEW.item_id;
END //

-- 恢复标准定界符
DELIMITER ;
```

#### 触发测试

```sql
-- 我们只执行一条插入订单的 SQL（买走 3 本书）
INSERT INTO orders (item_id, quantity) VALUES (1, 3);

-- 见证奇迹：核查库存表，发现 stock 自动变为了 97！
SELECT * FROM inventory;
```

### 触发器的四要素

| 要素 | 选项 |
| :--- | :--- |
| **时机 (Timing)** | **`BEFORE`** / **`AFTER`** |
| **事件 (Event)** | `INSERT` / `UPDATE` / `DELETE` |
| **表** | `ON 表名` |
| **频率** | `FOR EACH ROW`（**行级触发**，MySQL 只支持这个） |

> [!warning] 触发器的三个限制与风险
> 1. **不能对同一张表做增删改**（否则无限递归）
> 2. **`NEW` 在 BEFORE INSERT 里可修改**（用于校验/补值），**AFTER 里不可修改**
> 3. **触发器里的错误会导致整个业务语句回滚**，且**报错信息难排查**
>
> ⚠️ **生产环境建议**：触发器**把逻辑隐藏在数据库层，业务代码看不见**，维护困难。**优先在应用层实现**，触发器只用于**审计日志、强制约束**等少数场景。

```sql
-- 查看已有触发器
SHOW TRIGGERS\G
-- 删除
DROP TRIGGER tr_after_insert_order;
```

## 二、存储过程 (Stored Procedure)：复杂逻辑的终极封装

存储过程是**预先编译并存储在数据库中的一组 SQL 语句集合**。它不仅包含标准的增删改查，还能引入 **`IF` 条件判断、`WHILE` 循环控制**等编程特性。

### 核心优势

> [!important] 两大优势（记住）
> **1. 减少网络通信开销**：只需发送一条 `CALL` 调用指令，**成百上千条复杂的运算在数据库服务器本地瞬间跑完**。
> **2. 提升执行性能**：存储过程是**预编译**的，执行速度快于动态发送的单条 SQL。

### 实战演练：封装批量造数脚本

在云计算与数据库综合实训中，学员们经常需要**上万条测试数据**来验证负载均衡与数据库性能监控指标。如果**在应用层写个 for 循环一条条 insert 进数据库，网络延迟会让人抓狂**。用存储过程在底层生成，才是教科书级别的操作。

#### 编写带 WHILE 循环的存储过程

```sql
DELIMITER //

-- 创建名为 generate_users 的存储过程，接收一个参数 num 代表生成数量
CREATE PROCEDURE generate_users(IN num INT)
BEGIN
    -- 声明一个局部变量作为计数器
    DECLARE i INT DEFAULT 0;

    -- 开启循环
    WHILE i < num DO
        -- 构造带动态序号的测试数据，并利用 RAND() 生成随机年龄与班级
        INSERT INTO user (username, age, class_id)
        VALUES (CONCAT('test_user_', i), FLOOR(RAND() * 20 + 18), FLOOR(RAND() * 3 + 1));

        -- 计数器递增
        SET i = i + 1;
    END WHILE;
END //

DELIMITER ;
```

#### 一键呼叫执行

```sql
-- 瞬间生成 5000 条测试数据，网络开销几乎为 0
CALL generate_users(5000);

-- 查看数据是否成功生成
SELECT COUNT(*) FROM user;
```

### 存储过程的参数类型

| 类型 | 说明 |
| :--- | :--- |
| **`IN`** | **输入参数**（默认），调用时传值进去 |
| **`OUT`** | **输出参数**，过程执行后把值带出来 |
| `INOUT` | 既是输入也是输出 |

```sql
-- OUT 参数示例
DELIMITER //
CREATE PROCEDURE count_users(OUT total INT)
BEGIN
    SELECT COUNT(*) INTO total FROM user;
END //
DELIMITER ;

CALL count_users(@n);
SELECT @n;
```

### 存储过程控制结构

```sql
-- IF 判断
IF 条件 THEN
    语句;
ELSEIF 条件 THEN
    语句;
ELSE
    语句;
END IF;

-- WHILE 循环
WHILE 条件 DO
    语句;
END WHILE;

-- LOOP 循环
label: LOOP
    语句;
    LEAVE label;      -- 相当于 break
    ITERATE label;    -- 相当于 continue
END LOOP;
```

### 存储过程管理

```sql
-- 查看所有存储过程
SHOW PROCEDURE STATUS WHERE Db='mydb'\G

-- 查看定义
SHOW CREATE PROCEDURE generate_users\G

-- 删除
DROP PROCEDURE generate_users;
```

> [!warning] 存储过程在互联网公司的现状
> | 优点 | 缺点 |
> | :--- | :--- |
> | 减少网络往返 | **调试困难**（没有好的 IDE 断点） |
> | 预编译，快 | **版本管理困难**（不在代码仓库里） |
> | 权限可控 | **数据库压力集中**（CPU 全在 DB 上） |
> | 逻辑集中 | **难迁移**（换数据库要重写） |
>
> **现代趋势**：**逻辑尽量放应用层**，存储过程只用于**批量数据处理、定时统计任务、报表生成**。

---
