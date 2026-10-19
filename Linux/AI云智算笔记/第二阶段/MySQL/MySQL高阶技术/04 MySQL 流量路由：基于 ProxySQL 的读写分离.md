---
时间: 2026-10-19
课程: MySQL 高阶技术
节次: 第 4 节
环境: VMware + Rocky Linux 8.4 + MySQL 8.0.21 + ProxySQL 2.5 + 私网 192.168.0.0/24
---

在实际企业开发中，代码里通常只写一个数据库连接地址。通过引入 ProxySQL 中间件，让它像“交警”一样，拦截业务代码发来的 SQL 语句：遇到 INSERT/UPDATE/DELETE 就丢给 PRIMARY 节点，遇到 SELECT 就分发给 SECONDARY 节点。

ProxySQL 的默认管理端口是 **6032**（默认账号密码都是 admin），代理业务转发的端口是 **6033**。ProxySQL 原生支持 MySQL MGR 集群，它能通过监控账号自动识别谁是 PRIMARY，谁是 SECONDARY。

## 上半场：环境安装与账号准备

### 步骤 1：安装 ProxySQL（在 node1 上执行）

由于 ProxySQL 不在 Rocky Linux 的默认源中，需要配置官方源（国内访问官方源一般没问题，已关闭 GPG 校验加速下载）。

```bash
# 1. 写入 ProxySQL 源
cat > /etc/yum.repos.d/proxysql.repo << EOF
[proxysql]
name=ProxySQL YUM repository
baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/centos/8
gpgcheck=0
EOF

# 2. 安装 ProxySQL
dnf install proxysql -y
```

也可以直接安装 rpm 包：

```bash
dnf -y install proxysql-2.5.5-1-centos8.x86_64.rpm
```

```bash
# 3. 启动并设置开机自启
systemctl enable --now proxysql

# 4. 检查服务状态
systemctl status proxysql
```

### 步骤 2：创建代理需要的 MySQL 账号（在当前的 PRIMARY 节点上执行）

ProxySQL 需要两个账号：一个用来监控集群谁是主谁是备（`monitor`），另一个给业务代码实际使用（`app_user`）。

请先确认当前集群的 PRIMARY 是哪台（如果做了上节的拔网线测试，PRIMARY 可能是 node2 或 node3；如果没做，就是 node1）。在当前的 PRIMARY 节点上执行以下 SQL：

```bash
mysql -uroot -e "
-- 1. 创建监控账号 (ProxySQL 用它来探测节点存活和角色)
CREATE USER 'monitor'@'%' IDENTIFIED WITH mysql_native_password BY 'Monitor@123';

-- 授予监控最低需要的权限
GRANT SELECT ON performance_schema.* TO 'monitor'@'%';
GRANT SELECT ON sys.* TO 'monitor'@'%';

-- 2. 创建业务账号 (业务代码配置的账号)
CREATE USER 'app_user'@'%' IDENTIFIED WITH mysql_native_password BY 'App@123456';

-- 授予业务表的所有权限
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- 3. 刷新权限
FLUSH PRIVILEGES;
"
```

因为 MGR 的强一致性同步，你在 PRIMARY 上创建的这两个账号，会瞬间自动同步到另外两台 SECONDARY 上，不需要重复执行。

## 下半场：配置读写分离路由规则（以下操作全部在 node1 上执行）

### 步骤 3：登录 ProxySQL 管理后台

在 node1 的终端输入以下命令进入 ProxySQL 的配置界面：

```bash
mysql -uadmin -padmin -h127.0.0.1 -P6032 --prompt='ProxySQL> '
```

### 步骤 4：配置监控账号与 MGR 集群节点

在 `ProxySQL>` 提示符下，依次粘贴并执行以下 SQL：

