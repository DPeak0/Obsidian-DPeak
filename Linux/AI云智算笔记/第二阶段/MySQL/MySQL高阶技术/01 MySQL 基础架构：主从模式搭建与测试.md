---
时间: 2026-10-19
课程: MySQL 高阶技术
节次: 第 1 节
环境: VMware + Rocky Linux 8.4 + MySQL 8.0.21 + 私网 192.168.0.0/24
---

**本节目标**：在 node1（192.168.0.10）和 node2（192.168.0.20）上安装 MySQL 8.0，搭建一主一从架构。所有通信均基于私网 IP 进行，以确保传输速度与安全性。

## 一、安装 MySQL 8.0（node1 和 node2 分别执行）

Rocky Linux 8.4 官方源自带 MySQL 8.0，可直接通过 dnf 安装，无需额外配置源。

```bash
# 安装 MySQL 8.0 服务端和客户端
dnf install mysql-server -y

# 启动 MySQL 服务并配置开机自启
systemctl enable --now mysqld

# 确认服务运行正常
systemctl status mysqld
```

## 二、配置主库 Master（在 node1 上执行）

### 1. 写入主库自定义配置文件

```bash
cat > /etc/my.cnf.d/replication.cnf << EOF
[mysqld]
# 监听所有网卡接口
bind-address = 0.0.0.0
# 集群内唯一节点 ID
server-id = 1
# 开启二进制日志
log-bin = mysql-bin
# Binlog 格式使用 ROW（数据一致性最高，最佳实践）
binlog_format = ROW
EOF
```

### 2. 重启 MySQL 使配置生效

```bash
systemctl restart mysqld

# 查看端口号
netstat -tunpl | grep 3306
ss -tunpl | grep 3306
lsof -i:3306 -P
```

### 3. 初始化 root 密码并创建同步账号

MySQL 8.0 首次启动会在日志中生成临时密码，需先登录改密；改密后写入客户端凭据文件 `/root/.my.cnf`，后续所有 `mysql -uroot` 命令即可免密执行。

> [!note] 首次登录
> 如果 `mysql -uroot` 提示需要密码，用 `grep 'temporary password' /var/log/mysqld.log` 取出临时密码登录后再改密；部分光盘源安装版本 root 初始为空密码，可直接登录。两种情况下都应尽快执行 `ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';` 并写好 `/root/.my.cnf`（权限 600）。
> 同步账号要显式使用 `mysql_native_password` 原生认证插件，避免默认 `caching_sha2_password` 在复制连接时需要 SSL/RSA 公钥而报错。

```sql
-- 1. 创建同步账号（仅允许内网 192.168.0.0/24 网段连接，采用原生密码认证）
CREATE USER 'repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY 'Repl@123456';

-- 2. 赋予同步专用权限
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.0.%';

-- 3. 刷新权限字典
FLUSH PRIVILEGES;
```

> [!important] 认证插件要写全
> 只写 `IDENTIFIED BY '密码'` 时，MySQL 8.0 会用默认插件 `caching_sha2_password`，与上面“原生认证”的说法不符。要使用原生插件必须写成 `IDENTIFIED WITH mysql_native_password BY '密码'`。

### 4. 记录主库当前的 Binlog 坐标

```bash
mysql -uroot -e "SHOW MASTER STATUS\G"
```

**核心操作**：请务必记录下此时屏幕输出的 `File`（如 `mysql-bin.000001`）和 `Position`（如 `751`）的值，在步骤 4 配置从库时必须用到。

## 三、配置从库 Slave（在 node2 上执行）

### 1. 写入从库自定义配置文件

```bash
cat > /etc/my.cnf.d/replication.cnf << EOF
[mysqld]
bind-address = 0.0.0.0
# 从库 ID 必须与主库不同
server-id = 2
# 开启中继日志
relay-log = mysql-relay-bin
# 开启全局只读模式 (非 SUPER 权限用户无法写入数据)
read_only = 1
EOF
```

### 2. 重启 MySQL 使配置生效

```bash
systemctl restart mysqld
```

> [!warning] read_only 不是绝对只读
> `read_only=1` 只限制普通用户，**拥有 SUPER 权限的 root 仍然可以写入**。要让 root 也写不进去，需要再加 `super_read_only=1`（本节只需 `read_only` 即可）。

## 四、建立主从同步链路（在 node2 上执行）

### 1. 进入 MySQL 命令行

```bash
mysql -uroot
```

### 2. 配置主库信息

请将下面语句中的 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 替换为步骤 2 中获取的实际值。

```sql
-- 注意：本实验环境为 MySQL 8.0.21，必须使用 MASTER/SLAVE 语法
-- （SOURCE/REPLICA 新术语从 MySQL 8.0.22/8.0.23 才引入，8.0.21 上执行会报语法错误）
CHANGE MASTER TO
  MASTER_HOST = '192.168.0.10',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'Repl@123456',
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 751;

-- 启动复制线程
START SLAVE;

-- 查看复制状态
SHOW SLAVE STATUS\G
```

### 3. 关键状态确认

在终端输出中，必须确保以下两个线程的状态均为 Yes，即代表同步链路建立成功：

- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`

## 五、数据同步功能验证

### 1. 在主库（node1）生成测试数据

```bash
mysql -uroot -e "
CREATE DATABASE rep_test;
USE rep_test;
CREATE TABLE users (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Rocky'), ('HuaweiCloud');
"
```

### 2. 在从库（node2）查询测试数据

```bash
mysql -uroot -e "
USE rep_test;
SELECT * FROM users;
"
```

若在 node2 成功查询出 `Rocky` 和 `HuaweiCloud` 两条数据，则第 1 节实验圆满完成。
