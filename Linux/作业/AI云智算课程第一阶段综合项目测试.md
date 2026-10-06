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
[root@server01 ~]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 203.107.6.88                  2   6    17     7   +877us[+2549us] +/-   36ms

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
[root@server01 ~]# find /data -maxdepth 1 -name '*passwd*' -type f -exec chmod 400 {} \;
[root@server01 ~]# find /data -maxdepth 1 -name '*passwd*' -type f -ls
 34651979    116 -r--------   1  root     root       116316 Oct  1  2026 /data/passwd-0.80-3.el8.x86_64.rpm
 34651980      4 -r--------   1  root     root          497 Oct  1  2026 /data/passwd
 34651981      0 -r--------   1  root     root            0 Oct  1  2026 /data/opasswd
 34651982      4 -r--------   1  root     root         2806 Oct  1  2026 /data/passwd-
 34651983    288 -r--------   1  root     root       294704 Oct  1  2026 /data/grub2-mkpasswd-pbkdf2
 34651984      4 -r--------   1  root     root          605 Oct  1  2026 /data/gpasswd
 34651985     40 -r--------   1  root     root        38648 Oct  1  2026 /data/vncpasswd
 34651986     72 -r--------   1  root     root        71416 Oct  1  2026 /data/chgpasswd
 34651987      4 -r--------   1  root     root          601 Oct  1  2026 /data/chpasswd
 34651988     20 -r--------   1  root     root        17000 Oct  1  2026 /data/saslpasswd2
 34651989     24 -r--------   1  root     root        20976 Oct  1  2026 /data/lpasswd
 34651990      4 -r--------   1  root     root          221 Oct  1  2026 /data/kpasswd.xml
 34651991     48 -r--------   1  root     root        45192 Oct  1  2026 /data/smbpasswd.so
 34651992      4 -r--------   1  root     root          809 Oct  1  2026 /data/passwd-cb.pl
 34651993      4 -r--------   1  root     root          439 Oct  1  2026 /data/passwd.mo
 34651994      4 -r--------   1  root     root          324 Oct  1  2026 /data/grub2-mkpasswd-pbkdf2.1.gz
 34651995      4 -r--------   1  root     root         3079 Oct  1  2026 /data/sslpasswd.1ssl.gz
 34651996      4 -r--------   1  root     root         2753 Oct  1  2026 /data/gpasswd.1.gz
 34651997      4 -r--------   1  root     root         1108 Oct  1  2026 /data/lpasswd.1.gz
 34651998      8 -r--------   1  root     root         4128 Oct  1  2026 /data/passwd.1.gz
 34651999      4 -r--------   1  root     root         1136 Oct  1  2026 /data/vncpasswd.1.gz
 34652000      4 -r--------   1  root     root         2762 Oct  1  2026 /data/chgpasswd.8.gz
 34652001      4 -r--------   1  root     root         1872 Oct  1  2026 /data/chpasswd.8.gz
 34652002      4 -r--------   1  root     root         1556 Oct  1  2026 /data/saslpasswd2.8.gz
 34652003      4 -r--------   1  root     root         2678 Oct  1  2026 /data/smbpasswd.5.gz
 34652004      4 -r--------   1  root     root         2709 Oct  1  2026 /data/passwd.5.gz
 34652005      4 -r--------   1  root     root           38 Oct  1  2026 /data/passwd2des.3.gz
 34652006      4 -r--------   1  root     root         2443 Oct  1  2026 /data/passwd.vim
 34652007      8 -r--------   1  root     root         4468 Oct  1  2026 /data/masterpasswd.aug
 34652008      4 -r--------   1  root     root         1043 Oct  1  2026 /data/htpasswd.aug
 34652009      4 -r--------   1  root     root         3609 Oct  1  2026 /data/passwd.aug
 34652010      4 -r--------   1  root     root          920 Oct  1  2026 /data/htpasswd
 34652011      4 -r--------   1  root     root         1199 Oct  1  2026 /data/passwd.awk

```

3. 将/etc目录打包并压缩至/data/etc.tar.xz
```bash
[root@server01 ~]# tar -cJf /data/etc.tar.xz /etc
```
## **任务四：磁盘管理   15分**

1. 在server01主机上，给硬盘2创建5G的分区，格式化为xfs，挂载至/opt/web，要求每次开机均生效
```bash
[root@server01 ~]# mkdir /opt/web
[root@server01 ~]# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x2d61474d.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-104857599, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-104857599, default 104857599): +5G

Created a new partition 1 of type 'Linux' and of size 5 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@server01 ~]# mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@server01 ~]# echo '/dev/sdb1 /opt/web xfs defaults 0 0' >> /etc/fstab
[root@server01 ~]# mount -a
[root@server01 ~]# df | grep sdb1
/dev/sdb1        5232640   69544   5163096   2% /opt/web

```

2. 在硬盘2上创建2G逻辑卷/dev/vg0/data，格式化为ext4，挂载至/opt/data目录，要求每次开机均生效，将/etc目录整体备份至该目录中
```bash
[root@server01 ~]# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2):
First sector (10487808-104857599, default 10487808):
Last sector, +sectors or +size{K,M,G,T,P} (10487808-104857599, default 104857599): +10G                                                                         
Created a new partition 2 of type 'Linux' and of size 10 GiB.

