---
时间: 2026-10-19
课程: MySQL 高阶技术
节次: 00 命令速查
环境: VMware + Rocky Linux 8.4 + MySQL 8.0.21 + ProxySQL 2.5 + 私网 192.168.0.0/24
---

> 本文件只放命令和实验流程，原理与步骤说明见 01～04 各节笔记。

## 一、实验环境与节点规划

### 1. 系统环境

| 项目 | 配置 |
| :--- | :--- |
| 操作系统 | Rocky Linux 8.4 64 位（2 核 3G） |
| 操作账号 | 全程 root，无需 sudo |
| 安全组件 | iptables、SELinux、firewall 均已关闭 |
| MySQL | Rocky 8.4 光盘源自带 MySQL 8.0.21 |

### 2. 服务器清单

| 服务器 | 公网 IP | 私网 IP | 角色 |
| :--- | :--- | :--- | :--- |
| node1 | nat8: 192.168.100.10 | 192.168.0.10 | 主库 / 主 1 / MGR 引导节点 / ProxySQL 部署点 |
| node2 | 无 | 192.168.0.20 | 从库 / 主 2 / MGR 备节点 |
| node3 | 无 | 192.168.0.30 | MGR 第三节点（只读备） |

### 3. 主机名设置

```bash
# 三台分别执行
hostnamectl set-hostname node1
hostnamectl set-hostname node2
hostnamectl set-hostname node3
```

### 4. 实验大纲

| 节 | 实验主题 | 服务器 | 关键点 |
| :--- | :--- | :--- | :--- |
| 01 | 主从模式搭建与测试 | node1（主）、node2（从） | Binlog 复制、账号授权、同步状态排查 |
| 02 | 双主模式配置与防冲突 | node1、node2 | 双向复制、`auto_increment` 步长防主键冲突 |
| 03 | MGR（组复制）集群搭建 | node1、node2、node3 | 至少 3 节点、单主模式、自动容灾 |
| 04 | 基于 ProxySQL 的读写分离 | node1 + MGR 节点 | 写请求走主、读请求分发到备 |

## 二、安装与服务管理

```bash
# 安装 MySQL（node1 / node2 / node3）
dnf install mysql-server -y
systemctl enable --now mysqld
systemctl status mysqld

# 修改配置后重启
systemctl restart mysqld

# 查看 3306 端口（任选一条）
netstat -tunpl | grep 3306
ss -tunpl | grep 3306
lsof -i:3306 -P
```

```bash
# root 改密（首次取临时密码）
grep 'temporary password' /var/log/mysqld.log
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';

# 写入 /root/.my.cnf 后可免密执行 mysql -uroot
```

### 端口一览

| 端口 | 用途 |
| :--- | :--- |
| 3306 | MySQL 业务端口（也是 MGR 节点恢复数据用的端口） |
| 33061 | MGR 组内通讯端口 |
| 6032 | ProxySQL 管理端口（admin/admin） |
| 6033 | ProxySQL 业务代理端口 |

## 三、配置文件速查

所有配置写在 `/etc/my.cnf.d/` 下，改完必须 `systemctl restart mysqld`。

### 1. 主库（node1，第 1 节）

```bash
cat > /etc/my.cnf.d/replication.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
EOF
```

### 2. 从库（node2，第 1 节）

```bash
cat > /etc/my.cnf.d/replication.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 2
relay-log = mysql-relay-bin
read_only = 1
EOF
```

### 3. 双主（node1 追加 / node2 覆盖）

```bash
# node1：在原配置末尾追加
cat >> /etc/my.cnf.d/replication.cnf << EOF
auto_increment_increment = 2
auto_increment_offset = 1
EOF

# node2：覆盖写入完整配置（去掉 read_only，开启 log-bin）
cat > /etc/my.cnf.d/replication.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 2
relay-log = mysql-relay-bin
log-bin = mysql-bin
binlog_format = ROW
auto_increment_increment = 2
auto_increment_offset = 2
EOF
```

| 节点 | 步长 | 偏移 | 生成的 ID |
| :--- | :--- | :--- | :--- |
| node1 | 2 | 1 | 1、3、5… |
| node2 | 2 | 2 | 2、4、6… |

### 4. MGR（三台只有 server-id 和 local_address 不同）

```bash
cat > /etc/my.cnf.d/mgr.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
binlog_checksum = NONE
plugin_load_add = group_replication.so
group_replication_group_name = '11111111-2222-3333-4444-555555555555'
group_replication_start_on_boot = OFF
group_replication_local_address = '192.168.0.10:33061'
group_replication_group_seeds = '192.168.0.10:33061,192.168.0.20:33061,192.168.0.30:33061'
group_replication_single_primary_mode = ON
transaction_write_set_extraction = XXHASH64
EOF
```

