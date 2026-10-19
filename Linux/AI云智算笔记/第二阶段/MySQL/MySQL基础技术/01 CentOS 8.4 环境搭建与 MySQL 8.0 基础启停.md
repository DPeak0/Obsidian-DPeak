---
时间: 2026-10-13
课程: MySQL 8.0 数据库
节次: 第 1 节
环境: VMware + CentOS 8.4 + dnf 本地光盘源 + MySQL 8.0
---

**授课目标**：掌握在纯 Linux (CentOS 8.4) 环境下，**脱离网络使用本地光盘源**快速部署 MySQL 8.0 数据库服务器的能力，并熟练掌握基于 `systemctl` 的服务管理，以及 MySQL 8.0 全新的安全初始化流程。

## 一、VMware 虚拟机挂载 CentOS 8.4 DVD 光盘

### 1. 物理挂载（VMware 界面操作）

右键点击虚拟机 → 【设置】(Settings) → 【CD/DVD (IDE)】 → 勾选【**启动时连接**】(Connect at power on) → 选择【使用 ISO 映像文件】(Use ISO image file) → 浏览并选中你的 CentOS 8.4 ISO 文件 → 点击【确定】

### 2. 系统内挂载（Linux 命令行操作）

开启虚拟机，登录 root 账号，执行挂载命令将光盘挂载到 `/media` 目录下。

```bash
# 将系统识别的光驱设备 /dev/cdrom 挂载到 /media 目录
mount /dev/cdrom /media/

# 检查是否挂载成功（查看光盘内的文件）
ls /media/
# 预期会看到 AppStream、BaseOS 等目录
```

> [!important] CentOS 8 光盘结构与 7 不同（记住）
> **核心系统包在 `BaseOS` 目录**，而**应用软件（如 MySQL）在 `AppStream` 目录**。
>
> 所以配置本地源时**必须配两个仓库**（BaseOS + AppStream）。

## 二、配置本地 dnf 光盘源（彻底告别网络依赖）

CentOS 8 使用 **dnf** 作为新一代包管理器。为了保证安装速度和不受网络环境影响，我们配置本地光盘源。

### 1. 备份原有的网络源文件

```bash
# 进入 repo 配置文件目录
cd /etc/yum.repos.d/

# 创建备份文件夹并将所有默认源移入备份
mkdir backup
mv *.repo backup/
```

### 2. 创建本地 dnf 配置文件

```bash
vi /etc/yum.repos.d/local.repo
```

使用 vi 编辑器创建新的 repo 文件：按 `i` 键进入编辑模式，编辑完成后按 `ESC`，输入 `:wq` 保存并退出。

```ini
[BaseOS]
name=CentOS-8-BaseOS
baseurl=file:///media/BaseOS
gpgcheck=0
enabled=1

[AppStream]
name=CentOS-8-AppStream
baseurl=file:///media/AppStream
gpgcheck=0
enabled=1
```

### 3. 清理缓存并生成本地源缓存

```bash
dnf clean all
dnf makecache
```

**测试**：运行 `dnf list mysql-server`，如果能正常列出 MySQL 软件包信息，说明本地源配置成功！

```bash
dnf list mysql-server
```

**`gpgcheck=0` 的原因**：本地光盘源如果开了 `gpgcheck=1`，需要**额外指定 gpgkey 路径**。为了简化实验，这里关掉校验。

生产环境**建议开启校验**：
```ini
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```

## 三、使用 dnf 安装 MySQL 8.0

CentOS 8 的 AppStream 仓库中**已经默认包含了 MySQL 8.0 模块**，这让我们无需再去官网下载繁琐的 rpm 压缩包。

```bash
# -y 参数表示安装过程中自动回答 yes
dnf install mysql-server -y
```

等待进度条跑完，提示 `Complete!` 即代表安装成功。

**也可以用模块流指定版本**

```bash
dnf module list mysql              # 查看可用的 mysql 模块流
dnf module enable mysql:8.0        # 启用 8.0 流
dnf install mysql-server -y
```

## 四、systemctl 服务管理

在 CentOS 8 中，我们使用系统自带的 `systemctl` 来接管 MySQL 服务的启停（**注意服务名为 `mysqld`，不是 `mysql`**）

