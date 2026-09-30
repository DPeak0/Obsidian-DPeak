# **一、项目背景说明：**

| **🏢** **企业背景**                                                                                                                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **公司：**云享科技有限公司（CloudShare Tech）<br><br>**规模：**中型互联网公司，日活用户约 50 万，技术团队 200 人<br><br>**业务：**电商平台 + SaaS 服务，拥有多个生产集群和测试集群<br><br>**现状：**公司正在扩张，需要搭建一套完整的基础运维平台，覆盖虚拟化、网络服务、存储、Web 等核心基础设施。 |
**你的角色：你是刚入职的初级运维工程师（Linux 方向），Leader 给你布置了一项任务——在测试环境中独立搭建并验证公司基础设施的核心组件，为后续正式上线做技术验证。**
# **二、实验环境规划：**

| 主机名                | 角色                                        | 主机配置                                      | 网卡数量及IP地址                                                     |
| ------------------ | ----------------------------------------- | ----------------------------------------- | ------------------------------------------------------------- |
| server01           | DHCP+PXE+DNS主+yum源+chrony时间服务器+Apache+NFS | 内存2G，CPU：2<br><br>硬盘1：50G <br><br>硬盘2：50G | 网卡1： NAT模式，IP地址设置为xxx.xxx.xxx.100/24<br><br>网卡2： 10.10.10.254 |
| kvm-host           | KVM宿主机，DNS辅助，Nginx负载均衡器                   | 内存：8G CPU: 8 硬盘：100G                      | 网卡1： 10.10.10.101                                             |
| kvm-vm1（KVM Guest） | Apache主机                                  | 内存：2G cpu: 2<br><br>硬盘： 20G               | 网卡1： KVM桥接模式，IP地址通过DHCP自动获取，后续在任务七中根据要求修改静态地址                 |
| kvm-vm2（KVM Guest） | Apache主机                                  | 内存：2G cpu: 2<br><br>硬盘： 20G               | 网卡1： KVM桥接模式，IP地址通过DHCP自动获取，后续在任务七中根据要求修改静态地址                 |

# **三、项目需求总览**
任务一：Linux 系统基础配置  5分
任务二：用户与配置权限  5分
任务三：文件系统与归档  5分
任务四：磁盘管理  15分
任务五： DHCP+PXE自动化安装服务器  20分
任务六：KVM虚拟化技术 20分
任务七：DNS服务器 10分
任务八：web服务器 10分
任务九：巡检脚本及计划任务 10分
# **四、任务详情**

## **任务一：Linux 系统基础配置  5分**

前提要求： 测试前在**server01**上提前安装好带图形界面的CentOS8.4系统

1. 配置服务器主机名为 server01.yunxiang.com
```bash
[root@dpeak ~]# hostnamectl set-hostname server01.yunxiang.com
[root@dpeak ~]# bash
[root@server01 ~]# hostname
server01.yunxiang.com
```
2. 配置静态 IP 地址（10.10.10.254/24），网关 10.10.10.254，DNS 指向本机10.10.10.254
```bash
[root@server01 ~]# nmcli connection add type ethernet ipv4.method manual ipv4.addresses 192.168.200.100/24 ipv4.gateway 192.168.200.2 ipv4.dns 223.5.5.5 ifname ens160 con-name ens160 autoconnect yes 
Connection 'ens160' (60f52b1f-d769-40be-ba09-a1856d624521) successfully added.
[root@server01 ~]# nmcli connection add type ethernet ipv4.method manual ipv4.addresses 10.10.10.254/24 ipv4.gateway 10.10.10.254 ipv4.dns 10.10.10.254 ifname ens192 con-name ens192 autoconnect yes 
Connection 'ens192' (80d18411-813d-4dcf-8824-24de6377ba30) successfully added.
[root@server01 ~]# ifconfig 
ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.200.100  netmask 255.255.255.0  broadcast 192.168.200.255
        inet6 fe80::defa:88f2:87a3:5931  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:e4:85:c6  txqueuelen 1000  (Ethernet)
        RX packets 152  bytes 19260 (18.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 113  bytes 12306 (12.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.10.10.254  netmask 255.255.255.0  broadcast 10.10.10.255
        inet6 fe80::27cc:6be1:4a3c:458  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:e4:85:d0  txqueuelen 1000  (Ethernet)
        RX packets 18  bytes 2472 (2.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 34  bytes 3974 (3.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
3. 永久关闭 SELinux和firewalld防火墙
```bash
[root@server01 ~]# systemctl stop firewalld.service
[root@server01 ~]# systemctl disable firewalld.service 
Removed /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