> `server-id` 依次为 1 / 2 / 3，`group_replication_local_address` 换成本机私网 IP，`group_replication_group_seeds` 三台写一样。

### 5. 四类架构参数对比

| 参数 | 主从 Master | 主从 Slave | 双主 | MGR |
| :--- | :--- | :--- | :--- | :--- |
| `server-id` | 1 | 2 | 1 / 2 | 1 / 2 / 3 |
| `log-bin` | ✅ | ❌ | ✅ | ✅ |
| `relay-log` | ❌ | ✅ | ✅ | ❌ |
| `read_only` | ❌ | 1 | ❌ | ❌ |
| `gtid_mode` | ❌ | ❌ | ❌ | ON |
| `auto_increment_*` | ❌ | ❌ | ✅ | ❌ |
| 组复制插件 | ❌ | ❌ | ❌ | `group_replication.so` |

## 四、主从复制命令流程（第 1 节）

### 1. 主库：建同步账号

```sql
CREATE USER 'repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY 'Repl@123456';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.0.%';
FLUSH PRIVILEGES;
```

### 2. 主库：记录 Binlog 坐标

```bash
mysql -uroot -e "SHOW MASTER STATUS\G"
```

记下 `File`（如 `mysql-bin.000001`）和 `Position`（如 `751`）。

### 3. 从库：建立链路

```sql
CHANGE MASTER TO
  MASTER_HOST = '192.168.0.10',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'Repl@123456',
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 751;

START SLAVE;
SHOW SLAVE STATUS\G
```

判断成功的两个字段：`Slave_IO_Running: Yes`、`Slave_SQL_Running: Yes`。

### 4. 验证

```bash
# 主库写入
mysql -uroot -e "
CREATE DATABASE rep_test;
USE rep_test;
CREATE TABLE users (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Rocky'), ('HuaweiCloud');
"

# 从库查询
mysql -uroot -e "USE rep_test; SELECT * FROM users;"
```

> MySQL 8.0.21 只能用 `MASTER/SLAVE` 语法；`SOURCE/REPLICA` 是 8.0.22/8.0.23 之后才有的。

## 五、双主命令流程（第 2 节）

```bash
# 1. node2 补齐同步账号（第 1 节建号操作没有同步过来）
mysql -uroot -e "
CREATE USER IF NOT EXISTS 'repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY 'Repl@123456';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.0.%';
FLUSH PRIVILEGES;
"

# 2. node2 查看自己的 Binlog 坐标
mysql -uroot -e "SHOW MASTER STATUS\G"

# 3. node1 反向指向 node2
mysql -uroot -e "
CHANGE MASTER TO
  MASTER_HOST = '192.168.0.20',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'Repl@123456',
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 157;
START SLAVE;
SHOW SLAVE STATUS\G
"
```

### 验证奇偶 ID 与互相同步

```bash
# node1 写入 → 奇数 ID
mysql -uroot -e "
USE rep_test;
INSERT INTO users (name) VALUES ('Node1_Insert_A'), ('Node1_Insert_B');
SELECT * FROM users;
"

# node2 写入 → 偶数 ID，并能看到 node1 的数据
mysql -uroot -e "
USE rep_test;
INSERT INTO users (name) VALUES ('Node2_Insert_A'), ('Node2_Insert_B');
SELECT * FROM users ORDER BY id ASC;
"

# node1 再查一次，能看到 node2 写的偶数 ID
mysql -uroot -e "USE rep_test; SELECT * FROM users ORDER BY id ASC;"
```

## 六、MGR 命令流程（第 3 节）

### 1. 清理旧复制架构（node1、node2）

```bash
mysql -uroot -e "
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;
"
rm -f /etc/my.cnf.d/replication.cnf
```

### 2. 三台创建 MGR 同步账号

```bash
mysql -uroot -e "
SET SQL_LOG_BIN=0;
CREATE USER IF NOT EXISTS 'mgr_repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY 'Mgr@123456';
GRANT REPLICATION SLAVE ON *.* TO 'mgr_repl'@'192.168.0.%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='mgr_repl', MASTER_PASSWORD='Mgr@123456' FOR CHANNEL 'group_replication_recovery';
"
```

> 只授予 `REPLICATION SLAVE`，不要再加 `CONNECTION ADMIN`、`GROUP_REPLICATION_STREAM` 等动态权限，否则报 `Unknown dynamic privilege`。

### 3. 三台配置主机名解析

```bash
cat >> /etc/hosts << EOF
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3
EOF
```

### 4. 引导主节点（node1）

```bash
mysql -uroot -e "
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
"
```

> 最后一句必须执行，否则后续网络波动可能引发脑裂。

### 5. 其余节点加入（node2、node3）

