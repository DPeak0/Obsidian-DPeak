---
时间: 2026-09-15
---

# LVM数据保护

## LVM的快照技术

### 快照技术概述

- LVM的快照技术 —> 保证数据安全性
- LVM的误删除 —> 通过卷组的日志文件实现还原操作

快照技术：**保存拍摄快照瞬间的状态，未来可以基于快照实现数据还原**

LVM支持两种快照技术：

| 快照类型 | 全称 | 原理 |
| :--- | :--- | :--- |
| **COW快照** | Copy-On-Write（写前复制） | 数据修改前，先将**旧数据**复制到快照空间 |
| **ROW快照** | Redirect-On-Write（写时重定向） | 数据修改时，**新数据**写到新位置，原数据保留在原位 |

### COW快照原理

![](https://secure2.wostatic.cn/static/ap6Xj8jsQqaihRoeVmTj16/image.png?auth_key=1789466058-mjUBhUatsDKreWFs3zhEFj-0-62b9f7d96611d33763077a1f7f9df780)

COW快照的工作流程：

1. 创建快照时，**不复制实际数据**，只创建元数据映射
2. 当原卷的某个数据块**被写入前**，先将该块的**旧值**复制到COW空间
3. 然后才将新数据写入原卷
4. 未修改的数据块，快照和原卷**共享同一份数据**

### COW快照还原原理

![](https://secure2.wostatic.cn/static/vgMd17r48Nf7WZfFJ53e8T/image.png?auth_key=1789466058-8NsDQ9auWxXXN5EjapFaGH-0-59e76b676a6507907bb283666c3324c3)

> [!tip] COW vs ROW
> - **COW**：写入时需要**先复制旧数据再写新数据**，有额外开销，但快照数据集中在一个区域
> - **ROW**：写入直接到新位置，原数据不动，**写入性能更好**，但数据分散
> - VMware虚拟机的快照属于**ROW快照**：拍摄快照后会多出一个vmdk磁盘文件，原卷变为只读

### 创建COW快照

**对于LVM的COW快照来说，你需要手动指定COW空间的大小！**

COW快照本质就是一个**特殊类型的LV逻辑卷**，创建快照就是创建一个LV：

```bash
## 语法：lvcreate -n 快照名 -s -L COW空间大小 /dev/卷组名/逻辑卷名
## -s：指定为快照类型

## 示例：对lv1拍摄快照，COW空间64M
[root@localhost ~]# lvcreate -n lv1-snap01 -s -L 64M /dev/vg0/lv1
  Logical volume "lv1-snap01" created.

## 查看快照状态
[root@localhost ~]# lvs
  LV         VG  Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1        vg0 owi-aos---  1.00g
  lv1-snap01 vg0 swi-a-s--- 64.00m      lv1    0.00
  lv2        vg0 -wi-a-----  5.00g
```

> [!info] lvs 输出字段说明
> - `s` — 快照卷（snapshot）
> - `Origin` — 指向原卷
> - `Data%` — COW空间使用百分比，**100%意味着快照失效**

> [!warning] COW空间写满的后果
> **如果COW空间写满了，那么快照则无法使用！** 快照卷必须有足够的空间来存储所有被修改的数据块。

### 基于COW快照还原数据

#### 方法一：挂载快照卷拷贝文件

快照卷可以挂载，挂载后通过cp拷贝文件还原：

> [!warning] 挂载注意事项
> 快照卷的文件系统和原卷**一模一样，甚至UUID都一样**。如果原卷已挂载，快照是**不能挂载**的 —> 必须**先卸载原卷**，再挂载快照。

```bash
## 1. 卸载原卷
[root@localhost ~]# umount /lv1

## 2. 挂载快照卷，拷贝数据到临时目录
[root@localhost ~]# mkdir /lv1-snap /lv1-back
[root@localhost ~]# mount /dev/vg0/lv1-snap01 /lv1-snap/
[root@localhost ~]# cp -a /lv1-snap/* /lv1-back/
[root@localhost ~]# umount /lv1-snap

## 3. 重新挂载原卷，恢复数据
[root@localhost ~]# mount /dev/vg0/lv1 /lv1
[root@localhost ~]# cp -a /lv1-back/* /lv1/
```

#### 方法二：lvconvert 回滚（推荐）

通过专属命令 `lvconvert` 进行回滚 —> 把COW快照直接覆盖原卷，**只能使用一次**

**回滚之前，必须先卸载文件系统！**

```bash
## 1. 卸载原卷
[root@localhost ~]# umount /lv1

## 2. 执行回滚
[root@localhost ~]# lvconvert --merge /dev/vg0/lv1-snap01
  Merging of volume vg0/lv1-snap01 started.
  vg0/lv1: Merged: 100.00%

## 3. 查看结果（快照卷已消失，原卷恢复到快照时刻的状态）
[root@localhost ~]# lvs
  LV   VG  Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1  vg0 -wi-a----- 1.00g
  lv2  vg0 -wi-a----- 5.00g
```

### 快照自动扩容

lvm的配置文件中自带关于快照大小的扩容配置。可以设置指定的百分比阈值，到达了这个百分比就触发扩容，扩容后的大小也是你指定的百分比：

```bash
[root@localhost ~]# vim /etc/lvm/lvm.conf
snapshot_autoextend_threshold = 100  # COW空间使用到此百分比时触发扩容
snapshot_autoextend_percent = 20     # 每次扩容的百分比
```

> [!example] 举例说明
> 假设COW空间是100M：
> - `snapshot_autoextend_threshold = 70` —> 当使用到70M空间时，自动扩容
> - `snapshot_autoextend_percent = 20` —> 100 × 0.2 = 20M —> 扩容到120M

修改后需要重启 `lvm2-monitor` 服务使配置生效：

```bash
[root@localhost ~]# systemctl restart lvm2-monitor
```

### LVM快照实战案例

> [!example] 实战练习
> 1. **准备工作**：准备一个20G的磁盘，当作PV加入到vg0卷组中
> 2. **创建逻辑卷**：基于vg0创建名为lv1，大小为1G的逻辑卷，格式化为xfs，挂载到/lv1
> 3. **拷贝测试文件**：拷贝 `/etc/passwd /etc/shadow /etc/group /etc/gshadow` 到 /lv1
> 4. **拍摄快照**：给lv1拍摄快照，名为lv1-snap，COW空间100M
> 5. **模拟数据丢失**：删除原卷中的所有文件
> 6. **基于快照还原**：还原后重新挂载原卷lv1，验证数据文件存在
> 7. **配置自动扩容**：COW空间使用到50%时自动扩容50%
> 8. **验证扩容**：把/usr/sbin目录拷贝到/lv1，通过lvs查看COW空间大小

---

## LVM的误删除还原

通过卷组的**日志文件**实现还原操作。

LVM对卷组的任何操作都会在 `/etc/lvm/archive` 目录下生成一个对应的 `.vg` 文件。假设误删除了LV，可以借助这些vg日志文件进行还原。

### 查看卷组操作日志

```bash
[root@localhost ~]# ls -l /etc/lvm/archive/
```

### 列出卷组的操作历史

```bash
[root@localhost ~]# vgcfgrestore -l vg0

  File:         /etc/lvm/archive/vg0_00003-2008620836.vg
  VG name:      vg0
  Description:  Created *before* executing 'lvremove /dev/vg0/lv1'
  Backup Time:  Mon Sep 14 15:51:28 2026

  File:         /etc/lvm/archive/vg0_00020-1017164423.vg
  VG name:      vg0
  Description:  Created *before* executing 'lvremove /dev/vg0/lv1 -y'
  Backup Time:  Tue Sep 15 09:39:37 2026
```

> [!tip] 关键理解
> 日志文件记录的是**执行命令之前**的卷组状态。回滚时就是**回滚到执行命令之前的状态**。

### 执行还原

```bash
## 还原到删除lv1之前的vg状态
## -f：指定使用哪个日志文件
[root@localhost ~]# vgcfgrestore -f /etc/lvm/archive/vg0_00031-77537001.vg vg0
  Volume group vg0 has active volume: lv2.
  WARNING: Found 1 active volume(s) in volume group "vg0".
  Restoring VG with active LVs, may cause mismatch with its metadata.
Do you really want to proceed with restore of volume group "vg0", while 1 volume(s) are active? [y/n]: y
  Restored volume group vg0.

## 验证lv1已恢复
[root@localhost ~]# lvs
  LV   VG  Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1  vg0 -wi------- 1.00g
  lv2  vg0 -wi-a----- 5.00g
```

> [!warning] 注意事项
> - `vgcfgrestore` 只能恢复**卷组元数据**（LV的创建/删除/大小变化等），**不能恢复LV中的文件数据**
> - 如果误删除LV后又创建了新的LV占用了相同空间，可能无法完全恢复
> - 最佳实践：**操作前先备份数据**，快照才是数据保护的首选方案

---

# LVM的跨主机迁移

![](https://secure2.wostatic.cn/static/vm7tReQZZeaQMdwVsebnq7/image.png?auth_key=1789466058-rm7M4C8JkrSYwNWoDMhZkg-0-26e3dc8f1f33f784dd973c8dfe2d3296)

应用场景：旧服务器硬件老化需要升级，将硬盘拔出插入新服务器，**数据零拷贝迁移**。

核心原理：LVM的卷组元数据（metadata）保存在每个PV的头部，而不是保存在主机系统中。因此只要把磁盘移到另一台主机，就能通过 `vgexport/vgimport` 让新主机识别原有的VG/LV结构。

### 迁移步骤

#### 1. 卸载文件系统

```bash
[root@localhost ~]# umount /lv1
[root@localhost ~]# umount /lv2
```

#### 2. 禁用卷组

相当于把卷组上所有的逻辑卷禁用掉：

```bash
[root@localhost ~]# vgchange -an vg0
  0 logical volume(s) in volume group "vg0" now active

## 验证LV已变为inactive
[root@localhost ~]# lvscan
  inactive          '/dev/vg0/lv2' [5.00 GiB] inherit
  inactive          '/dev/vg0/lv1' [1.00 GiB] inherit
```

#### 3. 导出卷组

```bash
[root@localhost ~]# vgexport vg0
  Volume group "vg0" successfully exported
```

#### 4. 关机、拔盘

将原有机器设备关机，拔出硬盘，插入到新的机器设备。

#### 5. 导入并激活卷组

```bash
## 扫描PV
[root@localhost ~]# pvscan

## 导入卷组（新机器自动生成卷组信息文件）
[root@localhost ~]# vgimport vg0

## 激活卷组
[root@localhost ~]# vgchange -ay vg0
  2 logical volume(s) in volume group "vg0" now active

## 验证
[root@localhost ~]# lvscan
  ACTIVE            '/dev/vg0/lv2' [5.00 GiB] inherit
  ACTIVE            '/dev/vg0/lv1' [1.00 GiB] inherit

## 挂载使用
[root@localhost ~]# mkdir -p /lv1 /lv2
[root@localhost ~]# mount /dev/vg0/lv1 /lv1
[root@localhost ~]# mount /dev/vg0/lv2 /lv2
```

> [!warning] 迁移注意事项
> - 两台主机的 **LVM版本**需兼容
> - 磁盘接口类型需匹配
> - 迁移前务必**做数据备份**
> - 新机器如果VG名冲突，可用 `vgrename` 改名

---

# 硬链接/软链接

## 软链接（符号链接、快捷方式）

![](https://secure2.wostatic.cn/static/2pGJgZahzkm2mLjwf7N4m/image.png?auth_key=1789466058-jJENQQuhVe18AF3iGiHsLb-0-651b443c64ad7b70f7b359ed4c5b7a04)

软链接类似于Windows的快捷方式，是一个**独立文件**，其数据块中存储的是**源文件的路径字符串**。

访问过程：软链接 inode → 读取路径字符串 → 按路径找到源文件 inode → 读取 block 数据

```bash
## 创建软链接
[root@localhost ~]# ln -s /etc/1.txt /root/1.txt

## 查看软链接
[root@localhost ~]# ls -l /root/1.txt
lrwxrwxrwx. 1 root root 10 9月  15 11:50 /root/1.txt -> /etc/1.txt
```

> [!tip] ls -l 输出解读
> - `l` 开头 — 文件类型为软链接
> - `rwxrwxrwx` — 软链接权限恒为777（实际权限看目标文件）
> - `10` — 文件大小 = 目标路径字符串的字节数（`/etc/1.txt` 正好10个字符）
> - `->` — 指向目标文件

软链接的特点：

- **删除源文件会影响快捷方式**（变成悬空链接）
- **删除快捷方式不会影响源文件**
- **可以跨文件系统制作**（因为存的是路径字符串，不是inode号）
- **可以对目录创建软链接**

## 硬链接（文件副本）

![](https://secure2.wostatic.cn/static/n5XdwbYDYohJqtYKErdaNG/image.png?auth_key=1789466058-gbVK7V2WJi1rkdPcmcy5K8-0-67f7186c634148b82d0a4852695347b5)

硬链接相当于文件在系统中的副本，多个文件名指向**同一个inode**，共享同一份数据。

```bash
## 创建硬链接
[root@localhost ~]# ln /etc/1.txt /root/1.txt

## 验证所有硬链接共享同一inode
[root@localhost ~]# find / -inum 33744605
/etc/sysconfig/1.txt
/root/1.txt
/opt/1.txt
/tmp/1.txt
```

硬链接的特点：

- **所有硬链接的inode完全相同**
- 删除源文件后，硬链接**仍然可以访问数据**
- **无法跨文件系统制作**（因为新文件系统的inode可能已被占用）
- **不能对目录创建硬链接**（防止目录成环）

## 硬链接 vs 软链接对比

| 特性 | 硬链接 | 软链接 |
| :--- | :--- | :--- |
| inode | 与源文件**相同** | **独立的** inode |
| 本质 | 同一inode的多个目录项 | 存储目标路径的独立文件 |
| 跨文件系统 | ❌ 不可以 | ✅ 可以 |
| 对目录 | ❌ 不可以 | ✅ 可以 |
| 原文件删除 | ✅ 仍可访问 | ❌ 失效（悬空链接） |
| 文件类型 | 普通文件 `-` | 链接文件 `l` |
| 占用空间 | 仅增加目录项 | 占用少量空间（存储路径） |
| 原文件改名/移动 | 不影响硬链接 | 路径变化则软链接失效 |

---

# 软件包管理

## 软件包的作用

下载软件包的目的 —> 得到这个软件（应用程序）—> 获取这个软件的功能

- 实现网站功能：下载 httpd、nginx
- 实现文件上传下载：下载 vsftpd、httpd
- 实现共享目录：nfs-utils、samba
- 实现数据库存储：mysql-server、mariadb-server

## 软件包格式差异

Linux发行版分为两大类（按软件包格式和管理工具区分）：

| 分类 | 软件包格式 | 管理工具 | 代表发行版 |
| :--- | :--- | :--- | :--- |
| RHEL系列 | **.rpm** | rpm / yum / dnf | Rocky、CentOS、RHEL、Fedora |
| DEBIAN系列 | **.deb** | dpkg / apt | Debian、Ubuntu |

## RPM包介绍

RPM（Redhat Package Management 红帽包管理器），具备2层含义：

1. **软件包的格式**及后缀（`.rpm`）
2. **管理软件包的命令工具**

### RPM包命名规则

```text
vsftpd-3.0.3-33.el8.x86_64.rpm
│       │     │   │    │     └── 后缀
│       │     │   │    └── 硬件架构
│       │     │   └── 编译平台
│       │     └── 修订次数
│       └── 版本号（主版本.次版本）
└── 软件包名称
```

| 字段 | 含义 | 示例 |
| :--- | :--- | :--- |
| 软件名 | 包的名称 | vsftpd |
| 版本号 | 主版本.次版本 | 3.0.3 |
| 修订次数 | 修了多少次bug | 33 |
| 编译平台 | 在哪个平台上编译的 | el8（Enterprise Linux 8） |
| 硬件架构 | 支持的CPU架构 | x86_64 / aarch64 / noarch |

> [!info] 硬件架构说明
> - **x86_64**：Intel/AMD 的64位CPU
> - **aarch64**：ARM架构的64位CPU
> - **noarch**：无架构限制（通常是纯脚本/配置文件）

查看当前系统架构：

```bash
[root@localhost ~]# hostnamectl | grep Architecture
      Architecture: x86-64
```

### 获取RPM包的方式

1. **ISO镜像文件** —> 百分百兼容操作系统
    ```bash
    ## RHEL 8+ 的rpm包分布在两个目录下
    /run/media/root/Rocky-8-4-x86_64-dvd/
    ├── AppStream   # 常用应用程序的软件包
    └── BaseOS      # 系统必备的软件包

    ## RHEL 7及以下，所有rpm包都在 Packages 目录下
    ```

2. **对应软件的官方网站**获取rpm包
3. **第三方网站**：[rpmfind.net](https://rpmfind.net)
4. **国内开源镜像站**：华为、阿里、清华等（同步国外官网软件到国内）

### RPM管理工具

#### 安装

```bash
rpm -ivh 软件包文件（rpmfile）
  -i：安装（install）
  -v：显示安装的详细信息（verbose）
  -h：显示安装的过程（hash 进度条）

[root@localhost ~]# rpm -ivh vsftpd-3.0.3-36.el8.x86_64.rpm
```

判断是否安装成功：

```bash
## 方式一：通过rpm -q 查询
[root@localhost ~]# rpm -q vsftpd
vsftpd-3.0.3-36.el8.x86_64

## 方式二：查看释放的文件
[root@localhost ~]# ls -l /etc/vsftpd/
```

#### 重装

```bash
## 重新安装（先卸载再安装）
[root@localhost ~]# rpm -ivh --reinstall vsftpd-3.0.3-36.el8.x86_64.rpm
```

#### 更新与降级

```bash
## 更新（已安装则升级，未安装则安装）
rpm -Uvh rpm包文件

## 仅更新（已安装才升级，未安装则报错）
rpm -Fvh rpm包文件

## 降级
rpm -Uvh --oldpackage rpm包文件
```

#### 卸载

```bash
rpm -evh 软件包名
  -e：卸载（erase）— 删除包释放的所有文件

[root@localhost ~]# rpm -evh vsftpd
```

#### 查询

```bash
## 查询系统上所有已安装的软件包
rpm -qa

## 查询指定包是否安装
rpm -q 包名

## 查询包安装后释放的所有文件
rpm -ql 包名/rpmfile

## 查询包的配置文件
rpm -qc 包名

## 查询包的帮助文档
rpm -qd 包名

## 查询系统上的文件来自哪个软件包
rpm -qf 文件路径

## 查询包安装过程执行的脚本
rpm -q --scripts rpmfile

## 查询包的更新日志
rpm -q --changelog rpm包名
```

> [!tip] 常用查询组合
> ```bash
> ## 模糊搜索已安装的网络工具包
> [root@localhost ~]# rpm -qa | grep tools | grep net
> net-tools-2.0-0.52.20160912git.el8.x86_64
>
> ## 查找某个命令属于哪个包
> [root@localhost ~]# rpm -qf /usr/sbin/useradd
> shadow-utils-4.6-12.el8.x86_64
> ```

### RPM校验机制

![](https://secure2.wostatic.cn/static/4XiZg2TBRFTWnu9Am2GoRC/image.png?auth_key=1789466058-qcu8snyCBoPdrByaPgGu7Z-0-c538f87fb0eae3ce003b24b4ac694e42)

默认情况下，使用rpm安装软件包时会自动校验rpm包的**完整性**，校验方式为**数字签名**：

- **数字签名**：2把密钥文件 — 公钥文件和私钥文件
  - 私钥文件做**签名**
  - 公钥文件做**验证**

系统自带的校验公钥文件：

```bash
[root@localhost ~]# ls -l /etc/pki/rpm-gpg/
-rw-r--r--. 1 root root 1672 6月  19 2021 RPM-GPG-KEY-rockyofficial
-rw-r--r--. 1 root root 1672 6月  19 2021 RPM-GPG-KEY-rockytesting
```

> [!warning] 跳过校验
> 安装第三方rpm包时如果提示 `NOKEY`，可以临时跳过校验：
> ```bash
> [root@localhost ~]# rpm -ivh --nodigest --nosignature package.rpm
> ```
> 但这会降低安全性，**生产环境不建议跳过**。

### RPM管理工具实战例题

> [!example] 实战练习
> 1. 前往 rpmfind.net 网站，下载 vsftpd 的 rpm 包到 Linux 主机的 `/opt` 目录下
> 2. 将 ISO 镜像挂载到 `/media` 目录，把 `AppStream/Packages` 下的 vsftpd rpm 包拷贝到 `/opt`
> 3. 安装低版本的 rpm 包，并查询包是否安装
> 4. 列出所有已安装的软件包，同时列出 vsftpd 安装后释放的配置文件
> 5. 将 vsftpd 版本进行更新升级
> 6. 将版本进行降级，再次查询软件包版本
> 7. 卸载安装的 vsftpd

---

## 补充：常用RPM参数速查表

| 参数 | 作用 |
| :--- | :--- |
| `-i` | 安装（install） |
| `-e` | 卸载（erase） |
| `-U` | 升级或安装（upgrade） |
| `-F` | 仅升级（freshen） |
| `-v` | 显示详细信息（verbose） |
| `-h` | 显示进度条（hash） |
| `-q` | 查询（query） |
| `-qa` | 查询所有已安装的包 |
| `-qc` | 查询配置文件 |
| `-ql` | 查询安装释放的文件列表 |
| `-qd` | 查询帮助文档 |
| `-qf` | 查询文件属于哪个包 |
| `--nodeps` | 忽略依赖关系（⚠️ 危险） |
| `--force` | 强制安装/覆盖 |
| `--reinstall` | 重新安装 |
| `--oldpackage` | 降级安装 |