```sql
-- 1. 配置监控账号密码
UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='Monitor@123' WHERE variable_name='mysql-monitor_password';

-- 2. 告诉 ProxySQL 我们使用的是 MGR 集群，并定义节点组 (Hostgroup) 逻辑：
-- 写组(主):10, 备用写组:11, 读组(备):20, 离线组:30
INSERT INTO mysql_group_replication_hostgroups
(writer_hostgroup, backup_writer_hostgroup, reader_hostgroup, offline_hostgroup, active, max_writers, writer_is_also_reader, max_transactions_behind)
VALUES (10, 11, 20, 30, 1, 1, 0, 100);

-- 3. 将我们 3 台服务器统统加入到 10 组中
-- (不用担心，配置了 MGR 规则后，ProxySQL 会自动把 SECONDARY 节点踢到 20 读组去)
INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections) VALUES (10, '192.168.0.10', 3306, 1000);
INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections) VALUES (10, '192.168.0.20', 3306, 1000);
INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections) VALUES (10, '192.168.0.30', 3306, 1000);

-- 4. 加载到运行环境并保存到磁盘
LOAD MYSQL VARIABLES TO RUNTIME;
-- 把内存(main)中的 mysql_variables 配置加载到运行时(runtime)，立即生效。

SAVE MYSQL VARIABLES TO DISK;
-- 把当前内存(main)中的 mysql_variables 配置写入磁盘配置文件，持久化保存，重启后仍然有效。

LOAD MYSQL SERVERS TO RUNTIME;
-- 把内存(main)中的 mysql_servers 后端 MySQL 服务器列表加载到运行时(runtime)，立即生效。

SAVE MYSQL SERVERS TO DISK;
-- 把当前内存(main)中的后端 MySQL 服务器列表写入磁盘配置文件，持久化保存。

-- 可以简单记成：
-- LOAD ... TO RUNTIME：把内存(main)配置加载到运行时(runtime)，立即生效，但不写盘。
-- SAVE ... TO DISK：把内存(main)配置保存到磁盘，持久化，重启后不丢。
-- MYSQL VARIABLES：ProxySQL 连接后端 MySQL 的相关变量配置。
-- MYSQL SERVERS：ProxySQL 要连接/路由的后端 MySQL 服务器列表。
```

`mysql_group_replication_hostgroups` 各字段含义如下：

| 列名 | 值 | 含义 |
| :--- | :--- | :--- |
| `writer_hostgroup` | 10 | 写节点/主节点放到 Hostgroup 10 |
| `backup_writer_hostgroup` | 11 | 备用写节点放到 Hostgroup 11 |
| `reader_hostgroup` | 20 | 只读/从节点放到 Hostgroup 20 |
| `offline_hostgroup` | 30 | 离线/不可用节点放到 Hostgroup 30 |
| `active` | 1 | 启用这条 MGR 配置 |
| `max_writers` | 1 | 最多允许 1 个写节点，适合 MGR 单主模式 |
| `writer_is_also_reader` | 0 | 写节点不兼职读节点 |
| `max_transactions_behind` | 100 | 从节点落后超过 100 个事务，就认为太旧，不分配读流量或放离线组 |

### 步骤 5：验证 MGR 节点是否被正确识别

依然在 `ProxySQL>` 提示符下，执行以下查询：

```sql
SELECT hostgroup_id, hostname, status FROM runtime_mysql_servers;
```

**期望结果**：你会神奇地发现，你的 3 台机器被自动分家了！PRIMARY 节点保留在 `hostgroup_id = 10`，而另外两台 SECONDARY 节点变成了 `hostgroup_id = 20`，并且状态都是 `ONLINE`。

### 步骤 6：配置业务账号与读写分离路由规则

继续在 `ProxySQL>` 提示符下执行：

```sql
-- 1. 配置业务账号，默认把请求全发给写库 (组10)
INSERT INTO mysql_users(username, password, default_hostgroup) VALUES ('app_user', 'App@123456', 10);

-- 2. 编写正则路由规则 (Query Rules)
-- 规则1：碰到 SELECT ... FOR UPDATE 这种带锁的读，强制走主库 (组10)
INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply) VALUES (1, 1, '^SELECT.*FOR UPDATE', 10, 1);
-- 规则2：普通的 SELECT 查询，全部路由到只读备库 (组20)
INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply) VALUES (2, 1, '^SELECT', 20, 1);

-- 3. 加载并保存生效
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;

-- 4. 退出 ProxySQL 管理台
exit
```

> [!note] SELECT ... FOR UPDATE 的作用
> 在事务里查数据，并给这些行加排他锁，别的事务不能改、也不能再 `FOR UPDATE` 同一行。典型场景就是：库存扣减、账户转账、抢券、订单状态流转。

#### 准备数据表

以业务账号连接代理端口：

```bash
# 操作业务
mysql -uapp_user -pApp@123456 -h127.0.0.1 -P6033
```

```sql
-- 创建测试表
CREATE TABLE goods (
    id INT PRIMARY KEY,  -- 主键，行锁生效关键
    stock INT NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 初始化数据
INSERT INTO goods(id, stock) VALUES (1, 1);
```

举个例子，库存只有 1 件。

**不用 FOR UPDATE：**

