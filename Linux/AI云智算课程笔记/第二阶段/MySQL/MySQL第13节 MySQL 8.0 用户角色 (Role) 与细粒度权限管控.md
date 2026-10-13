---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 13 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：彻底告别企业开发中"**使用 root 账号打天下**"的危险操作。掌握 MySQL 用户生命周期管理与密码安全策略，精通 `GRANT`/`REVOKE` 细粒度权限控制，并深度解析 MySQL 8.0 极具跨时代意义的新特性——**角色 (Role)**，实现大规模技术团队权限的标准化、模块化下发。

## 一、用户基础管理与密码过期策略

在生产环境中，我们需要为**不同的开发人员和业务系统创建独立的低权限账号**，并**严格限制其登录的 IP 范围**。

### 1. 创建用户与访问控制

```sql
-- 创建一个只能在本地登录的用户
CREATE USER 'local_dev'@'localhost' IDENTIFIED BY 'wei@12345';

-- 创建一个允许从指定网段 (如内网实训机房 192.168.10.x) 远程登录的用户
CREATE USER 'remote_dev'@'192.168.0.%' IDENTIFIED BY 'wei@12345';

-- 创建一个允许从任意外部 IP 登录的用户 (生产环境极其危险，仅作测试)
CREATE USER 'web_app'@'%' IDENTIFIED BY 'wei@12345';
```

#### `@` 后面主机地址的写法

| 写法 | 含义 |
| :--- | :--- |
| `'user'@'localhost'` | **只能本机**（socket 或 127.0.0.1） |
| `'user'@'127.0.0.1'` | 只能通过本机 TCP 连接 |
| `'user'@'192.168.0.%'` | 允许 **192.168.0.0/24 网段** |
| `'user'@'192.168.0.137'` | 只允许**这一个 IP** |
| **`'user'@'%'`** | **允许任意 IP（生产高危）** |

**`%` 的通配规则**：`%` 匹配**任意长度的任意字符**（包括空）。
`192.168.%.%` 匹配 `192.168.*.*`。

⚠️ `_` 是匹配单个字符的通配符。

### 2. 密码过期与强制修改

为了满足**企业等保三级的安全合规要求**，数据库管理员需要强制用户定期修改密码。

```sql
-- 设置 web_app 用户的密码有效期为 90 天
ALTER USER 'web_app'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- 强制该用户在下次登录时必须立即修改密码
ALTER USER 'web_app'@'%' PASSWORD EXPIRE;
```

| 密码策略语句 | 作用 |
| :--- | :--- |
| `PASSWORD EXPIRE` | **立即过期**，下次登录必须改 |
| `PASSWORD EXPIRE INTERVAL n DAY` | **n 天后过期** |
| `PASSWORD EXPIRE NEVER` | **永不过期** |
| `PASSWORD EXPIRE DEFAULT` | 使用全局默认策略 |

> [!warning] 密码过期后登录会报错
> ```text
> ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
> ```
> 用户需要自行执行 `ALTER USER USER() IDENTIFIED BY '新密码';` 才能继续使用。

### 3. 其他用户管理语句

```sql
-- 查看所有用户
SELECT user, host FROM mysql.user;

-- 修改密码
ALTER USER 'local_dev'@'localhost' IDENTIFIED BY 'newpass@123';

-- 锁定/解锁账户
ALTER USER 'local_dev'@'localhost' ACCOUNT LOCK;
ALTER USER 'local_dev'@'localhost' ACCOUNT UNLOCK;

-- 删除用户
DROP USER 'local_dev'@'localhost';
```

## 二、细粒度权限分配 (GRANT / REVOKE)

创建好的新用户**默认没有任何权限（连基础的 SELECT 都不行）**。我们需要通过 `GRANT` 命令精确赋予权限，遵循"**最小权限原则**"。

### 1. 精确赋权 (GRANT)

**切忌滥用 `GRANT ALL`。真实的业务场景需要把权限控制到具体的数据库，甚至是具体的表。**

```sql
-- 场景：赋予 remote_dev 针对 mydb 数据库下所有表的 增、删、改、查 权限
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'remote_dev'@'192.168.0.%';

-- 场景：极其严格的控制，仅允许对 user 表进行只读操作
GRANT SELECT ON mydb.user TO 'local_dev'@'localhost';
```

#### 权限作用范围（作用域从大到小）

| 作用域 | 写法 |
| :--- | :--- |
| 全局 | `*.*` |
| 单库所有表 | `mydb.*` |
| **单表** | **`mydb.user`** |
| 单列 | `mydb.user(username)` |

#### 常用权限速查

| 权限 | 作用 |
| :--- | :--- |
| `SELECT` / `INSERT` / `UPDATE` / `DELETE` | 增删改查 |
| `CREATE` / `DROP` / `ALTER` | 库表结构变更 |
| `INDEX` | 创建/删除索引 |
| `CREATE VIEW` / `SHOW VIEW` | 视图相关 |
| `EXECUTE` | 执行存储过程 |
| `TRIGGER` | 创建触发器 |
| `FILE` | 文件导入导出（**高危**） |
| `PROCESS` / `SUPER` | **管理权限（高危）** |
| **`ALL PRIVILEGES`** | **全部权限** |