Command (m for help): w
The partition table has been altered.
Syncing disks.

[root@server01 ~]# pvcreate /dev/sdb2
  Physical volume "/dev/sdb2" successfully created.
[root@server01 ~]# vgcreate vg0 /dev/sdb2
  Volume group "vg0" successfully created
[root@server01 ~]# lvcreate -n data -L 2G vg0
  Logical volume "data" created.
[root@server01 ~]# mkfs.ext4 /dev/vg0/data
mke2fs 1.45.6 (20-Mar-2020)
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: 5eb61766-49f9-40a9-bb8d-04e83ece3919
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

[root@server01 ~]# mkdir /opt/data
[root@server01 ~]# echo '/dev/vg0/data /opt/data ext4 defaults 0 0' >> /etc/fstab
[root@server01 ~]# mount -a
[root@server01 ~]# df | grep data
/dev/mapper/vg0-data   1998672    6144   1871288   1% /opt/data
[root@server01 ~]# cp -a /etc /opt/data/
[root@server01 ~]# ls /opt/data/
etc  lost+found
```

3. 因数据扩容需要，将该逻辑卷扩容至5G，并确保ext4文件系统也已拉伸成功。拉伸成功后给/dev/vg0/data创建1G容量大小的快照/dev/vg0/snap01，挂载快照验证数据完整性。
```bash
[root@server01 ~]# lvextend -L 5G /dev/vg0/data
  Size of logical volume vg0/data changed from 2.00 GiB (512 extents) to 5.00 GiB (1280 extents).
  Logical volume vg0/data successfully resized.
[root@server01 ~]# lvs
  LV   VG  Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data vg0 -wi-ao---- 5.00g
  
[root@server01 ~]# resize2fs /dev/vg0/data
resize2fs 1.45.6 (20-Mar-2020)
Filesystem at /dev/vg0/data is mounted on /opt/data; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/vg0/data is now 1310720 (4k) blocks long.
[root@server01 ~]# df -h | grep data
/dev/mapper/vg0-data  4.9G   39M  4.6G   1% /opt/data

  
[root@server01 ~]# lvcreate -n snap01 -s -L 1G /dev/vg0/data
  Logical volume "snap01" created.
[root@server01 ~]# lvs
  LV     VG  Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data   vg0 owi-aos--- 5.00g
  snap01 vg0 swi-a-s--- 1.00g      data   0.01
  
[root@server01 ~]# mkdir /opt/snap01
[root@server01 ~]# mount /dev/vg0/snap01 /opt/snap01
[root@server01 ~]# ls /opt/snap01
etc  lost+found

```

4. 给服务器增加2G的swap，确保新增的swap优先级高于系统原有的swap分区。
```bash
[root@server01 ~]# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (3,4, default 3):
First sector (31459328-104857599, default 31459328):
Last sector, +sectors or +size{K,M,G,T,P} (31459328-104857599, default 104857599): +2G

Created a new partition 3 of type 'Linux' and of size 2 GiB.

Command (m for help): w
The partition table has been altered.
Syncing disks.

[root@server01 ~]# mkswap /dev/sdb3
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
no label, UUID=4c141d6a-d94b-4d20-bc5f-3090e2e30e05

[root@server01 ~]# swapon -p 666 /dev/sdb3
[root@server01 ~]# swapon -s
Filename                                Type            Size    Used    Priority
/dev/sdb3                               partition       2097148 0       666

```
## **任务五： DHCP+PXE自动化安装服务器  20分**

搭建DHCP+TFTP+PXE+httpd服务器，要求如下：
1. 在server01上安装dhcp,tftp,httpd软件包，确保这些服务每次开机均自动启动
```bash
[root@server01 ~]# yum install -y dhcp-server tftp-server httpd syslinux-nonlinux
[root@server01 ~]# systemctl enable dhcpd tftp httpd
Created symlink /etc/systemd/system/multi-user.target.wants/dhcpd.service → /usr/lib/systemd/system/dhcpd.service.
Created symlink /etc/systemd/system/sockets.target.wants/tftp.socket → /usr/lib/systemd/system/tftp.socket.
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.

```

2. 在dhcp服务器中配置作用域为10.10.10.0/24，地址池范围：10.10.10.20-10.10.10.50，网关为10.10.10.254，DNS地址10.10.10.254
```bash
[root@server01 ~]# vim /etc/dhcp/dhcpd.conf
subnet 10.10.10.0 netmask 255.255.255.0 {
  range 10.10.10.20 10.10.10.50;
  option domain-name-servers 10.10.10.254;
  option domain-name "internal.example.org";
  option routers 10.10.10.254;
  option broadcast-address 10.10.10.255;
  default-lease-time 600;
  max-lease-time 7200;
  next-server 10.10.10.254;
  filename "pxelinux.0";
}
```

3. 将/dev/cdrom挂载至/var/www/html/pub目录上，确保每次开机均自动挂载
```bash
[root@server01 ~]# echo "/dev/cdrom /var/www/html/pub iso9660 defaults,ro 0 0" >> /etc/fstab
[root@server01 ~]# mount -a
```

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
```bash

```


5.  通过该服务器安装kvm-host主机，确保该服务器通过kvm-host.cfg安装。
```bash

```
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