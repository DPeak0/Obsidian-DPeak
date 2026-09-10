# Debian 搭建邮件服务postfix+dovecot

邮件发送需要涉及到两个协议，一个是[SMTP](https://so.csdn.net/so/search?q=SMTP&spm=1001.2101.3001.7020)协议，用来发送邮件；另一个是IMASPS协议，用来接收邮件。在Linux上可以使用postfix来搭建SMTP服务器，dovecot搭建imaps服务器。安装了这两个服务器程序就可以收发邮件了。

| 主机名 | ip              | 角色                                 |
| ------ | --------------- | ------------------------------------ |
| dns    | 192.168.100.100 | DNS服务端                            |
| mail   | 192.168.100.10  | Postfix邮件发送器、Dovecot邮件接收器 |
| ca     | 192.168.100.20  | CA服务端                             |

## dns

### 搭建DNS服务

```bash
root@dns:~# apt install -y bind9
```
#### 修改配置文件

```bash
root@dns:~# vim /etc/bind/named.conf.default-zones 
zone "dbw.com" {
	type master;
	file "/etc/bind/dbw.com";
};

zone "100.168.192.in-addr.arpa" {
	type master;
	file "/etc/bind/192.db";
```
#### 添加正向及反向解析文件

```bash
root@dns:~# cp -a /etc/bind/db.local  /etc/bind/dbw.com
root@dns:~# cp -a /etc/bind/db.127 /etc/bind/192.db
```

- 修改正向解析文件

```bash
root@dns:~# vim /etc/bind/dbw.com 
;
; BIND data file for local loopback interface
;
$TTL	604800
@	IN	SOA	dbw.com. root.localhost. (
			      2		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	dns.dbw.com.
dns	IN	A	192.168.100.100
@	IN	MX 5	mail.dbw.com.
mail	IN	A	192.168.100.10
```

- 修改反向解析文件

```bash
root@dns:~# vim /etc/bind/192.db 
;
; BIND reverse data file for local loopback interface
;
$TTL	604800
@	IN	SOA	dbw.com. root.localhost. (
			      1		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	dns.dbw.com.
100	IN	PTR	dns.dbw.com.
10	IN	PTR	mail.dbw.com.
```

#### 重启服务

```bash
root@debian:~# systemctl restart bind9
```

## ca

### 证书

#### 安装openssl并修改配置文件

```bash
root@ca:~# apt install -y openssl
root@ca:~# vim /etc/ssl/openssl.cnf 
48	dir		= /CA		# Where everything is kept
```

#### 创建目录

```bash
root@ca:~# mkdir /CA
root@ca:~# cp -a /etc/ssl/* /CA
root@ca:~# cd /CA
```

#### 创建跟密钥和根证书

```bash
root@ca:~# openssl genrsa -out cakey.pem
root@ca:~# openssl req -x509 -new -key cakey.pem  -out cacrs.pem
```

#### 颁发证书

```bash
root@ca:/CA# openssl x509  -req  -in mail.csr -CA cacrs.pem -CAkey cakey.pem  -CAcreateserial -out mail.crt
Signature ok
subject=C = CN, ST = Some-State, O = dbw, OU = dbw, CN = *.dbw.com
Getting CA Private Key
root@ca:/CA# scp /CA/mail.crt  192.168.100.10:/CA	##发送颁布的证书
```

## mail

### 证书

#### 安装openssl并修改配置文件

```bash
root@mail:~# apt install -y openssl
root@mail:~# vim /etc/ssl/openssl.cnf	##修改48行目录位置
48	dir		= /CA		# Where everything is kept
```

#### 创建目录

```bash
root@mail:~# mkdir /CA
root@mail:~# cp -a /etc/ssl/* /CA
root@mail:~# cd /CA
```

#### 申请证书

```bash
root@mail:/CA# openssl genrsa -out  mailkey.key
root@mail:/CA# openssl req  -new -key mailkey.key  -out mail.csr
root@mail:/CA# scp /CA/mail.csr 192.168.100.20:/CA		###将申请传到ca服务器上，注意先在另一边建立文件夹
root@mail:/CA# ls 	##颁发后查看
certs  mail.crt  mail.csr  mailkey.key	openssl.cnf  private
```

### 安装postfix

```bash
root@mail:~# apt install -y postfix
```

#### 修改配置文件main.cf

```bash
root@mail:/CA# vim /etc/postfix/main.cf
    26	# TLS parameters
    27	smtpd_tls_cert_file=/CA/mail.crt		##修改为证书的位置
    28	smtpd_tls_key_file=/CA/mailkey.key		##修改为密钥的位置

    37	myhostname = mail.dbw.com		##修改为mail.dbw.com

    40	myorigin = dbw.com		##修改为dbw.com
    41	mydestination = $myhostname, dbw.com, mail, mail.dbw.com,localhost.localdomain, localhost		##添加mail.dbw.com

    43	mynetworks = 0.0.0.0/0		##修改为0.0.0.0/0

    49	home_mailbox = Maildir/		##在最后一行添加
```

#### 修改配置文件master

```bash
root@mail:~# cat /etc/postfix/master.cf
    12	#smtp      inet  n       -       y       -       -       smtpd		##注释
    29	smtps     inet  n       -       y       -       -       smtpd		##取消注释
    31	  -o smtpd_tls_wrappermode=yes		##取消注释
```

### 安装dovecot

```bash
root@mail:~# apt install -y dovecot-imapd 
```

#### 修改配置文件dovecot.conf

```bash
root@mail:~# vim /etc/dovecot/dovecot.conf 
    30	listen = *		##取消#注释并修改
    48	login_trusted_networks = 0.0.0.0/0	##取消#注释并修改
    49	protocols = imaps		##添加此条
```

#### 修改配置文件10-mail.conf

```bash
root@mail:~# vim /etc/dovecot/conf.d/10-mail.conf
    24	   mail_location = maildir:~/Maildir		##取消注释
    30	#mail_location = mbox:~/mail:INBOX=/var/mail/%u		##注释此条
```

#### 修改配置文件10-ssl.conf

```bash
root@mail:~# vim /etc/dovecot/conf.d/10-ssl.conf
	12	ssl_cert = </CA/mail.crt
    13	ssl_key = </CA/mailkey.key
```

#### 链接证书文件

```bash
root@mail:~# cd /etc/dovecot/private/
root@mail:/etc/dovecot/private# ln -s /CA/mail.crt ./
root@mail:/etc/dovecot/private# ln -s /CA/mailkey.key ./
```

### 批量新建用户

```bash
root@mail:~# vim user.sh 
#!/bin/bash
for i in {1..99}
do
	useradd -m -s /bin/false "user$i"
	echo "user$i:Chinaskills20!" | chpasswd
done
root@mail:~# bash user.sh 	##执行脚本
```

## 客户端测试

### thunderbird雷鸟

```bash
root@dns:~# apt install -y thunderbird
```

![image-20221012143159928](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143159928.png)

![image-20221012143220778](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143220778.png)

![image-20221012143313589](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143313589.png)

![image-20221012143340023](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143340023.png)

![image-20221012143356211](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143356211.png)

![image-20221012143420615](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143420615.png)

![image-20221012143548961](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143548961.png)

![image-20221012143818766](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012143818766.png)

#### 继续添加一个用户

![image-20221012144300326](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012144300326.png)

![image-20221012145445531](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012145445531.png)

- 发送邮件

![image-20221012145652554](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012145652554.png)

![image-20221012145736349](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20221012145736349.png)