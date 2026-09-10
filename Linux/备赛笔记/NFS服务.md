# Debian 搭建NFS服务

### NFS简介

客户端NFS和服务端NFS通讯过程

1、首先服务器端启动RPC服务，并开启111端口

2、启动NFS服务，并向RPC注册端口信息

3、客户端启动RPC（rpcbind服务），向服务端的RPC(rpcbind)服务请求服务端的NFS端口

4、服务端的RPC(rpcbind)服务反馈NFS端口信息给客户端。

5、客户端通过获取的NFS端口来建立和服务端的NFS连接并进行数据的传输

>ro：共享目录只读；
>rw：共享目录可读可写
>sync：同步，将数据同步写入内存缓冲区与磁盘中，效率低，但可以保证数据的一致性；
>async：异步，将数据先保存在内存缓冲区中，必要时才写入磁盘，效率高，但有丢失数据的风险；
>wdelay（默认）：如果有多个客户端要对同一个共享目录进行写操作，则将这些操作集中执行。对有很多小的IO写操作时，使用该选项可以有效的提高效率；
>no_wdelay：如果有多个客户端要对同一个共享目录进行写操作则立即写入。当设置了async选项时，no_wdelay选项无效，应与sync配合使用；
>root_squash（默认）：将来访的root用户映射为匿名用户或用户组；
>no_root_squash：来访的root用户保持root帐号权限；
>all_squash：所有访问用户都映射为匿名用户或用户组；
>no_all_squash（默认）：访问用户先与本机用户匹配，匹配失败后再映射为匿名用户或用户组；
>anonuid=<UID>：指定匿名访问用户的本地用户UID，默认为nfsnobody（65534）；
>anongid=<GID>：指定匿名访问用户的本地用户组GID，默认为nfsnobody（65534）；
>secure（默认）：限制客户端只能从小于1024的tcp/ip端口连接服务器；
>insecure：允许客户端从大于1024的tcp/ip端口连接服务器；
>subtree_check ：若输出目录是一个子目录，则nfs服务器将检查其父目录的权限；
>no_subtree_check（默认） ：即使输出目录是一个子目录，nfs服务器也不检查其父目录的权限，这样可以提高效率；
>hide：共享一个目录时，不共享该目录的子目录；
>no_hide：共享子目录；

## 服务端

### 1.安装软件包

```bash
root@debian:~# apt install -y nfs-kernel-server rpcbind 
```

### 2.创建共享目录

```bash
root@debian:~# mkdir -p /web/www
root@debian:~# chmod -R 777 /web/www
root@debian:~# echo "I LOVE XLHZ" > /web/www/hz
```

### 3.修改配置文件

```bash
root@debian:~# vim /etc/exports 
/web/www        192.168.100.10(rw)
```

### 4.重启服务

```bash
root@debian:~# systemctl restart nfs-server rpcbind
```

## 客户端

### 1.安装软件包

```bash
root@debian:~# apt install -y nfs-common
```

### 2.验证是否成功共享

```bash
root@debian:~# showmount -e 192.168.100.100
Export list for 192.168.100.100:
/web/www 192.168.100.10
```

### 3.添加自动挂载

```bash
root@debian:~# mkdir /web
root@debian:~# echo "192.168.100.100:/web/www /web nfs defaults 0 0" >> /etc/fstab 
root@debian:~# mount -a
root@debian:~# df
192.168.100.100:/web/www  16447488  6688256   8903680  43% /web
```

### 4.测试

```bash
root@debian:~# cat /web/hz
I LOVE XLHZ
```

