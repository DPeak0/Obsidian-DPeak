# Linux使用telnet远程登陆

### 安装telnet服务端

```bash
[root@dns ~]# rpm -qa telnet-server xinetd		##查看是否安装
[root@dns ~]# yum install -y telnet-server xinetd	##安装telent服务端和xinetd
[root@dns ~]# systemctl restart telnet.socket xinetd.service
```

### 登录测试

>默认情况下root用户是被禁止了telnet登录的，需要进行以下操作方可登录。

```bash
[root@dns ~]# mv /etc/securetty /etc/securetty.bak
```

```bash
[root@www ~]# yum install -y telnet
[root@www ~]# telnet 192.168.10.103
Trying 192.168.10.103...
Connected to 192.168.10.103.
Escape character is '^]'.

Kernel 3.10.0-693.el7.x86_64 on an x86_64
dns login: root
Password:
Last login: Tue Mar 21 04:15:22 from ::ffff:192.168.10.253
[root@dns ~]#
```