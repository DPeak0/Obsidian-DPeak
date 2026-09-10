# Debian 搭建Squid服务

## Squid服务简介

​		Squid cache（简称为Squid）是流行最广的，使用最普遍的开源缓存代理服务器。是用来缓存Internet数据的软件，接受来自人们需要下载的目标的请求并适当的处理这些请求。
​		也就是说，如果客户端向要下载一web界面，他请求squid为他取得这个界面。squid随之连接到远程服务器并向这个页面发出请求，然后squid显示的聚集数据到客户端机器，而且同时复制一份。当下一次有人需要同一界面时，squid可以简单的从磁盘中读到它，那样数据会立即传输到客户机上。

- **代理的工作机制**

  - 代替客户机向网站请求数据，从而可以隐藏用户的真实IP地址，从而起到一定的保护作用。

  - 将获得的网页数据（静态 Web 元素）保存到缓存中并发送给客户机，以便下次请求相同的数据时快速响应。

  - 提供缓存加速、应用层过滤控制的功能。

  - 主要支持http、ftp等应用协议

- **Squid 代理的类型**
  - 传统代理：适用于Internet，需在客户机指定代理服务器的地址和端口。
  - 透明代理：客户机不需指定代理服务器的地址和端口，而是通过默认路由、防火墙策略将Web访问重定向给代理服务器处理。
  - 反向代理：如果 Squid 反向代理服务器中缓存了该请求的资源，则将该请求的资源直接返回给客户端；否则反向代理服务器将向后台的 WEB 服务器请求资源，然后将请求的应答返回给客户端，同时也将该应答缓存在本地，供下一个请求者使用。

<hr>

## 路由功能

### 1.安装squid软件包

```bash
root@debian:~# apt install -y squid
```

### 2.修改配置文件

```bash
root@debian:~# head -n 1412 /etc/squid/squid.conf | tail -n 5
http_access allow all	##修改为all
#http_access deny all	##注释此条
```

### 2.开启路由转发功能

```bash
###临时修改
echo "1" > /proc/sys/net/ipv4/ip_forward

###永久修改
root@debian:~# vim /etc/sysctl.conf 
net.ipv4.ip_forward=1
root@debian:~# sysctl -p
```

### 3.重启服务

```bash
root@debian:~# systemctl restart squid.service
```
### 4.测试

squid默认监听的是3128端口

![image-20221010135245748](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221010135245748.png)

![image-20221010135924108](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221010135924108.png)

## 禁止用户访问指定网址

### 1.修改配置文件

```bash
root@debian:~# vim /etc/squid/squid.conf
root@debian:~# head -n 1412 /etc/squid/squid.conf | tail -n 2
acl apa src 192.168.200.20/24
http_access deny apa
```

### 2.测试

![image-20221010140621504](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221010140621504.png)