[root@server01 ~]# vim /etc/selinux/config 
[root@server01 ~]# cat /etc/selinux/config | grep -w SELINUX=disabled
SELINUX=disabled
```
4. 将系统时区设置为 Asia/Shanghai
```bash
[root@server01 ~]# timedatectl set-timezone Asia/Shanghai 
```
5. 配置时间服务器，指向ntp.aliyun.com，并允许10.10.10.0/24网络中的计算机可以从该主机同步时间
```bash
[root@server01 ~]# vim /etc/chrony.conf 
[root@server01 ~]# cat /etc/chrony.conf
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#pool 2.pool.ntp.org iburst
pool ntp.aliyun.com iburst

# Allow NTP client access from local network.
#allow 192.168.0.0/16
allow 10.10.10.0/24

# Serve time even if not synchronized to a time source.
#local stratum 10
local stratum 10

```
## **任务二： 用户与配置权限  5分**

1. 创建运维组，名称为yunwei，指定gid=2000
```bash
[root@server01 ~]# groupadd -g 2000 yunwei
[root@server01 ~]# getent group yunwei
yunwei:x:2000:

```

2. 创建it01，it02，ops_admin，ftpuser，webuser五个用户，密码均为RedHat1!
```bash
[root@server01 ~]# vim useradd.sh 
[root@server01 ~]# cat useradd.sh 
#!/bin/bash
for users in $@
do
	useradd $users
	echo 'RedHat1!' | passwd --stdin $users
done
[root@server01 ~]# chmod a+x useradd.sh 
[root@server01 ~]# ./useradd.sh it01 it02 ops_admin ftpuser webuser
Changing password for user it01.
passwd: all authentication tokens updated successfully.
Changing password for user it02.
passwd: all authentication tokens updated successfully.
Changing password for user ops_admin.
passwd: all authentication tokens updated successfully.
Changing password for user ftpuser.
passwd: all authentication tokens updated successfully.
Changing password for user webuser.
passwd: all authentication tokens updated successfully.

```

3. 将以上五个用户均加入yunwei组
```bash
[root@server01 ~]# gpasswd -M it01,it02,ops_admin,ftpuser,webuser yunwei 
[root@server01 ~]# groupmems -g yunwei -l
it01  it02  ops_admin  ftpuser  webuser 

```

4. 配置ops_admin用户可以无密码执行所有sudo命令
```bash
[root@server01 ~]# vim /etc/sudoers
ops_admin       ALL=(ALL)       NOPASSWD:ALL
[root@server01 ~]# su - ops_admin
[ops_admin@server01 ~]$ sudo -l
Matching Defaults entries for ops_admin on server01:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User ops_admin may run the following commands on server01:
    (ALL) NOPASSWD: ALL

```

5. 创建/data/ops目录，设置该目录拥有人为it01，拥有组为yunwei，要求拥有人和拥有组对该目录拥有完整权限，其他人无任何权限
```bash
[root@server01 ~]# mkdir -p /data/ops
[root@server01 ~]# chown it01:yunwei /data/ops
[root@server01 ~]# chmod 770 /data/ops
[root@server01 ~]# ls -ld /data/ops
drwxrwx---. 2 it01 yunwei 6 Oct  1 01:51 /data/ops

```

6. 创建/data/dev目录，设置拥有人和拥有组均为root，使用ACL配置webuser用户对该目录有完整权限，ftpuser用户对该目录无任何权限
```bash
[root@server01 ~]# mkdir /data/dev
[root@server01 ~]# setfacl -m u:webuser:rwx /data/dev/
[root@server01 ~]# setfacl -m u:ftpuser:--- /data/dev/
[root@server01 ~]# getfacl /data/dev/
getfacl: Removing leading '/' from absolute path names
# file: data/dev/
# owner: root
# group: root
user::rwx
user:ftpuser:---
user:webuser:rwx
group::r-x
mask::rwx
other::r-x

