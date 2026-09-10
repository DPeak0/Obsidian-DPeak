# Debian 配置SSH远程免密登录

> **ssh_config 为客户端配置文件	sshd_config 为服务端配置文件**
>
> 配置文件说明	http://t.csdn.cn/GNyuL

**Port 22**
		SSH 预设使用 22 这个 port，您也可以使用多的 port ！

**PermitRootLogin no**
		是否允许 root 用户直接登录，如果想root直接登录设置为yes，安全方面的考虑最好设置成no

**PasswordAuthentication yes**
		是否允许使用密码的认证登录

**PubkeyAuthentication yes**
		是否允许使用密钥的认证登录

**AllowUsers** user

​		允许登录的用户

**AllowGroups** group

​		允许登录的用户的用户组

**DenyUsers** user

​		拒绝登录的用户

**DenyGroups** group

​		拒绝登录的用户的用户组

<hr>

## 服务端

### 1.安装ssh服务端

```bash
root@debian:~# apt install -y openssh-server
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/ssh/sshd_config
Port 2222	##修改端口号
PermitRootLogin yes		##允许直接使用root登录
```

### 3.重启服务

```bash
root@debian:~# systemctl restart sshd.service
```

## 客户端

### 1.安装SSH客户端

```bash
root@debian:~# apt install -y openssh-client
```

#### 密码登录测试

```bash
root@debian:~# ssh root@192.168.100.100 -p 2222
root@192.168.100.100's password: 
Linux debian 4.19.0-11-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64
*************
holle world
*************
Last login: Wed Oct  5 09:07:08 2022 from 192.168.100.253
root@debian:~# 
```
### 2.免密登录配置

```bash
root@debian:~# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:KWU0JgvpA6AECus+c/1Dgi+H0S9Lvn8KpRchE0bCTyk root@debian
The key's randomart image is:
+---[RSA 2048]----+
|=o..o=o +        |
|=..E++.= .       |
|+  o+o..o        |
|.   o.oo..       |
| .  o..oS        |
|.  o.ooo.        |
| + .+=+.         |
|  +oo++o .       |
|    o+=++        |
+----[SHA256]-----+
root@debian:~# 
root@debian:~# ssh-copy-id root@192.168.100.100 -p 2222
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.100.100's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh -p '2222' 'root@192.168.100.100'"
and check to make sure that only the key(s) you wanted were added.

root@debian:~# 
```

#### 免密登录测试

```bash
root@debian:~# ssh root@192.168.100.100 -p 2222
Linux debian 4.19.0-11-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64
*************
holle world
*************
Last login: Wed Oct  5 09:19:09 2022 from 192.168.100.111
root@debian:~# 
```

其他命令

```bash
root@debian:~# ssh-keygen -R 192.168.100.100
```

## 扩展：限制指定IP登录

### 1.修改配置文件

```bash
root@debian:~# vim /etc/hosts.deny		
sshd:ALL except 192.168.100.111:deny	##只允许指定ip登录
```

### 2.重启服务

```bash
root@debian:~# systemctl restart sshd.service
```

### 登录测试

IP：192.168.100.112

```bash
root@debian:~# ssh root@192.168.100.100 -p 2222
ssh_exchange_identification: read: Connection reset by peer
```

IP：192.168.100.111

```bash
root@debian:~# ssh root@192.168.100.100 -p 2222
Linux debian 4.19.0-11-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64
*************
holle world
*************
Last login: Wed Oct  5 09:43:21 2022 from 192.168.100.111
root@debian:~# 
```

