# NTP/Chrony时间同步服务

## Chrony

### 服务端

### 1.安装软件包

```bash
root@debian:~# apt install -y chrony 
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/chrony/chrony.conf 
#pool 2.debian.pool.ntp.org iburst
server 192.168.100.100 iburst		##设置时间同步服务器
local stratum 10		##当server指定的时间同步服务器不可用时，使用本地时间进行同步
allow 192.168.100.0/24		##允许同步网段
```

### 3.重启服务

```bash
root@debian:~# systemctl restart chrony
```

### 4.开启NTP时间同步

```bash
root@debian:~# timedatectl set-ntp true 
```

### 5.查看同步的客户端

```bash
root@debian:~# chronyc clients 
Hostname                      NTP   Drop Int IntL Last     Cmd   Drop Int  Last
===============================================================================
debian                         14      0   6   -    36       0      0   -     -
192.168.100.30
```

### 客户端

### 1.安装软件包

```bash
root@debian:~# apt install -y chrony 
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/chrony/chrony.conf
#pool 2.debian.pool.ntp.org iburst
server 192.168.100.100 iburst
```

### 3.重启服务

```bash
root@debian:~# systemctl restart chrony
```

### 4.开启NTP时间同步

```bash
root@debian:~# timedatectl set-ntp true 
```

### 5.查看连接的时间同步服务器

```bash
root@debian:~# chronyc sources -v
210 Number of sources = 1

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^? 192.168.100.100              11   6     3     0    -22us[  -22us] +/- 1882us
```

### 4.设置自动同步

```bash
root@debian:~# apt install -y ntpdate
root@debian:~# systemctl stop ntp		##停止ntp后 再同步否则会冲突报错
root@debian:~# ntpdate 192.168.100.100
11 Oct 01:42:49 ntpdate[4586]: adjust time server 192.168.100.100 offset -0.009547 sec
root@debian:~# vim /etc/crontab 
*/5 * * * * root ntpdate 192.168.100.100		###每五分钟同步一次
```

## NTP

### 服务端

#### 安装软件包

```bash
root@lnxrtr1:~# apt install -y ntp
```

#### 修改配置文件

- /etc/ntp.conf 

```bash
root@lnxrtr1:~# vim /etc/ntp.conf 
server 127.127.1.0 fudge		##设置时间同步服务器
127.127.1.0 stratum 10			##当server指定的时间同步服务器不可用时，使用本地时间进行同步
allow 192.168.10.0/24			##允许同步网段
```

#### 重启服务

```bash
root@lnxrtr1:~# systemctl enable ntp
root@lnxrtr1:~# systemctl restart ntp
```

#### 查看

```bash
root@lnxrtr1:~# ntpq -p
```

### 客户端

```bash
root@lnxclt1:~# apt install -y ntpdate 
root@lnxclt1:~# ntpdate 192.168.10.254
19 Oct 16:48:13 ntpdate[3050]: step time server 192.168.10.254 offset -0.588908 sec
```