```

## **任务三： 文件系统与归档  5分**

1. 查找系统中具有SUID权限的文件，保存到/data/ops目录中，并保留权限
```bash
[root@server01 ~]# find / -path /data -prune -o  -perm -4000 -type f  -exec cp -a {} /data/ops \;
find: ‘/proc/37660/task/37660/fdinfo/5’: No such file or directory
find: ‘/proc/37660/fdinfo/6’: No such file or directory
[root@server01 ~]# ls -l /data/ops/
total 1724
-rwsr-xr-x. 1 root root                58768 Apr 12  2021 at
-rwsr-xr-x. 1 root root                79616 May 19  2021 chage
-rws--x--x. 1 root root                33944 May 19  2021 chfn
-rws--x--x. 1 root root                25552 May 19  2021 chsh
-rwsr-x---. 1 root cockpit-wsinstance  54704 May 19  2021 cockpit-session
-rwsr-xr-x. 1 root root                63368 Mar 15  2021 crontab
-rwsr-x---. 1 root dbus                63656 Jun 11  2021 dbus-daemon-launch-helper
-rwsr-xr-x. 1 root root                37720 Apr 12  2021 fusermount
-rwsr-xr-x. 1 root root                33600 Apr 12  2021 fusermount3
-rwsr-xr-x. 1 root root                84368 May 19  2021 gpasswd
-rwsr-xr-x. 1 root root                12016 May 28  2021 grub2-set-bootflag
-rwsr-x---. 1 root sssd               172208 May 31  2021 krb5_child
-rwsr-x---. 1 root sssd                97528 May 31  2021 ldap_child
-rwsr-xr-x. 1 root root                50584 May 19  2021 mount
-rwsr-xr-x. 1 root root               193872 Jun  2  2021 mount.nfs
-rwsr-xr-x. 1 root root                43560 May 19  2021 newgrp
-rwsr-xr-x. 1 root root                12184 Jun  2  2021 pam_timestamp_check
-rwsr-xr-x. 1 root root                33544 Mar 15  2021 passwd
-rwsr-xr-x. 1 root root                28976 Jun 11  2021 pkexec
-rwsr-xr-x. 1 root root                17016 Jun 11  2021 polkit-agent-helper-1
-rwsr-x---. 1 root sssd                29032 May 31  2021 proxy_child
-rwsr-xr-x. 1 root root                16696 Jun  3  2021 qemu-bridge-helper
-rwsr-x---. 1 root sssd                55472 May 31  2021 selinux_child
-rwsr-xr-x. 1 root root                20832 May 19  2021 spice-client-glib-usb-acl-helper
-rwsr-xr-x. 1 root root                50464 May 19  2021 su
---s--x--x. 1 root root               165632 May 19  2021 sudo
-rwsr-xr-x. 1 root root                33776 May 19  2021 umount
-rwsr-xr-x. 1 root root                37776 Jun  2  2021 unix_chkpwd
-rws--x--x. 1 root root                50168 Apr 19  2021 userhelper
-rwsr-xr-x. 1 root root                12984 Jun  2  2021 vmware-user-suid-wrapper
-rwsr-xr-x. 1 root root                12472 May 19  2021 Xorg.wrap


```

2. 查找系统中所有包含passwd的文件备份至/data目录，并将权限修改为400
```bash