```bash
mysql -uroot -e "START GROUP_REPLICATION;"
```

### 6. 验证集群

```bash
mysql -uroot -e "
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
"
```

期望：3 行全为 `ONLINE`，1 个 `PRIMARY`、2 个 `SECONDARY`。

### 7. 故障转移与恢复

```bash
# node1 模拟宕机
systemctl stop mysqld

# node2 查看新主
mysql -uroot -e "SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE FROM performance_schema.replication_group_members;"

# node1 恢复后以 SECONDARY 重新加入
systemctl start mysqld
mysql -uroot -e "START GROUP_REPLICATION;"
```

### 8. 卡在 RECOVERING 时的修复

```bash
# 三台都写 hosts
cat >> /etc/hosts << EOF
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3
EOF

# node2、node3 重启组复制，等 5～10 秒再查
mysql -uroot -e "
STOP GROUP_REPLICATION;
START GROUP_REPLICATION;
"
```

## 七、ProxySQL 命令速查（第 4 节）

### 1. 安装与启动（node1）

```bash
cat > /etc/yum.repos.d/proxysql.repo << EOF
[proxysql]
name=ProxySQL YUM repository
baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/centos/8
gpgcheck=0
EOF

dnf install proxysql -y
# 或：dnf -y install proxysql-2.5.5-1-centos8.x86_64.rpm

systemctl enable --now proxysql
systemctl status proxysql
```

### 2. 在 PRIMARY 上建代理账号

```sql
CREATE USER 'monitor'@'%' IDENTIFIED WITH mysql_native_password BY 'Monitor@123';
GRANT SELECT ON performance_schema.* TO 'monitor'@'%';
GRANT SELECT ON sys.* TO 'monitor'@'%';

CREATE USER 'app_user'@'%' IDENTIFIED WITH mysql_native_password BY 'App@123456';
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

FLUSH PRIVILEGES;
```

> MGR 会把这些账号自动同步到另外两台，无需重复创建。

### 3. 登录 ProxySQL 管理端

```bash
mysql -uadmin -padmin -h127.0.0.1 -P6032 --prompt='ProxySQL> '
```

### 4. 配置监控、MGR 分组与后端节点

```sql
UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='Monitor@123' WHERE variable_name='mysql-monitor_password';

INSERT INTO mysql_group_replication_hostgroups
(writer_hostgroup, backup_writer_hostgroup, reader_hostgroup, offline_hostgroup, active, max_writers, writer_is_also_reader, max_transactions_behind)
VALUES (10, 11, 20, 30, 1, 1, 0, 100);

INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections) VALUES (10, '192.168.0.10', 3306, 1000);
INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections) VALUES (10, '192.168.0.20', 3306, 1000);
INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections) VALUES (10, '192.168.0.30', 3306, 1000);

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

| Hostgroup | 含义 |
| :--- | :--- |
| 10 | 写组（PRIMARY） |
| 11 | 备用写组 |
| 20 | 读组（SECONDARY） |
| 30 | 离线组 |

### 5. 业务账号与路由规则

```sql
INSERT INTO mysql_users(username, password, default_hostgroup) VALUES ('app_user', 'App@123456', 10);

-- 带锁的读走主库
INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply) VALUES (1, 1, '^SELECT.*FOR UPDATE', 10, 1);
-- 普通查询走只读库
INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply) VALUES (2, 1, '^SELECT', 20, 1);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

### 6. LOAD / SAVE 四组命令

| 命令 | 作用 |
| :--- | :--- |
| `LOAD MYSQL VARIABLES TO RUNTIME` | 内存 → 运行时，立即生效（连接后端参数） |
| `SAVE MYSQL VARIABLES TO DISK` | 内存 → 磁盘，持久化 |
| `LOAD MYSQL SERVERS TO RUNTIME` | 内存 → 运行时（后端节点列表） |
| `SAVE MYSQL SERVERS TO DISK` | 内存 → 磁盘 |
| `LOAD MYSQL USERS TO RUNTIME` | 内存 → 运行时（业务账号） |
| `SAVE MYSQL USERS TO DISK` | 内存 → 磁盘 |
| `LOAD MYSQL QUERY RULES TO RUNTIME` | 内存 → 运行时（路由规则） |
| `SAVE MYSQL QUERY RULES TO DISK` | 内存 → 磁盘 |

> 只 `LOAD` 不 `SAVE`，重启 ProxySQL 后配置丢失；只 `SAVE` 不 `LOAD`，当前不生效。

### 7. 验证与审计