### 2. 回收权限 (REVOKE) 与权限审查

如果员工调岗或安全策略收紧，我们需要**随时回收某些敏感权限**。

```sql
-- 撤销 remote_dev 用户的 DELETE (删除) 权限
REVOKE DELETE ON mydb.* FROM 'remote_dev'@'192.168.0.%';

-- 审查该用户目前到底拥有哪些权限
SHOW GRANTS FOR 'remote_dev'@'192.168.0.%';
```

**改完权限要 `FLUSH PRIVILEGES` 吗**

**用 `GRANT`/`REVOKE`/`CREATE USER` 语句改权限，会自动刷新，不需要手动 `FLUSH PRIVILEGES`。**

**只有直接修改 `mysql.user` 等权限表**（不推荐的做法）才需要 `FLUSH PRIVILEGES`。

## 三、MySQL 8.0 核心新特性：角色 (Role) 架构设计

**痛点引入**：在企业级项目交付中，一个运维开发团队可能有 **20 个研发、5 个数据分析师**。如果按照上面的方法，DBA 需要给这 **25 个人每人写一遍 `GRANT` 语句**。一旦项目新增了一个表，**又要执行 25 次 `GRANT`**，这不仅低效且极易漏配。

**MySQL 8.0 破局方案：角色 (Role)。**
**角色就是一个"权限的集合"。你可以把权限先赋予给角色，然后再把角色直接发给对应的员工。**

### 1. 创建角色并打包权限

```sql
-- 创建两个标准化角色：'app_developer' (开发组) 和 'app_readonly' (只读分析组)
CREATE ROLE 'app_developer', 'app_readonly';

-- 将读写权限打包赋予开发组角色
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_developer';

-- 将只读权限打包赋予只读分析组角色
GRANT SELECT ON mydb.* TO 'app_readonly';
```

**角色默认是锁定的**：MySQL 8.0 创建的角色**默认处于锁定状态**（不能直接当用户登录），这是**安全设计**。

需要的话可以解锁：
```sql
ALTER USER 'app_developer' ACCOUNT UNLOCK;
```

### 2. 将角色分配给具体用户

先创建用于演示的示例用户（张三、李四、王五），再将角色分配给他们：

```sql
-- 创建示例用户（仅作角色分配演示）
CREATE USER 'zhangsan'@'%' IDENTIFIED BY 'wei@12345';
CREATE USER 'lisi'@'%' IDENTIFIED BY 'wei@12345';
CREATE USER 'wangwu'@'%' IDENTIFIED BY 'wei@12345';

-- 将开发组角色分配给张三和李四
GRANT 'app_developer' TO 'zhangsan'@'%', 'lisi'@'%';

-- 将只读角色分配给审计员王五
GRANT 'app_readonly' TO 'wangwu'@'%';
```

### 3. 角色激活机制（MySQL 8.0 巨坑防雷）

> [!danger] MySQL 8.0 最容易踩的坑（记住）
> **把角色分配给用户后，默认情况下角色是未激活状态，用户登录依然没有任何权限。必须显式激活！**

#### 方法一：手动激活（DBA 强制为用户激活其拥有的所有角色）

```sql
SET DEFAULT ROLE ALL TO 'zhangsan'@'%', 'lisi'@'%', 'wangwu'@'%';
```

#### 方法二：一劳永逸的全局配置法（强烈推荐）

修改 CentOS 8 中的 `/etc/my.cnf.d/mysql-server.cnf` 文件，在 `[mysqld]` 下方添加：

```ini
activate_all_roles_on_login = ON
```

配置完成后重启 `mysqld` 服务，从此**所有用户登录时将自动激活其绑定的角色权限**。

```bash
systemctl restart mysqld
```

#### 方法三：用户自己激活（会话级）

```sql
SET ROLE ALL;                      -- 激活所有已授予的角色（当前会话）
SET ROLE 'app_developer';          -- 只激活指定角色
SELECT CURRENT_ROLE();             -- 查看当前激活的角色
```

### Role 操作速查

| 语句 | 作用 |
| :--- | :--- |
| `CREATE ROLE 'r1','r2';` | 创建角色 |
| `GRANT 权限 ON db.* TO 'r1';` | **给角色打包权限** |
| `GRANT 'r1' TO 'user'@'host';` | **把角色授予用户** |
| **`SET DEFAULT ROLE ALL TO 'user';`** | **设置默认激活的角色** |
| `SET ROLE ALL;` | 当前会话激活所有角色 |
| `SELECT CURRENT_ROLE();` | 查看当前角色 |
| `REVOKE 'r1' FROM 'user';` | 回收角色 |
| `DROP ROLE 'r1';` | 删除角色 |

> [!important] 本节核心价值（记住）
> 本节内容完成了**从单点权限到企业级角色矩阵的跨越**。
>
> **"权限 → 角色 → 用户"** 三层结构：
> - 加新表 → 只需 `GRANT` 给角色 **1 次**
> - 加新人 → 只需 `GRANT` 角色给用户 **1 次**
> - 调整权限 → 改角色，**所有绑定用户自动生效**

---
