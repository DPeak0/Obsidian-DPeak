## CentOS

#### CentOS 开启路由转发

##### 1.临时开启

（写入内存，在内存中开启）

```bash
[root@localhost ~]# echo "1" > /proc/sys/net/ipv4/ip_forward
```

##### 2.永久开启

（写入内核）

```bash
[root@localhost ~]# sysctl -a | grep "ip_forward"
net.ipv4.ip_forward = 0
net.ipv4.ip_forward_use_pmtu = 0
[root@localhost ~]# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf 
[root@localhost ~]# sysctl -p
net.ipv4.ip_forward = 1
```
 常见失败原因：

> 1.电脑本身没有开启虚拟化支持，需要在重启时进入BIOS里设置。
>
> 2.配置nat转发
>
> iptables-t nat -F #清除原有的nat表中的规则
>
> iptables -F #清除原有的filter有中的规则
>
> iptables -P FORWARDACCEPT #缺省允许IP转发
>
> #利用iptables 实现nat MASQUERADE 共享上网
> iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
>
> #此处eth0 需要是能够访问外部网络的网卡接口
>
> 或者
>
> iptables -t nat-A POSTROUTING -s 192.168.121.0/24（内网充当其他同网段内网机器的网关） -j SNAT --to 192.168.159.128（公网ip）



#### 解决无法登录root问题

修改或移除securetty文件

```bash
#方法一：直接移除securetty文件
[root@dns ~]# mv /etc/securetty /etc/securetty.bak

#方法二：修改securetty文件
[root@dns ~]# vim /etc/securetty
pts/1
pts/2
pts/3
pts/4
pts/5
pts/6
pts/7
pts/8
pts/9
pts/10
pts/11
#在文件中追加pts/1~11
```

#### 挂载本地ISO文件配置本地yum源

```bash
[root@dns ~]# mkdir /dvd
[root@dns ~]# rm -rf /etc/yum.repos.d/*
[root@dns ~]# cat >/etc/yum.repos.d/dvd.repo<<END
>[dvd]
>name=dvd
>baseurl=file:///dvd
>gpgcheck=0
>END
[root@dns ~]# echo "/opt/CentOS-6.5.iso /dvd iso9660 defaults,ro,loop 0 0" >>/etc/fstab
[root@dns ~]# mount -a
[root@dns ~]# yum clean all;yum makecache ; yum repolist all
```

#### 生成指定大小文件

- 方法一：fallocate

```bash
[root@mail ~]# fallocate -l 6M test
```

- 方法二：truncate

```bash
[root@mail ~]# truncate -s 6M test
```



## Debian

> 命令使用手册	http://man.openbsd.org

### 搭建本地apt源

```bash
###连接iso镜像文件
root@debian:~# mkdir /dvd		##创建挂载目录
root@debian:~#vim /etc/fstab		##开启自动挂载
/dev/sr0 /dvd iso9660 defaults 0 0
root@debian:~# vim /etc/apt/sources.list	 ##添加apt源
deb [trusted=yes] file:/dvd/ buster main contrib
root@debian:~# apt update 		##更新apt源
```

### root登录问题

```bash
###先以其他用户（demo）登入系统
demo@debian:~$ su - root	##切换root登录
Password: 
root@debian:~# 
root@debian:~# vim /etc/gdm3/daemon.conf ##在[security]下添加
[security]
AllowRoot=true
root@debian:~# vim /etc/pam.d/gdm-password
###注释掉第三行
#auth   required        pam_succeed_if.so user != root quiet_success
```

### 查看服务进程及端口

#### ps

```bash
root@debian:~# ps -ef
```

#### netstat

```bash
root@debian:~# apt install -y net-tools 
root@debian:~# netstat -ntpl
```

### 添加开机自启命令

##### Debian

> 举例：永久静态路由
>
> 用法：在/etc/network/interfaces下插入一条
>
> up	[命令]

```bash
root@debian:~# vim /etc/network/interfaces
auto ens33
iface ens33 inet static
address 192.168.10.11/24
gateway	192.168.10.254
up ip route add 192.168.20.0/24 gw 81.6.63.114		##开机自动执行此条命令
```

##### Centos

```bash
[root@cs ~]# vim /etc/rc.local
```