```sql
-- 看节点有没有被自动分到 10 / 20 组
SELECT hostgroup_id, hostname, status FROM runtime_mysql_servers;

-- 看 monitor 账号探测日志（connect_error 为 NULL 即正常）
SELECT * FROM mysql_server_connect_log;

-- 看 SQL 实际被路由到哪个组
SELECT hostgroup, count_star, digest_text
FROM stats_mysql_query_digest
WHERE digest_text LIKE '%INSERT%' OR digest_text LIKE '%CREATE%' or digest_text like '%select%';
```

```bash
# 业务端口验证：普通 SELECT 走备库（返回 2、3 交替）
mysql -uapp_user -p'App@123456' -h127.0.0.1 -P6033 -e "SELECT @@server_id;"

# 带 FOR UPDATE 的读走主库（固定返回主库 server_id）
mysql -uapp_user -p'App@123456' -h127.0.0.1 -P6033 -e "SELECT @@server_id FOR UPDATE;"
```

## 八、实验案例速记

### 案例 1：主从（node1 → node2）

1. 两台装 MySQL，node1 配 `server-id=1` + `log-bin`，node2 配 `server-id=2` + `relay-log` + `read_only`。
2. node1 建 `repl` 账号，`SHOW MASTER STATUS` 记坐标。
3. node2 执行 `CHANGE MASTER TO …` → `START SLAVE` → 两个线程 Yes。
4. node1 建 `rep_test.users` 并插数据，node2 能查到即成功。

### 案例 2：双主（node1 ⇄ node2）

1. node1 加 `auto_increment_increment=2`、`offset=1`；node2 去掉 `read_only`、加 `log-bin` 和 `offset=2`。
2. node2 补建 `repl` 账号，取自己的 Binlog 坐标。
3. node1 反向 `CHANGE MASTER TO` 指向 node2，两个线程 Yes。
4. 两端分别插入，ID 一奇一偶，且互相可见。

### 案例 3：MGR（node1 + node2 + node3）

1. node3 装 MySQL；node1、node2 执行 `STOP SLAVE; RESET SLAVE ALL; RESET MASTER;` 并删旧配置。
2. 三台写 `mgr.cnf`（GTID + 组名 + seeds + 33061）。
3. 三台建 `mgr_repl` 账号并设 `group_replication_recovery` 通道。
4. 三台写 `/etc/hosts`。
5. node1 `bootstrap_group=ON` → `START GROUP_REPLICATION` → `OFF`；node2、node3 `START GROUP_REPLICATION`。
6. 查 `replication_group_members`，三行 ONLINE。
7. 可选：停 node1 看主切换，再启动并重新加入。

### 案例 4：ProxySQL 读写分离

1. node1 装 ProxySQL，在 PRIMARY 上建 `monitor`、`app_user`。
2. 6032 登录，配置监控账号 + MGR 分组（10/11/20/30）+ 三台 `mysql_servers`。
3. `LOAD/SAVE` 变量与服务器列表，查 `runtime_mysql_servers` 确认分家。
4. 配 `mysql_users` 与两条 query rules（FOR UPDATE → 10，SELECT → 20）。
5. `LOAD/SAVE` 用户与规则。
6. 用 6033 反复 `SELECT @@server_id` 看轮询；用 `stats_mysql_query_digest` 看 INSERT 是否落在 10 组。

## 九、易错点清单

- **8.0.21 语法**：只能用 `CHANGE MASTER TO` / `START SLAVE`，写 `SOURCE/REPLICA` 会语法报错。
- **认证插件**：复制账号要写 `IDENTIFIED WITH mysql_native_password BY …`，默认 `caching_sha2_password` 在无 SSL 时易报错。
- **`read_only=1`**：只挡普通用户，root（SUPER）照样能写；要彻底只读需 `super_read_only=1`。
- **双主要补账号**：node2 的 `repl` 是在主库取坐标之后才创建的，不会同步过去，必须手动补。
- **自增防冲突**：两台步长都设 2，但偏移必须一个 1、一个 2。
- **MGR 组名**：三台 `group_replication_group_name` 必须完全一致。
- **MGR 节点数**：至少 3 台，否则凑不够多数派。
- **`/etc/hosts`**：缺主机名解析会一直卡 `RECOVERING`，`hostname` 查到的名字要和 hosts 一致。
- **bootstrap**：`group_replication_bootstrap_group` 只在第一台临时开，启动成功立刻关。
- **MGR 权限**：只给 `REPLICATION SLAVE`，别加动态权限。
- **ProxySQL 生效**：改完必须 `LOAD … TO RUNTIME`，要重启不丢再 `SAVE … TO DISK`。
- **ProxySQL 账号**：`monitor` 只需 `performance_schema`、`sys` 的 SELECT；`app_user` 才是业务权限。
- **验证写库**：INSERT 不打印节点，用 `SELECT @@server_id FOR UPDATE` 或 `stats_mysql_query_digest` 看 hostgroup。
