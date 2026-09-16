---
时间: 2026-09-16
---

# YUM/DNF仓库管理包

## 为什么需要YUM/DNF

RPM包之间存在**依赖关系**：安装A软件包，可能依赖于B软件包，必须先安装依赖包才能安装A。
而 `rpm` 命令工具**无法自动解决依赖关系**，所以生产环境中大部分应用都是通过 `yum` 或 `dnf` 来安装管理。
![](https://oss.bwihz.cn/%20PicGo/20260916185355971.png)

- **yum**（Yellowdog Updater Modified）：管理rpm包的工具，自动解决依赖问题。如果识别到rpm包的依赖，yum会**自动先将依赖包安装**
- **dnf**：yum的升级版，从 **RHEL/Rocky 8** 开始，`yum` 命令其实是 `dnf` 的快捷方式（软链接）
![](https://oss.bwihz.cn/%20PicGo/20260916185554475.png)
## 什么是YUM仓库

YUM仓库就是存放rpm包和元数据（repodata）的目录。yum工具通过读取仓库配置文件（`.repo`）找到仓库地址，从仓库中获取软件包信息并安装。
![](https://oss.bwihz.cn/%20PicGo/20260916185627397.png)
仓库类型：

| 类型 | 说明 | baseurl 前缀 |
| :--- | :--- | :--- |
| **本地仓库** | rpm包来自本地（如ISO镜像、本地目录） | `file://` |
| **网络仓库** | rpm包来自网络仓库（如官方仓库） | `https://` / `http://` / `ftp://` |

## 仓库配置文件格式
![](https://oss.bwihz.cn/%20PicGo/20260916190107655.png)
![](https://oss.bwihz.cn/%20PicGo/20260916185958115.png)


```bash
[BaseOS]              # 仓库标识ID，任意定义，必须唯一
name=BaseOS          # 仓库的描述信息
baseurl=file:///media/BaseOS  # 仓库地址，指向 repodata 的上一级目录
gpgcheck=1            # 是否校验rpm包签名（1=校验，0=不校验）
gpgkey==file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial # 公钥文件路径，gpgcheck=1 时必须写
enabled=1             # 是否启用（1=启用，0=关闭，不写默认启用）
```

> [!tip] baseurl 指向规则
> baseurl 必须指向 **repodata 目录的上一级目录**，因为yum工具搜索时会自动查找指定地址下的 `repodata` 目录。

---
## 配置本地YUM仓库

### 准备工作
将系统自带的所有repo文件移走，自己从零配置：
```bash
[root@localhost ~]# cd /etc/yum.repos.d/
[root@localhost yum.repos.d]# mkdir bak
[root@localhost yum.repos.d]# mv *.repo bak/
```
### 步骤一：挂载ISO镜像
![](https://oss.bwihz.cn/%20PicGo/20260916190303734.png)
将ISO镜像文件连接到光驱设备，然后挂载：
```bash
[root@localhost ~]# mount /dev/sr0 /media
mount: /media: WARNING: device write-protected, mounted read-only.

## 验证挂载内容
[root@localhost ~]# ls /media/
AppStream  BaseOS  EFI  images  isolinux  LICENSE  media.repo  TRANS.TBL
```
### 步骤二：创建仓库配置文件

```bash
[root@localhost ~]# vim /etc/yum.repos.d/dvd.repo

[BaseOS]
name=baseos
baseurl=file:///media/BaseOS/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial

[AppStream]
name=appstream
baseurl=file:///media/AppStream/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial
```

> [!warning] 从8版本开始，ISO镜像中的rpm包拆分到了**两个目录**：
> - **BaseOS** — 系统必备的软件包
> - **AppStream** — 常用应用程序的软件包
>
> 7及以下版本所有rpm包都在一个 `Packages` 目录下

### 步骤三：清理缓存并重建
```bash
[root@localhost ~]# yum clean all
13 文件已删除

[root@localhost ~]# yum makecache
baseos                                                567 MB/s | 2.5 MB     00:00
appstream                                             608 MB/s | 7.1 MB     00:00
元数据缓存已建立。
```

### 步骤四：测试安装

```bash
[root@localhost ~]# yum install httpd -y
Last metadata expiration check: 1:31:15 ago on Wed 16 Sep 2026 05:36:33 PM CST.
Dependencies resolved.
============================================================================================
 Package              Arch      Version                                  Repository    Size
============================================================================================
Installing:
 httpd                x86_64    2.4.37-39.module+el8.4.0+571+fd70afb1    AppStream    1.4 M
Installing dependencies:
 apr                  x86_64    1.6.3-11.el8.1                           AppStream    124 k
 apr-util             x86_64    1.6.1-6.el8.1                            AppStream    104 k
 httpd-filesystem     noarch    2.4.37-39.module+el8.4.0+571+fd70afb1    AppStream     37 k
 httpd-tools          x86_64    2.4.37-39.module+el8.4.0+571+fd70afb1    AppStream    105 k
 mod_http2            x86_64    1.15.7-3.module+el8.4.0+553+7a69454b     AppStream    153 k
 rocky-logos-httpd    noarch    84.5-7.el8                               Baseos        22 k
Installing weak dependencies:
 apr-util-bdb         x86_64    1.6.1-6.el8.1                            AppStream     23 k
 apr-util-openssl     x86_64    1.6.1-6.el8.1                            AppStream     26 k

Transaction Summary
============================================================================================
Install  9 Packages

Total size: 2.0 M
Installed size: 5.4 M
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                    1/1
  Installing       : apr-1.6.3-11.el8.1.x86_64                                          1/9
  Running scriptlet: apr-1.6.3-11.el8.1.x86_64                                          1/9
  Installing       : apr-util-bdb-1.6.1-6.el8.1.x86_64                                  2/9
  Installing       : apr-util-openssl-1.6.1-6.el8.1.x86_64                              3/9
  Installing       : apr-util-1.6.1-6.el8.1.x86_64                                      4/9
  Running scriptlet: apr-util-1.6.1-6.el8.1.x86_64                                      4/9
  Installing       : httpd-tools-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64           5/9
  Running scriptlet: httpd-filesystem-2.4.37-39.module+el8.4.0+571+fd70afb1.noarch      6/9
  Installing       : httpd-filesystem-2.4.37-39.module+el8.4.0+571+fd70afb1.noarch      6/9
  Installing       : rocky-logos-httpd-84.5-7.el8.noarch                                7/9
  Installing       : mod_http2-1.15.7-3.module+el8.4.0+553+7a69454b.x86_64              8/9
  Installing       : httpd-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64                 9/9
  Running scriptlet: httpd-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64                 9/9
  Verifying        : rocky-logos-httpd-84.5-7.el8.noarch                                1/9
  Verifying        : apr-1.6.3-11.el8.1.x86_64                                          2/9
  Verifying        : apr-util-1.6.1-6.el8.1.x86_64                                      3/9
  Verifying        : apr-util-bdb-1.6.1-6.el8.1.x86_64                                  4/9
  Verifying        : apr-util-openssl-1.6.1-6.el8.1.x86_64                              5/9
  Verifying        : httpd-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64                 6/9
  Verifying        : httpd-filesystem-2.4.37-39.module+el8.4.0+571+fd70afb1.noarch      7/9
  Verifying        : httpd-tools-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64           8/9
  Verifying        : mod_http2-1.15.7-3.module+el8.4.0+553+7a69454b.x86_64              9/9
Installed products updated.

Installed:
  apr-1.6.3-11.el8.1.x86_64
  apr-util-1.6.1-6.el8.1.x86_64
  apr-util-bdb-1.6.1-6.el8.1.x86_64
  apr-util-openssl-1.6.1-6.el8.1.x86_64
  httpd-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64
  httpd-filesystem-2.4.37-39.module+el8.4.0+571+fd70afb1.noarch
  httpd-tools-2.4.37-39.module+el8.4.0+571+fd70afb1.x86_64
  mod_http2-1.15.7-3.module+el8.4.0+553+7a69454b.x86_64
  rocky-logos-httpd-84.5-7.el8.noarch

Complete!

```

yum会**自动解决依赖关系**，列出所有需要安装的依赖包并一次性安装。

---
# YUM/DNF工具的使用
## 安装、卸载、重装、更新
```bash
yum install 包名       # 安装
yum remove 包名        # 卸载
yum update 包名        # 更新
yum reinstall 包名     # 重装
```
## 查询
### 查询包是否安装
```bash
[root@localhost ~]# yum list --installed | grep httpd
```
### 列出仓库中所有可用的包
```bash
[root@localhost ~]# yum list
```
### 查询包的详细信息
```bash
[root@localhost ~]# yum info httpd
已安装的软件包
名称         : httpd
版本         : 2.4.37
架构         : x86_64
仓库         : @System
来自仓库     : AppStream
概况         : Apache HTTP Server
URL          : https://httpd.apache.org/
描述         : The Apache HTTP Server is a powerful, efficient, and extensible
             : web server.
```
### 根据文件搜索来源
```bash
## 搜索文件来自哪个仓库的哪个包
[root@localhost ~]# yum provides /usr/sbin/useradd
shadow-utils-2:4.6-12.el8.x86_64 : Utilities for managing accounts and shadow password files
仓库        ：BaseOS
匹配来源：
文件名    ：/usr/sbin/useradd
```
### 根据关键字搜索包
```bash
[root@localhost ~]# yum search pod 
上次元数据过期检查：0:07:29 前，执行于 2026年09月16日 星期三 10时58分47秒。
=============================== 名称 和 概况 匹配：pod ===============================
cockpit-podman.noarch : Cockpit component for Podman containers
libgpod.i686 : Library to access the contents of an iPod
libgpod.x86_64 : Library to access the contents of an iPod
pcp-pmda-podman.x86_64 : Performance Co-Pilot (PCP) metrics for podman containers
perl-Pod-Checker.noarch : Check POD documents for syntax errors
perl-Pod-Escapes.noarch : Resolve POD escape sequences
perl-Pod-Html.noarch : Convert POD files to HTML
perl-Pod-LaTeX.noarch : Convert POD data to formatted LaTeX
perl-Pod-Parser.noarch : Basic perl modules for handling Plain Old Documentation (POD)
perl-Pod-Perldoc.noarch : Look up Perl documentation in Pod format
perl-Pod-Plainer.noarch : Perl extension for converting Pod to old-style Pod
perl-Pod-Simple.noarch : Framework for parsing POD documentation
perl-Pod-Usage.noarch : Print a usage message from embedded POD documentation
perl-podlators.noarch : Format POD source into various output formats
podman.x86_64 : Manage Pods, Containers and Container Images
podman-docker.noarch : Emulate Docker CLI using podman
podman-plugins.x86_64 : Plugins for podman
podman-remote.x86_64 : A remote CLI for Podman: A Simple management tool for pods,
                     : containers and images
podman-tests.x86_64 : Tests for podman
=================================== 名称 匹配：pod ===================================
podman-catatonit.x86_64 : A signal-forwarding process manager for containers
=================================== 概况 匹配：pod ===================================
createrepo_c-devel.i686 : Library for repodata manipulation
createrepo_c-devel.x86_64 : Library for repodata manipulation
createrepo_c-libs.i686 : Library for repodata manipulation
createrepo_c-libs.x86_64 : Library for repodata manipulation
librepo.x86_64 : Repodata downloading library
librepo.i686 : Repodata downloading library
perl-Module-Metadata.noarch : Gather package and POD information from perl module
                            : files
toolbox.noarch : Script to launch privileged container with podman

```
### 查看仓库列表
```bash
## 列出启用的仓库
[root@localhost ~]# yum repolist
仓库 id                                    仓库名称
AppStream                                  appstream
BaseOS                                     baseos

## 列出所有状态的仓库（含禁用的）
[root@localhost ~]# yum repolist --all
```
---
# 私有YUM仓库搭建

## 背景
![](https://oss.bwihz.cn/%20PicGo/20260916191324755.png)
大批量Linux主机都需要配置仓库安装软件包。如果每台都用本地仓库，工作量巨大——每台都要连接光驱、挂载、配置repo文件。
解决方案：在局域网内**自建一台YUM仓库服务器**，所有主机的 `baseurl` 都指向这台服务器。未来需要新rpm包，只需要在仓库服务器上更新即可

## 搭建步骤

### 1. 安装httpd提供Web服务
```bash
[root@localhost ~]# yum install httpd -y
```

### 2. 创建仓库目录并下载rpm包
```bash
[root@localhost ~]# mkdir -p /var/www/html/rockylinux8/packages

## --downloadonly 只下载不安装，--destdir 指定保存目录
[root@localhost ~]# yum install nginx --downloadonly --destdir /var/www/html/rockylinux8/packages
```

### 3. 生成repodata元数据

```bash
## 安装createrepo工具
[root@localhost ~]# yum install createrepo_c -y

## 基于rpm包目录生成repodata
[root@localhost ~]# cd /var/www/html/rockylinux8
[root@localhost rockylinux8]# createrepo -v .
```

### 4. 启动服务

```bash
[root@localhost ~]# systemctl stop firewalld
[root@localhost ~]# setenforce 0
[root@localhost ~]# systemctl start httpd
```

### 5. 客户端配置

```bash
## 在客户端的repo文件中指向仓库服务器
[root@client ~]# vim /etc/yum.repos.d/private.repo
[private-baseos]
name=private baseos
baseurl=http://仓库服务器IP/rockylinux8/
gpgcheck=0
enabled=1
```

---

# 源码包和二进制包

## 源码包 vs RPM包

| 对比项 | 源码包 | RPM包 |
| :--- | :--- | :--- |
| 来源关系 | 原始代码 | 基于源码包二次编译打包 |
| 兼容性 | **所有环境都可安装** | 只能在特定发行版安装 |
| 版本 | **更新更快** | 相对落后 |
| 安装方式 | 必须经过**编译安装** | rpm / yum 工具直接安装 |
| 灵活性 | **更灵活**（自定义路径和功能） | 安装路径和功能固定 |
| 依赖管理 | 缺少的依赖**需要自己手动找** | yum自动解决 |

> [!tip] 二进制包
> 二进制包：把源码包编译好的配置文件、可执行文件**重新打包**好了，用户只需要解压缩即可使用。

## 源码包编译安装三步法

```mermaid
graph LR
    A["1. ./configure<br>预配置"] --> B["2. make<br>编译"]
    B --> C["3. make install<br>编译安装"]
```

| 步骤 | 命令 | 作用 |
| :--- | :--- | :--- |
| 预配置 | `./configure` | 检查编译环境、自定义功能和安装路径 |
| 编译 | `make` | 将源码编译为二进制可执行文件和配置文件 |
| 编译安装 | `make install` | 将编译好的文件移动到指定安装路径 |

---

## Nginx源码包安装实战

### Nginx简介

Nginx是一个高性能的Web服务器，同时具备以下功能：

- **Web服务器** — 搭建网站
- **负载均衡器** — 解决高并发问题，减轻后端服务器压力
- **反向代理** — 代理服务端请求
- **缓存服务器** — 缓存网站静态资源
- **邮件代理服务器**

### 安装步骤

#### 1. 下载源码包

```bash
[root@localhost ~]# cd /opt/
[root@localhost opt]# wget https://nginx.org/download/nginx-1.31.6.tar.gz
```

#### 2. 解压缩

```bash
[root@localhost opt]# tar -xf nginx-1.31.6.tar.gz
[root@localhost opt]# ls
nginx-1.31.6  nginx-1.31.6.tar.gz
```

#### 3. 预配置（./configure）

```bash
[root@localhost opt]# cd nginx-1.31.6
[root@localhost nginx-1.31.6]# ./configure --prefix=/usr/local/nginx
```

> [!danger] 常见报错及解决
> 源码编译安装最常见的问题就是**缺少依赖**，需要根据报错信息安装对应的开发包：
>
> | 报错信息 | 缺少的依赖 | 解决命令 |
> | :--- | :--- | :--- |
> | `C compiler cc is not found` | C编译器 | `yum install gcc` |
> | `the HTTP rewrite module requires the PCRE library` | PCRE正则库 | `yum install pcre-devel` |
> | `the HTTP gzip module requires the zlib library` | zlib压缩库 | `yum install zlib-devel` |

```bash
## 逐个解决依赖
[root@localhost nginx-1.31.6]# yum install gcc -y
[root@localhost nginx-1.31.6]# ./configure --prefix=/usr/local/nginx
# ... 报错缺少 pcre ...

[root@localhost nginx-1.31.6]# yum install pcre-devel -y
[root@localhost nginx-1.31.6]# ./configure --prefix=/usr/local/nginx
# ... 报错缺少 zlib ...

[root@localhost nginx-1.31.6]# yum install zlib-devel -y
[root@localhost nginx-1.31.6]# ./configure --prefix=/usr/local/nginx

Configuration summary
  + using system PCRE library
  + using system zlib library
  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
```

#### 4. 编译

```bash
[root@localhost nginx-1.31.6]# make
```

#### 5. 编译安装

```bash
[root@localhost nginx-1.31.6]# make install
```

#### 6. 验证安装

```bash
## 查看安装目录
[root@localhost ~]# ls -l /usr/local/nginx/
总用量 4
drwxr-xr-x. 2 root root 4096  9月 16 14:58 conf    # 配置文件
drwxr-xr-x. 2 root root   40  9月 16 14:58 html    # 网页文件
drwxr-xr-x. 2 root root    6  9月 16 14:58 logs    # 日志文件
drwxr-xr-x. 2 root root   19  9月 16 14:58 sbin    # 可执行程序
```

#### 7. 启动nginx

```bash
## 启动（源码安装的不能用systemctl，需要直接调用二进制文件）
[root@localhost ~]# /usr/local/nginx/sbin/nginx

## 验证进程
[root@localhost ~]# ps -ef | grep nginx
root       17670       1  0 15:01 ?   00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
nobody     17671   17670  0 15:01 ?   00:00:00 nginx: worker process
```

> [!warning] 源码安装 vs RPM安装的服务管理
> - **RPM安装**的软件可以用 `systemctl start/stop/restart` 管理
> - **源码安装**的软件需要直接调用二进制文件或自行编写systemd服务单元文件

---

# 服务管理（systemctl）

## systemd简介

- **systemd** 是系统上 **PID为1** 的守护进程（daemon），其他所有进程都是由systemd派发出来的
- systemd 是 RHEL 7+ 的守护进程，替代了之前的 **initd**（SysVinit）
- `systemctl` 是 systemd 的管理命令

### initd vs systemd

| 对比项 | initd（SysVinit） | systemd |
| :--- | :--- | :--- |
| 版本 | RHEL 6及以下 | **RHEL 7+** |
| PID | 1 | 1 |
| 启动方式 | 串行启动，较慢 | **并行启动，更快** |
| 服务管理 | `service` / `chkconfig` | `systemctl` |

## systemd单元类型

systemctl 根据不同的**单元（unit）**进行管理：

```bash
[root@localhost ~]# systemctl list-unit --type
automount  mount      scope      slice      swap       timer
device     path       service    socket     target
```

最常用的两种单元：

| 单元类型 | 说明 | 示例 |
| :--- | :--- | :--- |
| **service** | 服务单元，运行的软件/应用程序 | `httpd.service` |
| **target** | 启动目标，开机获得什么操作环境 | `multi-user.target`（命令行） |

## 服务管理命令

### 查看服务状态

```bash
[root@localhost ~]# systemctl status httpd.service
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
```

> [!tip] 重点关注
> - **Active** — 服务当前状态（`active (running)` 运行中 / `inactive (dead)` 已停止）
> - **Loaded** — 是否开机自启（`enabled` 开机自启 / `disabled` 不自启）
> - 启动失败时可以查看日志排查

### 启动、停止、重启

```bash
systemctl start 服务名      # 启动服务
systemctl stop 服务名       # 停止服务
systemctl restart 服务名    # 重启服务（先停止再启动）
```

### 重载服务

```bash
systemctl reload 服务名
```

> [!tip] reload vs restart
> - **restart**：先停止再启动，服务有短暂中断
> - **reload**：在**不关闭服务的情况下**重新读取配置文件使其生效
> - 不是所有服务都支持reload — 本质是发送 **SIGHUP（信号1）** 给进程
> - 适用于修改了配置文件但不想中断服务的场景

### 开机自启动

```bash
## 设置开机自启动
[root@localhost ~]# systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.

## 取消开机自启动
[root@localhost ~]# systemctl disable nginx
Removed /etc/systemd/system/multi-user.target.wants/nginx.service.
```

> [!info] 开机自启的原理
> 服务实现开机自启动，本质是在 `/etc/systemd/system/multi-user.target.wants/` 目录下创建了一个**指向服务单元文件的软链接**。

### 列出所有服务单元

```bash
## 列出所有服务单元文件
[root@localhost ~]# systemctl list-unit-files --type service

## 搜索特定服务
[root@localhost ~]# systemctl list-unit-files --type service | grep httpd
httpd.service                              disabled
```

### 服务单元文件位置

```bash
/usr/lib/systemd/system/     # 系统自带的服务单元文件
```

查看httpd的服务单元文件：

```bash
[root@localhost ~]# cat /usr/lib/systemd/system/httpd.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=notify
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND    # 启动命令
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful     # 重载命令
KillSignal=SIGWINCH

[Install]
WantedBy=multi-user.target     # 开机自启时加入的目标
```

## target启动目标

### 运行级别与target对照

| 运行级别 | target | 说明 |
| :--- | :--- | :--- |
| 0 | `poweroff.target` | 关机 |
| 1 | `rescue.target` | 救援模式 |
| 3 | `multi-user.target` | **字符页面（命令行）** |
| 5 | `graphical.target` | **图形化页面** |
| 6 | `reboot.target` | 重启 |

### 管理启动目标

```bash
## 查看当前默认启动目标
[root@localhost ~]# systemctl get-default
graphical.target

## 设置默认启动目标
[root@localhost ~]# systemctl set-default multi-user.target

## 临时切换启动目标（不重启立即生效）
[root@localhost ~]# systemctl isolate multi-user.target

## 查看当前运行级别（旧命令，仍可用）
[root@localhost ~]# runlevel
N 5
```

### 开机临时指定target

开机时按 **e** 进入编辑模式，在 `linux` 行末尾添加对应的target：

```bash
systemd.unit=multi-user.target     # 命令行模式
systemd.unit=rescue.target         # 救援模式
```

按 **Ctrl + X** 启动即可。

---

## 补充：systemctl 常用命令速查表

| 命令 | 作用 |
| :--- | :--- |
| `systemctl start 服务` | 启动服务 |
| `systemctl stop 服务` | 停止服务 |
| `systemctl restart 服务` | 重启服务 |
| `systemctl reload 服务` | 重载配置（不中断服务） |
| `systemctl status 服务` | 查看服务状态 |
| `systemctl enable 服务` | 设置开机自启 |
| `systemctl disable 服务` | 取消开机自启 |
| `systemctl is-active 服务` | 检查服务是否运行 |
| `systemctl is-enabled 服务` | 检查服务是否开机自启 |
| `systemctl get-default` | 查看默认启动目标 |
| `systemctl set-default target` | 设置默认启动目标 |
| `systemctl list-unit-files --type service` | 列出所有服务单元 |
| `systemctl daemon-reload` | 重新加载systemd配置（修改单元文件后执行） |
