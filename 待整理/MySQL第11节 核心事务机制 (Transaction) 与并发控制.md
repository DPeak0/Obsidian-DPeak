---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 11 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：深入理解数据库事务的 **ACID 核心特性**，熟练掌握通过 SQL 语句手动控制事务的**开启、提交与回滚**。掌握并发场景下的数据安全隐患（**脏读、不可重复读、幻读**），并透彻理解 MySQL 8.0 的**四大隔离级别**及底层应对策略。

## 一、事务的核心理念与 ACID 特性

在真实的业务场景（如**电商扣减库存、银行转账**）中，一个完整的业务动作往往由**多条 SQL 语句**组成。**事务（Transaction）就是将这多条 SQL 捆绑成一个不可分割的执行单元。**

为了保证数据的绝对安全，**InnoDB 引擎的事务严格遵循 ACID 四大特性**：

| 特性 | 中文 | 说明 |
| :--- | :--- | :--- |
| **A**tomicity | **原子性** | 事务是一个**原子操作单元**，其对数据的修改**要么全部执行，要么全都不执行**。决不允许出现"钱扣了，但对方没收到"的中间状态 |
| **C**onsistency | **一致性** | 事务在完成时，必须使**所有的数据都保持一致状态**。如转账前后，两个账户的**总金额必须保持不变** |
| **I**solation | **隔离性** | **并发执行的事务之间相互独立**，一个事务内部的操作及使用的数据对并发的其他事务是隔离的 |
| **D**urability | **持久性** | 事务**一旦提交（Commit），它对数据库中数据的改变就是永久性的**，即便随后服务器发生宕机重启，数据也不会丢失 |

**ACID 的底层实现机制（记住，面试常问）**

| 特性 | 靠什么实现 |
| :--- | :--- |
| **原子性** | **undo log（回滚日志）** |
| **一致性** | **由其他三个特性 + 业务约束共同保证** |
| **隔离性** | **锁 + MVCC** |
| **持久性** | **redo log（重做日志）+ WAL 机制** |

## 二、事务控制语言 (TCL) 实战演练

MySQL 8.0 默认开启了**自动提交（`autocommit = 1`）**，即**每一条单行的 DML 语句都会被当做一个独立的事务自动执行并提交**。在复杂业务中，我们需要**手动接管事务的生命周期**。

### 1. 准备转账测试数据

```sql
CREATE TABLE account(
    id int unsigned auto_increment primary key,
    name varchar(30) not null,
    balance decimal(10,2) not null
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO account(name, balance) VALUES ('Alice', 5000), ('Bob', 0);
```

**金额为什么用 `decimal` 而不是 `float`**：`float`/`double` 是**浮点数，存在精度误差**（`0.1 + 0.2 != 0.3`）。
**涉及金额必须用 `DECIMAL(M,D)`** —— 定点数，精确计算。

### 2. 成功的事务流转 (Commit)

模拟 Alice 向 Bob 转账 1000 元的正常流程。

```sql
-- 1. 显式开启事务 (或者使用 BEGIN)
START TRANSACTION;

-- 2. 执行业务 SQL 组
UPDATE account SET balance = balance - 1000 WHERE name = 'Alice';
UPDATE account SET balance = balance + 1000 WHERE name = 'Bob';

-- 3. 确认无误，手动提交事务，将数据永久写入磁盘
COMMIT;
```

### 3. 失败的事务挽救 (Rollback)

```sql
START TRANSACTION;
UPDATE account SET balance = balance - 1000 WHERE name = 'Alice';
-- 假设此时服务器突然断电，或者应用程序抛出异常，触发回滚指令
-- 撤销自 START TRANSACTION 之后的所有修改，恢复到事务开启前的状态
ROLLBACK;
```

### 事务控制语句速查

| 语句 | 作用 |
| :--- | :--- |
| **`START TRANSACTION`** / **`BEGIN`** | **开启事务** |
| **`COMMIT`** | **提交事务**（永久生效） |
| **`ROLLBACK`** | **回滚事务**（撤销所有修改） |
| `SAVEPOINT name` | 设置保存点 |
| `ROLLBACK TO name` | 回滚到保存点（**不结束整个事务**） |
| `SET autocommit = 0` | 关闭自动提交（**之后每条语句都要手动 COMMIT**） |

```sql
-- SAVEPOINT 示例：部分回滚
START TRANSACTION;
INSERT INTO user(username) VALUES ('u1');
SAVEPOINT sp1;
INSERT INTO user(username) VALUES ('u2');
ROLLBACK TO sp1;     -- 只撤销 u2 的插入，u1 保留
COMMIT;
```

## 三、并发问题与隔离级别

### 1. 三大并发读问题

| 问题 | 现象 | 本质 |
| :--- | :--- | :--- |
| **脏读 (Dirty Read)** | 读到**别的事务还没提交**的数据 | 读到了**可能被回滚**的数据 |
| **不可重复读 (Non-repeatable Read)** | **同一事务内两次读同一行，结果不同** | 别的事务在这中间**更新并提交**了 |
| **幻读 (Phantom Read)** | **同一事务内两次查同一范围，行数变多/变少** | 别的事务在这中间**插入/删除并提交**了 |

> [!important] 不可重复读 vs 幻读 的区别（记住）
> - **不可重复读**：针对**同一行的修改**（UPDATE）
> - **幻读**：针对**范围内的行的增删**（INSERT / DELETE）

### 2. 四大隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
| :--- | :---: | :---: | :---: | :--- |
| **READ UNCOMMITTED**（读未提交） | ❌ | ❌ | ❌ | 性能最高，问题最多，**几乎不用** |
| **READ COMMITTED**（读已提交） | ✅ | ❌ | ❌ | **Oracle/PostgreSQL 默认** |
| **REPEATABLE READ**（可重复读） | ✅ | ✅ | ✅* | **MySQL 默认** |
| **SERIALIZABLE**（串行化） | ✅ | ✅ | ✅ | 全部加锁，**性能最低** |

> [!important] MySQL 的默认隔离级别是 RR（记住）
> ```sql
> -- 查看当前隔离级别
> SELECT @@transaction_isolation;
> -- 结果：REPEATABLE-READ
>
> -- 修改（会话级）
> SET SESSION transaction_isolation = 'READ-COMMITTED';
> -- 修改（全局，需重启连接生效）
> SET GLOBAL transaction_isolation = 'READ-COMMITTED';
> ```

***MySQL 的 RR 通过 **Next-Key Lock（间隙锁）** 在大多数场景下**也解决了幻读****：这是 MySQL 与其他数据库的**重要差异**：
- 标准定义的 RR **允许**幻读
- **MySQL 的 RR 用间隙锁额外防住了幻读**

所以实际上 **MySQL 的 RR = 可重复读 + 无幻读**。

### 3. MVCC 快照读

**MySQL 能在 RR 级别避免不可重复读，靠的是 **MVCC（多版本并发控制）****

- 每行数据有隐藏的**版本号/事务ID**
- 读操作**读的是快照**（事务开始时的一致性视图），**不加锁**
- 所以 **"读"和"写"不互相阻塞** —— 这是 MySQL 高性能的关键

---
