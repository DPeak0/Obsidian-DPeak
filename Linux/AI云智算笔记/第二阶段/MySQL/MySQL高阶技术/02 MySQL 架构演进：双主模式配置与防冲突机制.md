---
时间: 2026-10-19
课程: MySQL 高阶技术
节次: 第 2 节
环境: VMware + Rocky Linux 8.4 + MySQL 8.0.21 + 私网 192.168.0.0/24
---

在双主（Active-Active 或 Active-Standby）架构中，node1 和 node2 互为主从。最核心的技术难点在于**防冲突机制**：如果两台服务器同时对同一张表插入数据，自增主键极易产生冲突。因此需要通过设置自增步长和初始偏移量，让 node1 专门生成奇数 ID（1,3,5...），node2 专门生成偶数 ID（2,4,6...）。

## 步骤 1：修改双主防冲突配置

### 1. 在 node1（192.168.0.10）上执行

追加自增参数到配置文件中，设置步长为 2，偏移量为 1（生成奇数 ID）。

```bash
cat >> /etc/my.cnf.d/replication.cnf << EOF
# 双主防冲突：步长与偏移量
auto_increment_increment = 2
auto_increment_offset = 1
EOF

# 重启使配置生效
systemctl restart mysqld
```

### 2. 在 node2（192.168.0.20）上执行

node2 需要从之前的纯从库转变为“主从一体”节点：移除只读限制，开启 Binlog，并设置步长为 2，偏移量为 2（生成偶数 ID）。

```bash
# 覆盖写入新的配置文件
cat > /etc/my.cnf.d/replication.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
server-id = 2
relay-log = mysql-relay-bin
# 开启 Binlog（作为主库必备）
log-bin = mysql-bin
binlog_format = ROW
# 双主防冲突：步长与偏移量
auto_increment_increment = 2
auto_increment_offset = 2
EOF

# 重启使配置生效
systemctl restart mysqld
```

## 步骤 2：在 node2 上补齐同步账号 repl（在 node2 上执行）

第 1 节实验中，`repl` 账号是在 node1 上创建的，随后才执行 `SHOW MASTER STATUS` 获取日志坐标，因此 node2 是从创建账号之后的坐标开始同步数据的，创建账号的那条命令并未同步到 node2 上。需要先在 node2 上手动补齐该账号并刷新权限，才能让 node1 反向连接 node2。

```bash
mysql -uroot -e "
CREATE USER IF NOT EXISTS 'repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY 'Repl@123456';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.0.%';
FLUSH PRIVILEGES;
"
```

## 步骤 3：获取 node2 的主库状态（在 node2 上执行）

```bash
mysql -uroot -e "SHOW MASTER STATUS\G"
```

**核心操作**：请再次记录下 node2 屏幕输出的 `File`（如 `mysql-bin.000001`）和 `Position`（如 `157`）的值。

## 步骤 4：将 node1 配置为 node2 的从库（在 node1 上执行）

### 1. 进入 node1 的 MySQL 命令行

```bash
mysql -uroot
```

### 2. 配置同步链路

请将 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 替换为步骤 3 中在 node2 上获取的值。

```sql
CHANGE MASTER TO
  MASTER_HOST = '192.168.0.20',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'Repl@123456',
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 157;

-- 启动复制线程
START SLAVE;

-- 查看复制状态
SHOW SLAVE STATUS\G
```

确认终端输出中的以下两个字段均为 Yes，即代表双向同步链路建立成功：

- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`

## 步骤 5：双主高可用与防冲突验证

验证两台机器都能独立写入，且 ID 不会冲突，数据还能互相同步。

### 1. 在 node1 上写入数据

```bash
mysql -uroot -e "
USE rep_test;
INSERT INTO users (name) VALUES ('Node1_Insert_A'), ('Node1_Insert_B');
SELECT * FROM users;
"
```

观察输出，新插入的数据 ID 应该为**奇数**（如 3、5）。

### 2. 在 node2 上写入数据并查询

```bash
mysql -uroot -e "
USE rep_test;
INSERT INTO users (name) VALUES ('Node2_Insert_A'), ('Node2_Insert_B');
SELECT * FROM users ORDER BY id ASC;
"
```

观察输出，node2 新插入的数据 ID 应该为**偶数**（如 6、8，取决于前面的自增状态）。此时能在 node2 上看到包含 `Node1_Insert` 和 `Node2_Insert` 的所有完整数据。

### 3. 回到 node1 上再次查询

```bash
mysql -uroot -e "USE rep_test; SELECT * FROM users ORDER BY id ASC;"
```

如果 node1 也能看到 node2 刚刚写入的偶数 ID 数据，说明双主架构完美运行。