```bash
# 1. 启动 MySQL 服务
systemctl start mysqld

# 2. 停止 MySQL 服务
systemctl stop mysqld

# 3. 重启 MySQL 服务
systemctl restart mysqld

# 4. 查看 MySQL 运行状态（绿色 active (running) 表示正常运行）
systemctl status mysqld

# 5. 设置 MySQL 开机自启动（企业服务器必备操作）
systemctl enable mysqld
```

> [!warning] 默认只能本机访问
> MySQL 8 默认启动后是 **socket 方式**，但**只能本机使用 MySQL 服务器，不能远程访问**。如果要远程访问，需要开启 MySQL 服务器的 3306 端口：

```bash
vi /etc/my.cnf.d/mysql-server.cnf
```

```ini
[mysqld]
port=3306
bind-address=0.0.0.0
```

```bash
systemctl restart mysqld
```

> [!warning] `bind-address=0.0.0.0` 的安全风险
> 监听所有网卡后，**任何能连通 3306 的主机都可以尝试连接**。
>
> 生产环境必须配合：
> 1. **防火墙限制来源 IP**：`firewall-cmd --add-rich-rule='rule family=ipv4 source address=192.168.10.0/24 port port=3306 protocol=tcp accept'`
> 2. **数据库层限制来源**：创建用户时限定 `@'192.168.10.%'`，**绝不用 `@'%'`**
> 3. 开启 SSL 或走内网专线

## 五、MySQL 8.0 初始密码获取与安全初始化

### 1. 无密码直接登录（CentOS 8 AppStream 版本的特性）

与官方 RPM 包生成随机密码不同，**通过 CentOS 8 dnf 安装的 MySQL 8.0 首次启动时，默认 root 用户没有密码**。

```bash
# 直接登录 MySQL
mysql -u root
```

> [!important] 与官方 RPM 包的区别（记住）
> | 安装方式 | 初始密码 |
> | :--- | :--- |
> | **CentOS 8 dnf 安装** | **root 无密码**，可直接 `mysql -u root` |
> | **MySQL 官方 RPM 包** | 生成**随机密码**，存在 `/var/log/mysqld.log` 中 |
>
> 官方包查密码：`grep 'temporary password' /var/log/mysqld.log`

### 2. 手动修改初始密码

进入 `mysql>` 提示符后，**必须第一时间设置符合 MySQL 8.0 密码强度要求的强密码**（包含大小写字母、数字和特殊符号）。

```sql
-- 修改 root 密码为 'wei@12345'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'wei@12345';

-- 刷新权限并退出
FLUSH PRIVILEGES;
exit;
```

### 3. 使用自带的安全向导加固数据库（推荐做法）

退出 MySQL 后，在 Linux 命令行执行安全配置向导，一步步提升数据库安全性。

```bash
mysql_secure_installation
```

#### 安全向导交互流程解析

| 提示 | 建议输入 | 说明 |
| :--- | :--- | :--- |
| `Enter password for user root:` | `wei@12345` | 输入刚才设置的密码 |
| `VALIDATE PASSWORD COMPONENT...`（是否安装密码校验插件?） | **`y`** | 启用密码强度校验 |
| `Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG:`（密码强度级别） | **`1`** | 中级 |
| `Remove anonymous users?`（是否移除匿名用户?） | **`y`** | 移除匿名用户，安全必备 |
| `Disallow root login remotely?`（是否禁止 root 远程登录?） | **教学环境 `n`，生产环境必选 `y`** | 教学环境方便连接 |
| `Remove test database and access to it?`（是否删除 test 数据库?） | **`y`** | 删除测试库 |
| `Reload privilege tables now?`（是否重新加载权限表?） | **`y`** | 立即生效 |

> [!important] 密码强度等级（记住）
> | 级别 | 要求 |
> | :--- | :--- |
> | **0 = LOW** | 长度 ≥ 8 |
> | **1 = MEDIUM** | 长度 ≥ 8 + **数字 + 大小写字母 + 特殊字符** |
> | **2 = STRONG** | MEDIUM + **必须对照字典文件检查，不能是常见词** |
>
> 生产环境建议用 **2 (STRONG)**。

**第1节小结**：至此，MySQL 8.0 已经以极其标准、安全的姿态运行在 CentOS 8.4 服务器上了。

---