```sql
-- 事务 A
START TRANSACTION;
SELECT stock FROM goods WHERE id = 1;  -- 读到 1，觉得有货
UPDATE goods SET stock = stock - 1 WHERE id = 1;  -- 变成 0
sleep(10);
COMMIT;  -- 释放锁

-- 事务 B
START TRANSACTION;
SELECT stock FROM goods WHERE id = 1;  -- 也读到 1，也觉得有货
UPDATE goods SET stock = stock - 1 WHERE id = 1;  -- 变成 -1，超卖！
COMMIT;
```

两个事务都以为有货，结果库存变成负数。

**用 FOR UPDATE：**

```sql
-- 事务 A
START TRANSACTION;
SELECT stock FROM goods WHERE id = 1 FOR UPDATE;  -- 锁住这一行，读到 1
UPDATE goods SET stock = stock - 1 WHERE id = 1;  -- 变成 0
sleep(10);
COMMIT;  -- 释放锁

-- 事务 B
START TRANSACTION;
SELECT stock FROM goods WHERE id = 1 FOR UPDATE;  -- 卡住，等 A 提交
-- 事务A没提交时，这一行会阻塞等待；A提交后，这里查到 stock=0
-- A 提交后，B 才读到 0，发现没货，不买了
COMMIT;
```

所以它的核心作用是：把并发操作“排队”，让同一行数据在同一时间只能被一个事务锁定修改。它相当于一种悲观锁：我担心别人会改，所以我先锁住，别人只能等我。

### 步骤 7：终极效果验证（见证读写分离）

现在模拟业务代码，使用 `app_user` 连接 ProxySQL 的业务端口 6033。在 node1 的系统终端里，连续多次执行下面这行测试命令（查询当前的 `server_id`）：

```bash
mysql -uapp_user -p'App@123456' -h127.0.0.1 -P6033 -e "SELECT @@server_id;"
```

激动人心的时刻到了：

- 因为你发送的是 SELECT 语句，ProxySQL 会拦截它并转发给 20 组（读库）
- 你会看到返回的 `server_id` 在 2 和 3 之间交替出现（代表 node2 和 node3 正在轮询为你提供查询服务，成功分摊了读压力）
- 如果你执行的是 INSERT，请求就会百分之百打在写库（主库）上

## 拓展实验：验证写请求 100% 打到主库

说明：如果第 3 节做了“拔网线”容灾测试，MGR 集群会把 PRIMARY 自动切换到 node2（`server_id=2`），node1 和 node3 变为只读备库，ProxySQL 会自动感知这个变化，把写请求的组（10 组）指向 node2。以下两个实验即基于该环境（PRIMARY=node2）验证写请求的走向。

由于单纯执行 INSERT 语句屏幕只会返回 `Query OK, 1 row affected`，并不会直接打印出在哪台机器上执行的，因此用以下两个实验来直观验证。

### 实验 1：直观验证法（利用排他锁强制走写库）

在第 4 节配置 ProxySQL 规则时，我们写了一条专门的规则：遇到带锁的 `SELECT ... FOR UPDATE`，代表马上要修改数据，必须路由给主库。请在 node1 的终端连续多次执行以下两条命令进行对比：

```bash
# 1. 普通读测试（走组20）：你会看到结果在 1 和 3 之间交替（由 node1 和 node3 轮询处理）
mysql -uapp_user -p'App@123456' -h127.0.0.1 -P6033 -e "SELECT @@server_id;"

# 2. 写意图测试（走组10）：强制走写库，你会发现无论执行多少次，结果 100% 都是 2（由 node2 独占处理）
mysql -uapp_user -p'App@123456' -h127.0.0.1 -P6033 -e "SELECT @@server_id FOR UPDATE;"
```

### 实验 2：硬核审计法（通过 ProxySQL 内部统计证实）

我们直接发送真实的 CREATE 和 INSERT 语句，然后通过 ProxySQL 的“审计后台”来抓包，看看它到底把 SQL 发给了哪个组。

#### 1. 通过代理端口 (6033) 真实写入数据

```bash
mysql -uapp_user -p'App@123456' -h127.0.0.1 -P6033 -e "
CREATE DATABASE IF NOT EXISTS rw_test;
USE rw_test;
CREATE TABLE IF NOT EXISTS demo (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(20));
INSERT INTO demo (name) VALUES ('ProxySQL_Write_Test');
"
```

#### 2. 登录 ProxySQL 管理后台 (6032) 查看流量路由审计表

```bash
mysql -uadmin -padmin -h127.0.0.1 -P6032 -e "
SELECT hostgroup, count_star, digest_text
FROM stats_mysql_query_digest
WHERE digest_text LIKE '%INSERT%' OR digest_text LIKE '%CREATE%' or digest_text like '%select%';
"
```

