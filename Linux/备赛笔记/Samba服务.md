# Debina 搭建Samba服务

> 原文链接：https://blog.csdn.net/misisippi68/article/details/105745520

##### 配置文件说明

samba定义的变量：
%S = 当前服务名（如果有的话）
%P = 当前服务的根目录（如果有的话）
%u = 当前服务的用户名（如果有的话）
%g = 当前用户说在的主工作组
%U = 当前对话的用户名
%G = 当前对话的用户的主工作组
%H = 当前服务的用户的Home目录
%v = Samba服务的版本号。
%h = 运行Samba服务机器的主机名
%m = 客户机的NETBIOS名称
%L = 服务器的NETBIOS名称
%M = 客户机的主机名
%N = NIS服务器名
%p = NIS服务的Home目录
%R = 说采用的协议等级(值可以是CORE, COREPLUS, LANMAN1, LANMAN2，NT1)
%d = 当前服务进程的ID
%a = 客户机的结构（只能识别几项：Samba，WfWg，WinNT，Win95）
%I = 客户机的IP
%T = 当前日期和时间

**[共享名称]**
这个共享名称很重要，它是一个代号而已，用户在“网上邻居”中所看到的共享目录名
[global]全局配置
**hosts allow = 192.168.10.0/24**
#允许192.168.10.0/24网段访问，ALL为全部
**hosts deny = 192.168.10.0/24**
#禁止192.168.10.0/24网段访问，ALL为全部
**map to guest = Bad User **
#无密码登录。将所有samba系统主机所不能正确识别的用户都映射成guest（匿名）用户
**guest account = nobody**
用来设置Samba中guest用户的系统用户名。
[homes] 共享目录
[printers] 共享打印机
**comment = 任意字符串**
说明：comment是对该共享的描述，可以是任意字符串。
**path = 共享目录路径**
说明：path用来指定共享目录的路径。可以用%u、%m这样的宏来代替路径里的unix用户和客户机的Netbios名，用宏表示主要用于[homes]共享域。例如：如果我们不打算用home段做为客户的共享，而是在/home/share/下为每个Linux用户以他的用户名建个目录，作为他的共享目录，这样path就可以写成：**path = /home/share/%u; **。用户在连接到这共享时具体的路径会被他的用户名代替，要注意这个用户名路径一定要存在，否则，客户机在访问时会找不到网络路径。同样，如果我们不是以用户来划分目录，而是以客户机来划分目录，为网络上每台可以访问samba的机器都各自建个以它的netbios名的路径，作为不同机器的共享资源，就可以这样写：path = /home/share/%m 。
**browseable = yes/no**
说明：browseable用来指定该共享是否在“网上邻居”中可见。
**writable = yes/no**
说明：writable用来指定该共享路径是否可写。
**read only = yes/no**
说明：设置共享目录为只读，这个选项和writable是互斥的，最好不要两个同时出现如果同时出现，那么最后出现的那个设置为主要的设置。
**available = yes/no**
说明：available用来指定该共享资源是否可用。
**admin users = 该共享的管理者**
说明：admin users用来指定该共享的管理员（对该共享具有完全控制权限）。在samba 3.0中，如果用户验证方式设置成“security=share”时，此项无效。
例如：admin users =bobyuan，jane（多个用户中间用逗号隔开）。
**valid users = 用户或@组**
说明：valid users用来指定允许访问该共享资源的用户。
例如：valid users = bobyuan，@bob，@tech（多个用户或者组中间用逗号隔开，如果要加入一个组就用“@+组名”表示。）
**invalid users = 用户或@组**
说明：invalid users用来指定不允许访问该共享资源的用户。
例如：invalid users = root，@bob（多个用户或者组中间用逗号隔开。）
**write list = 用户或@组**
说明：write list用来指定可以在该共享下写入文件的用户。
例如：write list = bobyuan，@bob
**read list = 用户或@组**
说明：read list用来指定可以在该共享下读取文件的用户。
例如：read list = bobyuan，@bob
**public = yes/no**
"public = yes"选项是指这个共享对所有人都是可见的
**guest ok = yes/no**
说明：匿名模式，允许guest账户访问
"guest ok = yes"选项是指允许未经身份验证的用户作为匿名用户访问共享。
**create mask = 0666**
说明：指定用户通过Samba在该共享目录中创建文件的默认权限。0600代表创建文件的权限为rw-rw-rw-
**directory mask = 0777**
指定用户通过Samba在该共享目录中创建目录的默认权限。0600代表创建目录的权限为drwxrwxrwx

**veto files = /\*.exe/\*.jpg*/**
不允许在目录指定关键字的文件或目录


### 特殊权限


```bash
root@debian:~# chmod o+t /samba 	##只有所以者和root可以删除文件
root@debian:~# chown zhangsan /samba	##目录所有者不受sbit影响
root@debian:~# chmod g+s /samba 	##新建文件为该目录所属组
```


<hr>

## 服务端


### 一、匿名登录

#### 1.安装samba

```bash
root@debian:~# apt install -y samba
```

#### 2.创建共享目录

```bash
root@debian:~# mkdir -p /samba/guest		##创建共享目录
```

#### 3.修改配置文件

```bash
root@debian:~# vim /etc/samba/smb.conf

[guest]
        path = /samba/guest		##指定共享目录
        read only = yes		##是否只读
        guest ok = yes		##是否允许匿名登录
```

#### 4.重启服务

```bash
root@debian:~# systemctl restart smbd.service nmbd.service 
```

## 二、用户认证登录

#### 1.创建共享目录和用户

```bash
root@debian:~# mkdir -p /samba/dbw		##创建共享目录
root@debian:~# useradd dbw -s /sbin/nologin		##创建用户
root@debian:~# echo dbw:123 | chpasswd		##设置用户密码
root@debian:~# smbpasswd -a dbw		##添加samba用户
root@debian:~# pdbedit -L		##查看samba已添加用户
dbw:1001:
```

#### 2.修改配置文件

```bash
root@debian:~# vim /etc/samba/smb.conf
[dbw]
        path = /samba/dbw		##指定共享目录
        writable = yes		##可写
        guest ok = no		##是否允许匿名登录
```

#### 3.重启服务

```bash
root@debian:~# systemctl restart smbd.service nmbd.service 
```



## 客户端

### 一 . 匿名登录测试

安装samba客户端

```bash
root@debian:~# apt install -y smbclient 
```

查看samba服务器共享目录

```bash
root@debian:~# smbclient -L 192.168.100.100
Enter WORKGROUP\root's password: 
        Sharename       Type      Comment
        ---------       ----      -------
        guest           Disk      dbw guest
```

进入目录

```bash
root@debian:~# smbclient //192.168.100.100/guest 
Enter WORKGROUP\root's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Oct  5 02:55:13 2022
  ..                                  D        0  Wed Oct  5 02:47:33 2022
  Dbw.txt                             A        0  Wed Oct  5 02:54:50 2022
                16447356 blocks of size 1024. 8825916 blocks available
```

### 二 . 用户认证登录测试

挂载到本地

```bash
root@debian:~# mkdir /samba		##创建挂载目录
root@debian:~# echo "//192.168.100.100/dbw /samba cifs username=dbw,password=123" >> /etc/fstab		##开启开机自动挂载
root@debian:~# mount -a		##挂载
root@debian:~# df | grep dbw	##查看是否挂载
//192.168.100.100/dbw  16447356  7621548   8825808  47% /samba
```

  进入目录

```bash
root@debian:~# cd /samba
root@debian:/samba# ls
'I LOVE XLHZ!.txt'
```

