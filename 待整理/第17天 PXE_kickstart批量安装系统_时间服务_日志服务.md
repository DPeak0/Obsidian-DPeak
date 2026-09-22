---
时间: 2026-09-18
---

# PXE 预启动技术

## 什么是 PXE

PXE 讲究的就是**通过网络方式来安装操作系统**。对比于传统安装系统方式，支持**多台机器同时安装系统** —> 解决了批量安装系统的问题，不需要人工一台一台地安装，可以让多台服务器设备同时安装操作系统。

![](https://oss.bwihz.cn/%20PicGo/20260921203323018.png)

## 传统安装操作系统的流程

![](https://oss.bwihz.cn/%20PicGo/20260921203323957.png)

### 第一步：挂载光驱设备

于是你就看到上面的安装页面。BIOS 读取到 ISO 中的引导程序 —> **isolinux.bin** —> 加载引导文件 **isolinux.cfg**，就可以看到安装页面了。

![](https://oss.bwihz.cn/%20PicGo/20260921203324962.png)

### 第二步：选择第一个进入图形化的安装页面

指定了内核文件位置、指定了 **initrd.img** 文件位置、指定了 ISO 镜像文件。

**initrd.img** 文件提供了一个**假根文件系统**，内核需要执行这个文件系统中的程序，然后去找到 ISO 中的 **install.img** 文件。

![](https://oss.bwihz.cn/%20PicGo/20260921203326042.png)

**install.img** 提供了图形化的安装页面了。

![](https://oss.bwihz.cn/%20PicGo/20260921203327196.png)

## PXE 的核心问题

pxe 安装系统中，被安装的节点叫做 **pxe 客户端**。pxe 客户端要想看到上面的安装页面，是不是需要找到上面执行的这些文件？

那么问题来了 — 在 pxe 安装过程中，它是通过网络安装的，你并没有给它挂载 ISO 镜像文件，那么**文件从哪里来？**？

> [!important] 答案
> **vmlinuz、isolinux.bin、isolinux.cfg、install.img 等文件，pxe 客户端从哪里来？—> 从 pxe 服务端下载过来的！**
>
> pxe 服务端提供这些安装系统要执行的文件，然后通过网络的方式让 pxe 客户端去下载文件到 pxe 客户端的内存中去执行。

## PXE 依赖的三大服务

### 1. DHCP — 动态主机配置协议

**DHCP**（Dynamic Host Configuration Protocol 动态主机配置协议）

DHCP 设备可以动态地给其他主机分配网络地址：IP地址、子网掩码、网关地址、DNS地址...

例如：`192.168.38.0/24`、`192.168.22.0/24`

#### DHCP 分配网络地址的过程

![](https://oss.bwihz.cn/%20PicGo/20260921203328335.png)

客户端 — 服务端（DHCP设备）：

| 步骤 | 报文 | 说明 |
| :--- | :--- | :--- |
| 1 | **DHCPDISCOVER** | 客户端通过**广播**发送数据包，询问谁是 DHCP 设备 |
| 2 | **DHCPOFFER** | 服务端 DHCP 设备回应，分配一个 IP 地址然后回应给客户端 |
| 3 | **DHCPREQUEST** | 客户端请求 DHCP 设备分配的 IP 地址 |
| 4 | **DHCPACK** | 服务端 DHCP 设备确认，最终将 IP 地址分配给客户端 |

> [!tip] 记忆口诀
> **DORA** — Discover / Offer / Request / Ack
>
> 广播找服务器 → 服务器给offer → 客户端确认要这个 → 服务器最终确认

### 2. TFTP — 简单文件传输协议

**tftp** 是简单的文件传输协议，实现小文件的下载和上传功能。在 pxe 中，主要是为客户端提供**引导文件的下载**。

- 默认的目录在 `/var/lib/tftpboot` 目录下
- 所以你想要客户端去下载引导文件，需要将其引导文件放到上面目录下

> [!warning] 关键点：引导程序文件不同
> **在 pxe 安装系统中，引导程序文件不能是 isolinux.bin，需要使用一个网络引导程序文件 —> `pxelinux.0`**

![](https://oss.bwihz.cn/%20PicGo/20260921203329425.png)

### 3. HTTP — 共享 ISO 文件

通过 HTTP 共享 ISO 中的文件。

---

## PXE 批量安装系统实战

**准备工作**：最少 2 台主机

- 1 台主机需要有操作系统 —> **pxe 服务端**
- 1 台主机可以没有操作系统 —> **pxe 客户端**

**目的**：通过 pxe 网络安装系统，让 pxe 客户端安装 rocky linux 8 系统

### 服务端配置步骤

#### 1. 配置本地 YUM 仓库

```bash
# 先备份存在的 repo 文件
[root@localhost ~]# cd /etc/yum.repos.d/
[root@localhost yum.repos.d]# mkdir bak
[root@localhost yum.repos.d]# mv *.repo bak/
[root@localhost yum.repos.d]# ls
bak

# 生成 repo 文件
cat > /etc/yum.repos.d/dvd.repo <<EOF
[BaseOS]
name=BaseOS
baseurl=file:///media/BaseOS
gpgcheck=0
enabled=1

[AppStream]
name=AppStream
baseurl=file:///media/AppStream
gpgcheck=0
enabled=1
EOF

[root@localhost yum.repos.d]# mount /dev/sr0 /media
mount: /media: WARNING: device write-protected, mounted read-only.
```

#### 2. 安装所需软件包

```bash
[root@localhost ~]# yum install httpd dhcp-server tftp-server syslinux-nonlinux -y
```

| 软件包 | 作用 |
| :--- | :--- |
| `dhcp-server` | 提供 DHCP 服务，分配 IP 并指向 tftp 服务器 |
| `tftp-server` | 提供引导文件下载 |
| `httpd` | 共享 ISO 内容（提供 install.img 和 rpm 包） |
| `syslinux-nonlinux` | 提供 **pxelinux.0** 网络引导程序 |

#### 3. 关闭防火墙和 SELinux

```bash
# 临时关闭
[root@localhost ~]# systemctl stop firewalld
[root@localhost ~]# setenforce 0

# 永久关闭
[root@localhost ~]# systemctl disable firewalld
[root@localhost ~]# sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

#### 4. 配置 DHCP

指定分配的网络对应的子网、DNS、网关、指定 tftp 机器的 IP 地址、指定网络引导文件 pxelinux.0。

```bash
[root@localhost ~]# cd /etc/dhcp/
[root@localhost dhcp]# cp /usr/share/doc/dhcp-server/dhcpd.conf.example dhcpd.conf
[root@localhost dhcp]# vim dhcpd.conf

# A slightly different configuration for an internal subnet.
subnet 192.168.200.0 netmask 255.255.255.0 {   # 子网，必须是 dhcp 设备存在的子网，不能虚拟
  range 192.168.200.100 192.168.200.110;       # dhcp 地址池，客户端动态得到 ip 地址就是从这个地址池划分的
  option domain-name-servers 223.5.5.5;        # 分配的 dns 地址
  option domain-name "yutianedu.com";          # 分配的域，比如主机名是 servera，未来解析的时候自动变成 servera.yutianedu.com
  option routers 192.168.200.2;                # 网关地址
  option broadcast-address 192.168.200.255;    # 广播地址，可以不写
  default-lease-time 600;                      # 默认租约时间，客户端获取的 ip 地址快到期的话会自动续约
  max-lease-time 7200;                         # 最大租约时间
  next-server 192.168.200.137;                 # 指定 tftp 服务器的 ip 地址，让 pxe 客户端知道去找谁下载文件
  filename "pxelinux.0";                       # 指定网络引导程序文件，默认是相对路径，会去 /var/lib/tftpboot 目录下找这个文件
}
```

> [!info] 关键参数说明
> - **next-server** — 告诉客户端 TFTP 服务器在哪里（**这是 PXE 的枢纽**）
> - **filename** — 告诉客户端下载哪个引导文件（相对路径，基于 `/var/lib/tftpboot`）

#### 5. 配置 TFTP

让引导文件、配置文件、引导程序文件的依赖全部放入到这个 TFTP 对应的目录下。

默认访问目录：`/var/lib/tftpboot`

```bash
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# cp /media/isolinux/* .
[root@localhost tftpboot]# ls
boot.cat   initrd.img    ldlinux.c32   memtest     vesamenu.c32
boot.msg   isolinux.bin  libcom32.c32  splash.png  vmlinuz
grub.conf  isolinux.cfg  libutil.c32   TRANS.TBL
```

**pxelinux.0 文件也需要放到这个目录下**：

```bash
[root@localhost tftpboot]# cp /usr/share/syslinux/pxelinux.0 .
[root@localhost tftpboot]# ls
boot.cat   initrd.img    ldlinux.c32   memtest     TRANS.TBL
boot.msg   isolinux.bin  libcom32.c32  pxelinux.0  vesamenu.c32
grub.conf  isolinux.cfg  libutil.c32   splash.png  vmlinuz
```

**pxelinux.0 作为网络引导程序文件，读取的引导配置文件必须是存放到 `pxelinux.cfg/default` 中**：

```bash
[root@localhost tftpboot]# mkdir pxelinux.cfg
[root@localhost tftpboot]# cp isolinux.cfg pxelinux.cfg/default
```

#### 6. 配置 HTTP

让客户端能够访问到 ISO 镜像中的文件，最终得到 install.img 文件进入图形化的安装页面。同时这个 HTTP 也可以提供 rpm 包的路径，未来 pxe 客户端安装系统的相关 rpm 包的时候也是从这个路径下去得到 rpm 和安装的。

```bash
[root@localhost ~]# cd /var/www/html/
[root@localhost html]# mkdir iso
[root@localhost html]# mount /dev/sr0 iso/
mount: /var/www/html/iso: WARNING: device write-protected, mounted read-only.
[root@localhost html]# ls iso/
AppStream  BaseOS  EFI  images  isolinux  LICENSE  media.repo  TRANS.TBL
```

#### 7. 修改引导配置文件

定义倒计时、定义默认选择的项、定义读取的镜像文件中的文件在哪个路径。

```bash
[root@localhost ~]# cd /var/lib/tftpboot/pxelinux.cfg/
[root@localhost pxelinux.cfg]# vim default

timeout 60            # 倒计时 60 秒

label linux
  menu label ^Install Rocky Linux 8 For yutianedu
  menu default                                            # 默认选中项
  kernel vmlinuz
  append initrd=initrd.img inst.repo=http://192.168.200.137/iso quiet
```

#### 8. 启动服务

```bash
[root@localhost pxelinux.cfg]# systemctl start dhcpd tftp httpd
```

![](https://oss.bwihz.cn/%20PicGo/20260921203330473.png)

---

## kickstart 脚本文件

### 为什么需要 ks 脚本

最终进入到了图形化的安装页面，但是我发现：

**时区、root 用户密码、安装目的地、网络和主机名**这些配置，还是需要人工来实现。

如果数量一多，还是消耗人力。为了让它能够实现自动化配置这些内容，于是出现了一个叫做 **ks 脚本文件**。

**ks 脚本文件的作用**：提前定义这些配置内容，在进行 pxe 批量安装系统的时候，这些 pxe 客户端会下载 ks 脚本文件，根据文件中定义的内容去执行对应的操作，全部都是采用自动化实现。

![](https://oss.bwihz.cn/%20PicGo/20260921203331348.png)

> [!info] ks 脚本的生成方式
> - **原来**：基于模板文件手动修改
> - **现在**：提供了图形化生成工具 —> **system-config-kickstart**
> - ⚠️ 但这个图形化工具从 **8 版本开始已经被舍弃掉了**，最后一个拥有的版本是 **7**

### ks 脚本示例

```bash
[root@localhost html]# pwd
/var/www/html
[root@localhost html]# ls ks/
ks.cfg
```

```text
#platform=x86, AMD64, 或 Intel EM64T
#version=DEVEL

# Install OS instead of upgrade
install

# Keyboard layouts
keyboard 'us'

# Root password
rootpw --iscrypted $1$lOboUwAI$y19Mo0jcA7ss1AXi.xkFC1

# System language
lang en_US

# System authorization information
auth  --useshadow  --passalgo=sha512

# Use graphical install
graphical

# SELinux configuration
selinux --disabled

# Do not configure the X Window System
skipx

# Firewall configuration
firewall --disabled

# Network information
network  --bootproto=dhcp --device=ens160

# Reboot after installation
reboot

# System timezone
timezone Africa/Abidjan

# Use network installation
url --url="http://192.168.200.137/iso"

# System bootloader configuration
bootloader --location=mbr

# Clear the Master Boot Record
zerombr

# Partition clearing information
clearpart --all --initlabel

# Disk partitioning information
part /boot --fstype="xfs" --size=512
part /     --fstype="xfs" --size=10240

%post
touch /opt/xym
%end

%packages
@fonts
%end
```

> [!tip] ks 脚本核心指令
> | 指令 | 作用 |
> | :--- | :--- |
> | `install` | 全新安装（非升级） |
> | `rootpw --iscrypted` | root 密码（加密形式） |
> | `url --url=` | 安装源地址 |
> | `zerombr` | 清除 MBR |
> | `clearpart --all --initlabel` | 清空所有分区 |
> | `part / --fstype= --size=` | 分区方案 |
> | `%post ... %end` | 安装后执行的脚本 |
> | `%packages ... %end` | 要安装的软件包 |

### 引导配置指向 ks 文件

引导配置文件需要指定 ks 的位置，这样 pxe 客户端才知道去哪里下载：

```bash
[root@localhost ~]# cd /var/lib/tftpboot/pxelinux.cfg/
[root@localhost pxelinux.cfg]# vim default

label linux
  menu label ^Install Rocky Linux 8 For yutianedu
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.repo=http://192.168.200.137/iso  inst.ks=http://192.168.200.137/ks/ks.cfg quiet

  ### inst.ks=http://192.168.200.137/ks/ks.cfg
```

> [!important] 关键参数
> - `inst.repo=` — 安装源（ISO 内容，提供 rpm 包）
> - `inst.ks=` — kickstart 脚本位置（自动化配置文件）

---

## PXE + kickstart 实战案例

> [!example] 实战练习
> 准备 2 台主机：
> - 1 台 pxe 服务端
> - 1 台 pxe 客户端
>
> 要求通过 **pxe + kickstart** 方式来实现 pxe 客户端系统**自动安装**

---

# 时间服务

## NTP 与 chronyd

Linux 系统上存在一个叫做 **chronyd** 服务，这个服务是用来**代替 ntp 服务**的。

- **ntp**：时间授时协议
- **架构**：C/S 架构
- 客户端通过 ntp 向服务端**同步时间**

![](https://oss.bwihz.cn/%20PicGo/20260921203332658.png)

| 对比 | ntp（ntpd） | chrony（chronyd） |
| :--- | :--- | :--- |
| 定位 | 传统时间服务 | **NTP 的替代品**（RHEL 7+ 默认） |
| 同步速度 | 较慢 | **更快**（尤其是不稳定网络） |
| 时钟漂移处理 | 一般 | **更好** |
| 对间歇性网络 | 适应性差 | **适应性好** |

## 查看时间源

查看当前的时间源服务器有哪些，并且还可以看到对应的状态：

```bash
[root@localhost ~]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 203.107.6.88                  2   6    37     7  -6487ns[ +670us] +/-   32ms
```

> [!info] 状态符号含义
> | 符号 | 含义 |
> | :--- | :--- |
> | `^*` | **时间同步成功**（当前使用的源） |
> | `^+` | 可接受的备选源 |
> | `^?` | **和时间源服务器同步有问题** |
> | `^-` | 被排除的源 |

## 修改时间源

让主机向其他的时间源同步时间 — 修改 chronyd 的配置文件 `/etc/chrony.conf`：

```text
pool ntp.aliyun.com iburst
```

```bash
[root@localhost ~]# systemctl restart chronyd
[root@localhost ~]# chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? 203.107.6.88                  2   6     3     2   +573us[ +573us] +/-   31ms
^? 113.141.164.39                0   6     0     -     +0ns[   +0ns] +/-    0ns
^? 113.141.164.38                4   6     1     1    -19ms[  -19ms] +/-  146ms
^? 1.82.219.234                  0   6     0     -     +0ns[   +0ns] +/-    0ns
```

> [!tip] iburst 参数
> `iburst` 表示启动时**快速发送多个包**（而不是每 64 秒一个），使时间同步**更快完成**。推荐始终加上。

## timedatectl 命令

`timedatectl` 命令：既可以修改系统时间、也可以修改时区。

```bash
[root@localhost ~]# timedatectl
               Local time: Mon 2026-09-21 14:21:54 CST
           Universal time: Mon 2026-09-21 06:21:54 UTC
                 RTC time: Mon 2026-09-21 06:21:54
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

| 字段 | 说明 |
| :--- | :--- |
| `Local time` | 本地时间（已应用时区） |
| `Universal time` | UTC 世界标准时间 |
| `RTC time` | 硬件时钟（CMOS） |
| `Time zone` | 当前时区 |
| `System clock synchronized` | 是否已同步（yes = 同步成功） |
| `NTP service` | NTP 服务是否激活 |

常用操作：

```bash
# 查看所有可用时区
[root@localhost ~]# timedatectl list-timezones

# 设置时区
[root@localhost ~]# timedatectl set-timezone Asia/Shanghai

# 手动设置时间（需先关闭 NTP）
[root@localhost ~]# timedatectl set-ntp false
[root@localhost ~]# timedatectl set-time "2026-09-21 14:30:00"

# 开启 NTP 同步
[root@localhost ~]# timedatectl set-ntp true
```

## 时间源服务器部署

**背景**：未来在很多的业务场景中，主机之间的时间要求严格，但是可能大部分的主机都是**纯内网环境，不能上网**。

我们会**自建一个内部的时间源服务器**，让内网中的其他主机向我同步时间。

> [!info] chronyd 是 C/S 架构，既是客户端也是服务端

### 服务端配置

将自己当前主机作为时间源服务器，让别的主机可以向我同步：

```bash
[root@localhost ~]# vim /etc/chrony.conf
...
# Allow NTP client access from local network.
allow 192.168.200.0/24          # 允许谁向我同步时间

# Serve time even if not synchronized to a time source.
local stratum 10                # 将本机作为时间源服务器
...

[root@localhost ~]# systemctl restart chronyd
[root@localhost ~]# systemctl stop firewalld
[root@localhost ~]# setenforce 0
```

> [!tip] local stratum 参数
> `local stratum 10` 表示**即使本机没有同步到外部时间源，也作为第 10 层级的时间服务器对外提供服务**。数字越大层级越低（精度越差），10 是常用的内部时间源层级。

### 客户端配置

指定向谁同步：

```bash
[root@localhost ~]# grep 192.168.200.137 /etc/chrony.conf
pool 192.168.200.137 iburst
[root@localhost ~]# systemctl restart chronyd

[root@localhost ~]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 192.168.200.137               3   6    17     4  -1364ns[-6606ns] +/-   40ms
```

---

# 日志服务

## 两大日志服务

Linux 系统自带 2 大日志服务：

- **systemd-journald**
- **rsyslog**

![](https://oss.bwihz.cn/%20PicGo/20260921203333577.png)

用户日志、系统日志、服务日志、计划任务日志...

我们可以根据日志信息来判断程序或者系统是否存在问题：

- 启动某个服务发现报错了 —> 查看日志
- 启动系统发现报错无法正常进入 —> 查看系统日志
- 给用户修改密码 —> 查看安全日志

## journald vs rsyslog 对比

![](https://oss.bwihz.cn/%20PicGo/20260921203334491.png)

| 对比项 | systemd-journald | rsyslog |
| :--- | :--- | :--- |
| 默认存储 | **易失性**（内存） | **持久化**（磁盘） |
| 日志位置 | `/run/log/journal` | `/var/log/` |
| 日志分类 | 仅存在**一个文件** | **分类存储**（不同日志不同文件） |
| 查看方式 | `journalctl`（**专用工具**） | 文本工具直接查看（**普通文本文件**） |
| 二进制格式 | 是（不可直接 cat） | 否（纯文本） |

### journald 日志文件位置（易失性）

```bash
[root@localhost ~]# ls -l /run/log/journal/
total 0
drwxr-s---+ 2 root systemd-journal  60 Sep 21 13:58 45d6a6a91742449b874bd1fe186aa1c3
drwxr-s---+ 2 root systemd-journal 100 Sep 21 14:10 e7d346cf5c374f319ac9465707809f31

[root@localhost ~]# ls -l /run/log/journal/45d6a6a91742449b874bd1fe186aa1c3/
total 8192
-rw-r-----+ 1 root systemd-journal 8388608 Sep 21 13:58 system.journal
```

### rsyslog 日志文件位置（持久化）

```bash
[root@localhost ~]# ls /var/log/
anaconda         dnf.log      libvirt   speech-dispatcher     vmware-vgauthsvc.log.0
audit            dnf.rpm.log  maillog   spooler               vmware-vmsvc-root.log
boot.log         firewalld    messages  sssd                  vmware-vmtoolsd-root.log
btmp             gdm          private   swtpm                 vmware-vmusr-root.log
chrony           glusterfs    qemu-ga   tuned                 wtmp
cron             hawkey.log   rhsm      vmware-network.1.log  Xorg.9.log
cups             httpd        samba     vmware-network.2.log
dnf.librepo.log  lastlog      secure    vmware-network.log
```

---

## rsyslog 日志服务

![](https://oss.bwihz.cn/%20PicGo/20260921203335396.png)

rsyslog 日志服务针对于系统进行**分类**，将不同的日志信息存放到不同的日志文件中：

| 日志文件 | 内容 |
| :--- | :--- |
| `/var/log/secure` | **安全日志**（登录、认证、sudo） |
| `/var/log/cron` | **计划任务日志** |
| `/var/log/boot.log` | **启动日志** |
| `/var/log/maillog` | **邮件日志** |
| `/var/log/messages` | **大部分日志信息**都存放到这个文件（服务启动报错看这里） |

> [!tip] 排错思路
> - 服务**启动失败** —> 看 `/var/log/messages`
> - 服务正常启动，但**无法访问** —> 看服务自己的日志（如 `/var/log/httpd`）

### 日志格式解读

![](https://oss.bwihz.cn/%20PicGo/20260921203336527.png)

一条日志由 4 部分组成：

1. **日志产生的时间**
2. **日志在哪个主机上产生的**（显示的是主机名）
3. **日志是由哪个进程产生的**（后面括号中的数字是**进程 ID**）
4. **日志信息**

### rsyslog 配置文件

配置文件：`/etc/rsyslog.conf`

```text
*.info;mail.none;authpriv.none;cron.none                /var/log/messages

# The authpriv file has restricted access.
authpriv.*                                              /var/log/secure

# Log all the mail messages in one place.
mail.*                                                  -/var/log/maillog

# Log cron stuff
cron.*                                                  /var/log/cron

# Everybody gets emergency messages
*.emerg                                                 :omusrmsg:*

# Save news errors of level crit and higher in a special file.
uucp,news.crit                                          /var/log/spooler

# Save boot messages also to boot.log
local7.*                                                /var/log/boot.log
```

### 日志规则语法

```text
日志类型.日志级别        日志文件路径
```

**日志类型**（哪些程序产生的日志）：

| 类型 | 说明 |
| :--- | :--- |
| `mail` | 邮件 |
| `cron` | 计划任务 |
| `authpriv` | 安全 |
| `*` | 所有类型 |

**日志级别**（从低到高）：

![](https://oss.bwihz.cn/%20PicGo/20260921203337524.png)

| 级别 | 说明 |
| :--- | :--- |
| `debug` | 调试信息 |
| `info` | 一般信息 |
| `notice` | 需要注意 |
| `warning` / `warn` | 警告 |
| `err` / `error` | 错误 |
| `crit` | 严重 |
| `alert` | 警报（需立即处理） |
| `emerg` / `panic` | 紧急（系统不可用） |
| `none` | 不记录 |

> [!tip] 通配符规则
> - `*.info` — **所有类型的 info 及以上级别**
> - `mail.none` — 排除 mail 类型
> - `authpriv.*` — authpriv 类型的**所有级别**

### 自定义日志文件实战

**需求**：要求 info 级别以及 info 以上级别的**任意类型**的日志都记录到 `/var/log/yutianedu.log` 日志文件中。

```bash
[root@localhost ~]# vim /etc/rsyslog.conf
*.info                                                  /var/log/yutianedu.log

[root@localhost ~]# systemctl restart rsyslog
```

通过 `logger` 命令手动发送日志信息进行测试：

```bash
[root@localhost ~]# logger -i -t yutian -p mail.info '这是一条测试日志信息'
```

| 参数 | 作用 |
| :--- | :--- |
| `-i` | 记录进程 ID |
| `-t` | 指定进程名 |
| `-p` | 指定**日志类型.日志优先级** |
| 最后一个参数 | 日志信息 |

---

## systemd-journald 日志服务

### 日志持久化存储

默认的日志文件保存在 `/run/log/journal`（易失性），可通过配置改为持久化。

配置文件：`/etc/systemd/journald.conf`

![](https://oss.bwihz.cn/%20PicGo/20260921203338479.png)

`Storage` 参数有三个值：

| 值 | 说明 |
| :--- | :--- |
| `auto` | **需要人为手动创建** `/var/log/journal` 目录才会持久化 |
| `persistent` | **无论目录是否存在，都会持久化存储** |
| `none` | **不记录日志** |

**auto 模式下手动开启持久化**：

```bash
[root@localhost ~]# mkdir -p /var/log/journal
[root@localhost ~]# systemctl restart systemd-journald
[root@localhost ~]# journalctl --flush
[root@localhost ~]# ls -l /var/log/journal
total 0
drwxr-xr-x. 2 root root 28 Sep 21 16:15 937ddda267354b09af8a4d7796e12021
```

> [!tip] 三行搞定持久化
> ```bash
> mkdir -p /var/log/journal
> systemctl restart systemd-journald
> journalctl --flush      # 把内存中的日志刷到磁盘
> ```

### journalctl 命令

`journalctl` 命令专门查看 journald 日志：

```bash
# 实时监控 journald 日志的追加情况
journalctl -f

# 查看指定程序服务的日志
journalctl -u sshd

# 查看指定时间段的日志信息
journalctl --since "2026-09-21 13:00:00" --until "2026-09-21 15:00:00"
journalctl --since "1 hour ago"

# 查看详细信息
journalctl -xe

# 查看指定日志级别
journalctl -p err
```

**journalctl -p 可用的日志级别**：

```bash
[root@localhost ~]# journalctl -p
alert    crit     debug    emerg    err      info     notice   warning
```

### 按字段过滤

`journalctl _字段=值` 可以精确过滤：

```bash
[root@localhost ~]# journalctl _
_AUDIT_LOGINUID=              _KERNEL_SUBSYSTEM=            _SYSTEMD_SESSION=
_AUDIT_SESSION=               _MACHINE_ID=                  _SYSTEMD_SLICE=
_BOOT_ID=                     _PID=                         _SYSTEMD_UNIT=
_CAP_EFFECTIVE=               _SELINUX_CONTEXT=             _SYSTEMD_USER_SLICE=
_CMDLINE=                     _SOURCE_MONOTONIC_TIMESTAMP=  _SYSTEMD_USER_UNIT=
_COMM=                        _SOURCE_REALTIME_TIMESTAMP=   _TRANSPORT=
_EXE=                         _STREAM_ID=                   _UDEV_DEVNODE=
_GID=                         _SYSTEMD_CGROUP=              _UDEV_SYSNAME=
_HOSTNAME=                    _SYSTEMD_INVOCATION_ID=       _UID=
_KERNEL_DEVICE=               _SYSTEMD_OWNER_UID=
```

常用字段：

| 字段 | 作用 |
| :--- | :--- |
| `_BOOT_ID=` | 查看指定**启动**日志 |
| `_COMM=` | 查看指定**命令**产生的日志 |
| `_EXE=` | 查看指定**命令文件**产生的日志 |
| `_HOSTNAME=` | 查看指定**主机名**的日志 |
| `_PID=` | 查看指定**进程**的日志 |
| `_UID=` | 查看指定**用户**的日志 |
| `_SYSTEMD_UNIT=` | 查看指定**服务单元**的日志 |

![](https://oss.bwihz.cn/%20PicGo/20260921203339521.png)

---

## logrotate 日志轮询

**日志轮询服务**：目的是为了解决**日志文件占据存储空间越来越大**的问题。

即**日志切割**。

```bash
[root@localhost ~]# cat /etc/logrotate.conf
# see "man logrotate" for details
# rotate log files weekly
weekly                  # 每周一轮询切割

# keep 4 weeks worth of backlogs
rotate 4                # 保留的副本数量

# create new (empty) log files after rotating old ones
create                  # 切割之后创建一个空日志文件

# use date as a suffix of the rotated file
dateext                 # 以时间格式对切割后的文件进行命名

# uncomment this if you want your log files compressed
#compress               # 开启日志文件压缩

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d    # 引入 /etc/logrotate.d 目录下的配置文件
```

| 参数 | 含义 |
| :--- | :--- |
| `weekly` | 切割周期（还可用 `daily`、`monthly`、`yearly`） |
| `rotate 4` | 保留 4 个历史副本，超出则删除最旧的 |
| `create` | 切割后新建空日志文件（可指定权限：`create 0644 root root`） |
| `dateext` | 用日期作为后缀（如 `messages-20260921`），而非数字序号 |
| `compress` | 压缩历史日志（`gzip`） |
| `include /etc/logrotate.d` | 引入各服务自己的轮询配置 |

> [!info] 各服务的独立配置
> 大多数服务（httpd、nginx、syslog 等）在 `/etc/logrotate.d/` 下有自己的轮询配置文件，可以**单独定制**切割策略。

---

# 补充：本篇核心服务速查

| 服务 | 配置文件 | 作用 |
| :--- | :--- | :--- |
| **dhcpd** | `/etc/dhcp/dhcpd.conf` | 分配 IP + 指向 TFTP 服务器 |
| **tftp** | `/var/lib/tftpboot` | 提供引导文件下载 |
| **httpd** | `/var/www/html` | 共享 ISO 内容 |
| **chronyd** | `/etc/chrony.conf` | 时间同步（C/S 架构） |
| **rsyslog** | `/etc/rsyslog.conf` | 分类持久化日志 |
| **systemd-journald** | `/etc/systemd/journald.conf` | 统一日志（默认易失） |
| **logrotate** | `/etc/logrotate.conf` | 日志切割轮询 |

> [!tip] PXE 安装流程速记
> ```
> 客户端开机
>   ↓ DHCP 获取 IP + 得知 TFTP 地址 + 得知引导文件名
>   ↓ TFTP 下载 pxelinux.0 引导程序
>   ↓ TFTP 下载 pxelinux.cfg/default 配置文件
>   ↓ TFTP 下载 vmlinuz（内核）+ initrd.img
>   ↓ HTTP 下载 inst.repo（ISO 内容）+ inst.ks（ks 脚本）
>   ↓ 按 ks 脚本自动安装完成，重启
> ```
