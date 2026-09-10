# vsftpdf配置详解

`环境Centos 7`

### 服务简介

FTP是仅基于TCP的服务，不支持UDP。 FTP传输使用2个端口，一个数据端口和一个命令端口。通常来说这两个端口是21（命令端口）和20（数据端口）。FTP存在俩种模式：主动模式和被动模式。

- **PORT（主动）模式**
   所谓主动模式，指的是FTP服务器“主动”去连接客户端的数据端口来传输数据，其过程具体来说就是：客户端通过访问服务端的21端口，然后客户端分配一个端口供ftp服务端获取数据，然后服务端通过20端口主动到客户端指定端口获取数据，20为服务器的出向端口。

  > 此模式，防火墙只需要开放21端口的对外访问策略

- **PASV（被动）模式**
所谓被动模式，指的是FTP服务器“被动”等待客户端来连接自己的数据端口，其过程具体是：客户端通过访问服务端的21端口，然后客户端提交PASV命令，让服务端分配一个用于传输数据的端口，此端口范围为vsftpd.conf配置的pasv_min_port-pasv_max_port，然后客户端通过分配的端口上传数据。（注意此模式下的FTP服务器不需要开启tcp 20端口）

  > 此模式，防火墙需要开放21端口的对外访问策略和pasv_min_port - pasv_max_port端口范围内的访问策略

#### 匿名用户

#### 配置文件

```bash
anonymous_enable=YES		##是否允许匿名用户登录
local_enable=YES		##是否允许登录本地用户
write_enable=YES		##是否允许写入
anon_umask=022		##匿名用户上传权限：022新建文件权限为644 目录权限为755；077新建文件权限为600，目录权限为700 
anon_root=/ftp		##指定匿名用户主目录
chroot_local_user=yes		##是否将用户限制在用户主目录
allow_writeable_chroot=YES		##如果限制在用户主目录允许写入根路径
anon_upload_enable=YES			##允许匿名用户上传文件夹
anon_mkdir_write_enable=YES		##允许匿名用户登录后创建文件夹
anon_other_write_enable=YES		##允许匿名用户修改名字，删除文件夹等操作
anon_world_readable_only=no		##允许下载
```

### 本地用户

#### 配置文件

```bash
anonymous_enable=NO		##是否允许匿名用户登录
local_enable=YES		##是否允许登录本地用户
write_enable=YES		##是否允许写入
local_umask=022		##本地用户上传权限：022新建文件权限为644 目录权限为755；077新建文件权限为600，目录权限为700 
local_root=/ftp		##指定虚拟用户主目录
chroot_local_user=yes		##是否将用户限制在用户主目录
allow_writeable_chroot=YES		##如果限制在用户主目录允许写入根路径
```

### 虚拟用户

#### 配置文件

```bash
guest_enable=YES #设定启用虚拟用户功能。
guest_username=vsftpd #指定虚拟用户的宿主用户
user_config_dir=/etc/vsftpd/guestuser #虚拟用户配置文件目录，文件名必须和虚拟用户名相同
virtual_use_local_privs=YES     #当该参数激活（YES）时，虚拟用户使用与本地用户相同的权限，当此参数关闭（NO）时，虚拟用户使用与匿名用户相同的权限。默认情况下此参数是关闭的（NO）。
chroot_local_user=yes		##是否将用户限制在用户主目录
allow_writeable_chroot=YES		##如果限制在用户主目录允许写入根路径
pam_service_name=vsftpd			##PAM文件路径
```

#### 安装Berkeley DB工具
```bash
# （linux centos 7 貌似可以省略这一步，不需要安装） 
[root@ftp ~]yum install db4 db4-utils -y
```
#### 创建用于验证vsftpd的数据文件

```bash
[root@ftp vsftpd]# vim /etc/vsftpd/ftp_user.txt
ftpuser1
123456
```
#### 加密普通文件
```bash
# -T -t 指定hash加密类型，-f 指定一个原文件，生成一个db类型。
[root@ftp ~]# db_load -T -t hash -f /etc/vsftpd/ftp_user.txt /etc/vsftpd/ftp_user.db
[root@ftp ~]# file ftp_user.txt
ftp_user.txt: ASCII text # 一个ASCII文本
[root@ftp ~]# file ftp_user.db
ftp_user.db: Berkeley DB (Hash, version 9, native byte-order) # 被加密过的文件
[root@ftp ~]# rm -rf ftp_user.txt # 加密后可以删除旧的数据文本可以保证安全
性
```
#### 降低文件的权限

```bash
[root@ftp ~]# chmod 600 ftp_user.db
[root@ftp ~]# ll | grep ftp_user.db
-rw------- 1 root root 12288 10月 10 22:32 ftp_user.db
```
#### 创建映射用户
```bash
# -d 指定用户的家目录，-s /sbin/nologin 禁止登录当前服务器
[root@ftp ~]# useradd -d /var/ftpdir -s /sbin/nologin vsftpd
```
### 修改PAM文件
```bash
[root@ftp ~]# vim /etc/pam.d/vsftpd
# 添加如下内容，注意需要注释掉之前文件之前的内容，仅仅添加下面两行参数 
auth required pam_userdb.so db=/etc/vsftpd/ftp_user 
account required pam_userdb.so db=/etc/vsftpd/ftp_user
```

#### 虚拟用户配置文件

```bash
[root@ftp ~]# vim /etc/vsftpd/guestuser/ftpuser1
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES 
anon_world_readable_only=no #可下载
```

### 其他配置

```bash
max_clients=50  	##最大同时连接数
max_per_ip=5		##单个ip最大连接数
pasv_enable=YES		##开启被动模式
pasv_min_port=30000	##被动模式端口范围最小值
pasv_max_port=40000 ##被动模式端口范围最大值
```

### FTP 数字代码的意义
110 新文件指示器上的重启标记
120 服务器准备就绪的时间（分钟数）
125 打开数据连接，开始传输
150 打开连接
200 成功
202 命令没有执行
211 系统状态回复
212 目录状态回复
213 文件状态回复
214 帮助信息回复
215 系统类型回复
220 服务就绪
221 退出网络
225 打开数据连接
226 结束数据连接
227 进入被动模式（IP 地址、ID 端口）
230 登录因特网
250 文件行为完成
257 路径名建立
331 要求密码
332 要求账号
350 文件行为暂停
421 服务关闭
425 无法打开数据连接
426 结束连接
450 文件不可用
451 遇到本地错误
452 磁盘空间不足
500 无效命令
501 错误参数
502 命令没有执行
503 错误指令序列
504 无效命令参数
530 未登录网络
532 存储文件需要账号
550 文件不可用
551 不知道的页类型
552 超过存储分配
553 文件名不允许