# DNS主从

### 1.安装软件包

```bash
root@debian:~# apt install -y bind9
```

### 2.复制配置文件

```bash
root@debian:~# cp -a /etc/bind/db.local /etc/bind/skills.com
root@debian:~# cp -a /etc/bind/db.127 /etc/bind/192.db
```

### 3.修改配置文件

#### - Master

```bash
root@debian:~# vim /etc/bind/named.conf.default-zones 
zone "skills.com" {
        type master;
        Allow-update {192.168.100.20;};
        file "/etc/bind/skills.com";
};

zone "192.168.100.in-addr.arpa" {
        type master;
		Allow-update {192.168.100.20;};
        file "/etc/bind/192.db";
};

root@debian:~# vim /etc/bind/skills.com 
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     skills.com. root.skills.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      skills.com.
@       IN      A       192.168.100.10
@       IN      A       192.168.100.20
ftp     IN      A       192.168.100.10
www     IN      A       192.168.100.10

root@debian:~# vim /etc/bind/192.db
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     skills.com. root.skills.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      skills.com.
10      IN      PTR     www.skills.com.
10      IN      PTR     ftp.skills.com.
```

- **重启服务**

```bash
root@debian:~# systemctl restart bind9
```

#### - Slave

```bash
root@debian:~# vim /etc/bind/named.conf.default-zones 
zone "skills.com" {
        type slave;
        master {192.168.100.10;};
        file "/etc/bind/skills.com";
};

zone "192.168.100.in-addr.arpa" {
        type slave;
		master {192.168.100.10;};
        file "/etc/bind/192.db";
};

root@debian:~# vim /etc/bind/skills.com 
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     skills.com. root.skills.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      skills.com.
@       IN      A       192.168.100.20
@       IN      A       192.168.100.10
www     IN      A       192.168.100.10
ftp     IN      A       192.168.100.10

root@debian:~# vim /etc/bind/192.db
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     skills.com. root.skills.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      skills.com.
10      IN      PTR     www.skills.com.
10      IN      PTR     ftp.skills.com.
```


- **重启服务**

```bash
root@debian:~# systemctl restart bind9
```

### 4.添加DNS解析

```bash
root@debian:~# vim /etc/resolv.conf 
nameserver 192.168.100.10
nameserver 192.168.100.20
```

### 5.DNS解析测试

```bash
root@debian:~# apt install -y dnsutils 	##安装测试软件包
root@debian:~# nslookup www.skills.com
Server:         192.168.100.10
Address:        192.168.100.10#53

Name:   www.skills.com
Address: 192.168.100.10

root@debian:~# nslookup ftp.skills.com
Server:         192.168.100.10
Address:        192.168.100.10#53

Name:   ftp.skills.com
Address: 192.168.100.10
```

### 禁止某网段访问

```bash
blackhole { 10.0.0.1; };
```

### 关于DNS的常用命令

#### 1.启动、停止、重启与自启DNS服务

查看进程状态： systemctl status named
重启dns服务器：systemctl start named
关闭dns服务器： systemctl stop named
重启dns服务器： systemctl restart named
重新加载dns服务器：systemctl reload named
dns开机自动启动：systemctl enable named
取消dns开机自动启动：systemctl disable named

#### 2.DNS服务故障排除

nslookup ：测试域名解析情况
netstat -an | grep 53： 检查TCP或者UDP的53号端口情况
named-checkconf -z /etc/named.conf： 检查配置文件是否错误