```

3. 将/etc目录打包并压缩至/data/etc.tar.xz

## **任务四：磁盘管理   15分**

1. 在server01主机上，给硬盘2创建5G的分区，格式化为xfs，挂载至/opt/web，要求每次开机均生效

2. 在硬盘2上创建2G逻辑卷/dev/vg0/data，格式化为ext4，挂载至/opt/data目录，要求每次开机均生效，将/etc目录整体备份至该目录中

3. 因数据扩容需要，将该逻辑卷扩容至5G，并确保ext4文件系统也已拉伸成功。拉伸成功后给/dev/vg0/data创建1G容量大小的快照/dev/vg0/snap01，挂载快照验证数据完整性。

4. 给服务器增加2G的swap，确保新增的swap优先级高于系统原有的swap分区。

## **任务五： DHCP+PXE自动化安装服务器  20分**

搭建DHCP+TFTP+PXE+httpd服务器，要求如下：

1. 在server01上安装dhcp,tftp,httpd软件包，确保这些服务每次开机均自动启动

2. 在dhcp服务器中配置作用域为10.10.10.0/24，地址池范围：10.10.10.20-10.10.10.50，网关为10.10.10.254，DNS地址10.10.10.254

3. 将/dev/cdrom挂载至/var/www/html/pub目录上，确保每次开机均自动挂载

4. 制作两个ks文件，ks-host.cfg和ks-vm.cfg，其中ks-host.cfg用于安装kvm-host宿主机，ks-vm.cfg安装kvm-vm1和kvm-vm2

其中**kvm-host.cfg**文件需要安装的系统具有图形界面，安装后脚本%post可以根据后续题目要求自行定义，磁盘分区信息如下：
/boot  500M  
/     60G
swap  4G

kvm-vm.cfg文件用户安装kvm-vm1和kvm-vm2，要求该系统最小化安装，不要安装图形界面
分区信息如下：
/boot  500M
swap   2G
/    10G

5.  通过该服务器安装kvm-host主机，确保该服务器通过kvm-host.cfg安装。

## **任务六：KVM虚拟化技术  20分**

1. 通过上述PXE安装完kvm-host主机后，确保该主机IP地址为10.10.10.101，主机名为kvm-host,在该主机中安装KVM套件

2. 配置桥接器br0

3. 创建/data/kvm-vm1.qcow2和/data/kvm-vm2.qcow2两个精简磁盘的文件，容量为20G

4. 通过PXE分别安装kvm-vm1和kvm-vm2两台虚拟机，磁盘选择上述创建的磁盘文件，网络选择br0，ks文件选择kvm-vm.cfg文件，确保这两台主机安装完成后主机名符合要求。

5. 确保kvm-host和kvm-vm1,kvm-vm2三台主机时间均同步自server01时间服务器。

**6.** **要求在server01主机上通过ssh可以免密访问其他所有主机，并验证。**

## **任务七：DNS服务器  10分**

1. 在server01上搭建主DNS，配置域名为yunxiang.com，在该服务器添加以上所有主机的A记录

server01.yunxiang.com  A 10.10.10.254

kvm-host.yunxiang.com A 10.10.10.101

kvm-vm1.yunxiang.com  A  10.10.10.11

kvm-vm2.yunxiang.com A 10.10.10.12

并添加这些主机的反向解析

2. 在kvm-host主机上配置该DNS的备份DNS，并验证DNS记录同步成功。

## **任务八：web服务器  10分**

1. 在server01上创建/nfsdata目录，使用nfs共享该目录，确保kvm-vm1和kvm-vm2两台主机对该目录可以访问并拥有写权限

2. 在kvm-host安装nginx服务器，提供负载均衡，负载均衡算法为轮循，kvm-vm1权重为1,kvm-vm2权重为2

3. 在kvm-vm1和kvm-vm2上安装apache，并确保每次启动均自动开启该服务，将server01上的nfs共享挂载至/var/www/html，写入内容Hello, yunxiang.com到index.html文件中

当用户输入http:// kvm-host.yunxiang.com可以访问到apache中的内容。

## **任务九： 巡检脚本及计划任务  10分**

在kvm-host主机上编写一个完整的系统巡检脚本 /usr/local/bin/syscheck.sh，功能要求如下：

1. 检查kvm-host主机磁盘分区使用率，超过 80% 时报警，显示”根分区使用率超过80%，请及时处理”

2. 检查以下关键服务状态：named、nginx、chronyd

3.将巡检结果输出到 /var/log/syscheck/check_$(date +%Y%m%d_%H%M).log

4. 创建一个计划任务，要求每周一-周五9-17点每隔5分钟执行一次