---
时间: 2026-10-19
课程: MySQL 高阶技术
节次: 第 3 节
环境: VMware + Rocky Linux 8.4 + MySQL 8.0.21 + 私网 192.168.0.0/24
---

MGR（MySQL Group Replication，组复制）依靠分布式 Paxos 算法（多数派选举）保证数据强一致性，脑裂自动防御，并内置故障自动转移功能。

- **集群要求**：必须至少 3 个节点（满足 `(N/2)+1` 的多数派，允许 1 台宕机）
- **本节架构**：node1（主）、node2（备 1）、node3（备 2），采用单主模式（Single-Primary），所有写操作都在主节点，宕机后其他两台自动选举出新主

## 步骤 1：node3（第三节点）初始化安装

在 node3（192.168.0.30）上执行安装和启动：

```bash
dnf install mysql-server -y
systemctl enable --now mysqld
```

## 步骤 2：清理旧架构（重置 node1 和 node2）

MGR 是基于 GTID（全局事务 ID）的新型集群，需要清理前两节的传统主从复制配置和状态。在 node1 和 node2 上分别执行：

```bash
# 1. 登录 MySQL 停止旧的同步链路
mysql -uroot -e "
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;
"

# 2. 删除旧的配置文件
rm -f /etc/my.cnf.d/replication.cnf
```

## 步骤 3：为 3 台服务器写入 MGR 专属配置

MGR 的配置要求高度统一，核心是开启 GTID 和配置组间通讯种子节点（通信端口默认使用 33061）。

### 1. 在 node1（192.168.0.10）上执行

```bash
cat > /etc/my.cnf.d/mgr.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
# 开启 GTID
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
binlog_checksum = NONE
# MGR 核心组件配置
plugin_load_add = group_replication.so
group_replication_group_name = '11111111-2222-3333-4444-555555555555'
group_replication_start_on_boot = OFF
group_replication_local_address = '192.168.0.10:33061'
group_replication_group_seeds = '192.168.0.10:33061,192.168.0.20:33061,192.168.0.30:33061'
group_replication_single_primary_mode = ON
transaction_write_set_extraction = XXHASH64
EOF

systemctl restart mysqld
```

### 2. 在 node2（192.168.0.20）上执行

注意：`server-id` 和 `local_address` 与 node1 不同。

```bash
cat > /etc/my.cnf.d/mgr.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 2
log-bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
binlog_checksum = NONE
plugin_load_add = group_replication.so
group_replication_group_name = '11111111-2222-3333-4444-555555555555'
group_replication_start_on_boot = OFF
group_replication_local_address = '192.168.0.20:33061'
group_replication_group_seeds = '192.168.0.10:33061,192.168.0.20:33061,192.168.0.30:33061'
group_replication_single_primary_mode = ON
transaction_write_set_extraction = XXHASH64
EOF

systemctl restart mysqld
```

### 3. 在 node3（192.168.0.30）上执行

注意：`server-id` 和 `local_address` 与另外两台不同。

```bash
cat > /etc/my.cnf.d/mgr.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 3
log-bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
binlog_checksum = NONE
plugin_load_add = group_replication.so
group_replication_group_name = '11111111-2222-3333-4444-555555555555'
group_replication_start_on_boot = OFF
group_replication_local_address = '192.168.0.30:33061'
group_replication_group_seeds = '192.168.0.10:33061,192.168.0.20:33061,192.168.0.30:33061'
group_replication_single_primary_mode = ON
transaction_write_set_extraction = XXHASH64
EOF

systemctl restart mysqld
```

## 步骤 4：创建 MGR 专属同步账号（node1、node2、node3 分别执行一次）

在每一台服务器上建立专属的集群恢复账号，并设置系统复制通道。

> [!warning] 只授予 REPLICATION SLAVE
> 不要授予 `CONNECTION ADMIN`、`GROUP_REPLICATION_STREAM` 等动态权限，否则会报 `Unknown dynamic privilege` 错误。

```bash
mysql -uroot -e "
SET SQL_LOG_BIN=0;
-- 1. 创建账号（显式使用 mysql_native_password，避免恢复通道在非 SSL 下需要 RSA 公钥）
CREATE USER IF NOT EXISTS 'mgr_repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY 'Mgr@123456';

-- 2. 仅赋予核心的数据同步权限，避免动态权限报错
GRANT REPLICATION SLAVE ON *.* TO 'mgr_repl'@'192.168.0.%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;

-- 3. 配置 MGR 的专属恢复通道（8.0.21 使用 CHANGE MASTER TO 语法）
CHANGE MASTER TO MASTER_USER='mgr_repl', MASTER_PASSWORD='Mgr@123456' FOR CHANNEL 'group_replication_recovery';
"
```

