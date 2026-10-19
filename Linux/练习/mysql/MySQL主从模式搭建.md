### 初始化环境

| 服务器   | 公网 IP                  | 私网 IP               |
| ----- | ---------------------- | ------------------- |
| node1 | VMnet8： 192.168.200.10 | VMnet1：192.168.0.10 |
| node2 | 无                      | 192.168.0.20        |
| node3 | 无                      | 192.168.0.30        |
- 修改主机名、配置本地hosts、本地yum源
```bash
[root@node1 ~]# hostnamectl set-hostname node1
[root@node1 ~]# echo -e '192.168.0.10 node1\n192.168.0.20 node2\n192.168.0.30 node3' >> /etc/hosts
[root@node1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3

[root@node1 ~]# mkdir /etc/yum.repos.d/bak
[root@node1 ~]# mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak
[root@node1 ~]# cat > /etc/yum.repos.d/dvd.repo <<EOF
> [BaseOS]
> name=BaseOS
> baseurl=file:///media/BaseOS
> gpgcheck=0
>
> [AppStream]
> name=AppStream
> baseurl=file:///media/AppStream
> gpgcheck=0
> EOF
[root@node1 ~]# mount /dev/sr0 /media/
 yum makecachemount: /media: WARNING: device write-protected, mounted read-only.
[root@node1 ~]# yum clean all ; yum makecache
13 files removed
BaseOS                                                 37 MB/s | 2.5 MB     00:00
AppStream                                              31 MB/s | 7.1 MB     00:00
Last metadata expiration check: 0:00:01 ago on Mon 19 Oct 2026 08:17:06 PM CST.
Metadata cache created.
```

- 测试连通性
```bash
[root@node1 ~]# ping node2 -c 1
PING node2 (192.168.0.20) 56(84) bytes of data.
64 bytes from node2 (192.168.0.20): icmp_seq=1 ttl=64 time=0.410 ms

--- node2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.410/0.410/0.410/0.000 ms
[root@node1 ~]# ping node3 -c 1
PING node3 (192.168.0.30) 56(84) bytes of data.
64 bytes from node3 (192.168.0.30): icmp_seq=1 ttl=64 time=2.47 ms

--- node3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.466/2.466/2.466/0.000 ms
```

- 安装MySQL8.0
```bash
[root@node1 ~]# rpm -qa mysql-server
mysql-server-8.0.21-1.module+el8.3.0+242+87d3366a.x86_64
[root@node1 ~]# systemctl enable mysqld --now
```

### 配置主库Master
```bash
[root@node1 ~]# vim /etc/my.cnf.d/replication.cnf
[root@node1 ~]# cat /etc/my.cnf.d/replication.cnf
[mysqld]
# 监听所有网卡接口
bind-address = 0.0.0.0
# 集群内唯一节点 ID
server-id = 1
# 开启二进制日志
log-bin = mysql-bin
# Binlog 格式使用 ROW（数据一致性最高，最佳实践）
binlog_format = ROW
```
- 重启MySQL使配置生效
```bash
[root@node1 ~]# systemctl restart mysqld.service
```