**见证奇迹**：你会清晰地在输出结果中看到，CREATE 和 INSERT 语句对应的 `hostgroup` 字段全部都是 10（写库组，即 node2）。而之前你测试的 SELECT 语句对应的都会是 20 组。

### 实验 3：monitor 用户检测

如果你想验证 `monitor` 账号是否正常工作，可以在 ProxySQL admin 端口执行：

```bash
# 登录ProxySQL
mysql -uadmin -padmin -h127.0.0.1 -P6032

# 查看ProxySQL通过monitor用户对MGR的监控
# SELECT * FROM mysql_server_connect_log;
# 这个表记录了 ProxySQL 使用 monitor 账号尝试连接各个后端 MySQL 节点的日志，可以看到探测记录，成功 / 失败。
```

```sql
mysql> SELECT * FROM mysql_server_connect_log;
+--------------+------+------------------+-------------------------+---------------+
| hostname     | port | time_start_us    | connect_success_time_us | connect_error |
+--------------+------+------------------+-------------------------+---------------+
| 192.168.0.10 | 3306 | 1792343826860797 | 403                     | NULL          |
| 192.168.0.20 | 3306 | 1792343827518270 | 2382                    | NULL          |
| 192.168.0.30 | 3306 | 1792343828188122 | 2245                    | NULL          |
| 192.168.0.10 | 3306 | 1792343886874427 | 938                     | NULL          |
| 192.168.0.30 | 3306 | 1792343887535214 | 1247                    | NULL          |
| 192.168.0.20 | 3306 | 1792343888196213 | 2227                    | NULL          |
| 192.168.0.10 | 3306 | 1792343946879069 | 1042                    | NULL          |
| 192.168.0.20 | 3306 | 1792343947416272 | 2229                    | NULL          |
| 192.168.0.30 | 3306 | 1792343947949223 | 1642                    | NULL          |
| 192.168.0.10 | 3306 | 1792344006891957 | 447                     | NULL          |
| 192.168.0.30 | 3306 | 1792344007535111 | 1360                    | NULL          |
| 192.168.0.20 | 3306 | 1792344008167514 | 715                     | NULL          |
| 192.168.0.20 | 3306 | 1792344066904679 | 1058                    | NULL          |
| 192.168.0.30 | 3306 | 1792344067596131 | 1579                    | NULL          |
| 192.168.0.10 | 3306 | 1792344068281185 | 1533                    | NULL          |
| 192.168.0.30 | 3306 | 1792344126922265 | 4043                    | NULL          |
| 192.168.0.10 | 3306 | 1792344127657547 | 517                     | NULL          |
| 192.168.0.20 | 3306 | 1792344128400388 | 1426                    | NULL          |
| 192.168.0.10 | 3306 | 1792344186934947 | 1163                    | NULL          |
| 192.168.0.30 | 3306 | 1792344187477555 | 3122                    | NULL          |
| 192.168.0.20 | 3306 | 1792344188023749 | 2635                    | NULL          |
| 192.168.0.10 | 3306 | 1792344246946915 | 417                     | NULL          |
| 192.168.0.30 | 3306 | 1792344247663125 | 1990                    | NULL          |
| 192.168.0.20 | 3306 | 1792344248375983 | 877                     | NULL          |
| 192.168.0.30 | 3306 | 1792344306952001 | 1487                    | NULL          |
| 192.168.0.10 | 3306 | 1792344307434344 | 1179                    | NULL          |
| 192.168.0.20 | 3306 | 1792344307915039 | 2228                    | NULL          |
| 192.168.0.10 | 3306 | 1792344366958135 | 688                     | NULL          |
| 192.168.0.20 | 3306 | 1792344367407304 | 831                     | NULL          |
| 192.168.0.30 | 3306 | 1792344367853817 | 648                     | NULL          |
+--------------+------+------------------+-------------------------+---------------+
30 rows in set (0.00 sec)
```

**结果解读**：`monitor` 账号完全正常工作！

`connect_error` 全部是 `NULL`，代表 ProxySQL 使用 monitor 账号成功连接三台 MGR 节点（192.168.0.10 / 0.20 / 0.30）。

字段含义：

- `hostname` + `port`：后端 MySQL 节点地址
- `time_start_us`：发起探测连接的时间戳
- `connect_success_time_us`：建立连接耗时（微秒，几百～几千 us，正常）
- `connect_error`：`NULL` = 连接无错误。如果这里出现文字报错，代表 monitor 账号密码 / 权限不对或者网络不通。

## 结语

主从复制保障备份，双主保障架构扩展，MGR 保障高可用与数据强一致，ProxySQL 保障业务层的智能分流。这套完整的架构完全可以支撑起真实生产环境中百万级以上的高并发业务。