> [!note] 为什么要关 Binlog
> `SET SQL_LOG_BIN=0;` 是为了防止这个新建用户的操作被记入日志并传播，确保每个节点独立创建互不冲突。

## 步骤 5：配置主机名解析（3 台服务器上都要执行）

MGR 的 `MEMBER_HOST` 字段显示的是主机名（如 node1 等）而不是 IP。如果节点无法通过主机名解析到对应 IP，加入集群后就会一直卡在 `RECOVERING` 状态无法恢复数据。请将 IP 与主机名的映射写入 3 台服务器的系统配置：

```bash
cat >> /etc/hosts << EOF
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3
EOF
```

请确保上面的主机名和服务器实际的主机名完全一致（可通过 `hostname` 命令查看），如果有出入，请修改为实际的主机名。

## 步骤 6：启动 MGR 集群（引导主节点，在 node1 上执行）

MGR 集群必须由一台机器“引导”建立，这就好比建微信群的第一个人。

```bash
mysql -uroot -e "
-- 开启引导模式
SET GLOBAL group_replication_bootstrap_group=ON;
-- 启动组复制
START GROUP_REPLICATION;
-- 关闭引导模式（关键：启动成功后必须关闭，否则后续网络波动可能导致脑裂）
SET GLOBAL group_replication_bootstrap_group=OFF;
"
```

## 步骤 7：将 node2 和 node3 加入集群（在 node2 和 node3 上分别执行）

```bash
mysql -uroot -e "START GROUP_REPLICATION;"
```

## 步骤 8：验证 MGR 终极高可用状态（在任意一台服务器上执行）

```bash
mysql -uroot -e "
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
"
```

**期望结果**：看到 3 行数据，所有 `MEMBER_STATE` 均为 `ONLINE`，其中一台 `MEMBER_ROLE` 为 `PRIMARY`（可读写），另外两台为 `SECONDARY`（只读备用）。三个节点全部 `ONLINE` 即代表 MGR 集群搭建成功。

## 拓展实验（可选）：MGR 自动故障转移（拔网线测试）

体验 MGR 最核心的自动容灾能力。

### 1. 模拟主库宕机（在 node1 执行）

```bash
systemctl stop mysqld
```

### 2. 见证奇迹（在 node2 执行）

```bash
mysql -uroot -e "SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE FROM performance_schema.replication_group_members;"
```

你会发现 node1 消失或变成 `UNREACHABLE`，而 node2 或 node3 会被自动选举为新的 `PRIMARY`（主节点），业务写操作瞬间接管。

### 3. 恢复宕机节点（在 node1 执行）

```bash
systemctl start mysqld
mysql -uroot -e "START GROUP_REPLICATION;"
```

此时 node1 会以 `SECONDARY`（只读备库）的身份重新加入集群，自动同步宕机期间落下的数据。

## 附录 A：MGR 节点卡在 RECOVERING 状态排查

**现象**：MGR 集群启动后，部分节点状态一直为 `RECOVERING`（恢复中）。

**原因**：当节点加入集群后，需要通过 MySQL 的默认端口（3306）向主库拉取缺失的数据。`MEMBER_HOST` 字段显示的是主机名（如 node1 等）而不是 IP，如果 node2 和 node3 无法通过主机名解析到对应的 IP 地址，它们就无法连上主库，恢复过程就会一直卡住，直到超时。

**修复步骤 1：配置主机名解析（在 3 台服务器上都要执行）**

```bash
cat >> /etc/hosts << EOF
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3
EOF
```

请确保上面的主机名和你查出来的完全一致，如果第三台不叫 node3，请修改为实际的主机名。

**修复步骤 2：重启 node2 和 node3 的组复制（在 node2 和 node3 上分别执行）**

```bash
mysql -uroot -e "
STOP GROUP_REPLICATION;
START GROUP_REPLICATION;
"
```

**修复步骤 3：再次验证状态**（重启完成后，稍微等待 5 到 10 秒钟，在任意一台机器上执行）

```bash
mysql -uroot -e "
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
"
```

此时，三个节点的状态应该全部变成 `ONLINE`。

