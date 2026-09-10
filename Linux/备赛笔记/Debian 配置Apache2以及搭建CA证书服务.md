# Debian  搭建Apache2以及配置SSL证书服务

## Apache2服务端

### 1.安装软件包

```bash
root@debian:~# apt install -y apache2
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/apache2/ports.conf 	##修改端口
Listen 80 443
root@debian:~# cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf.bak		##备份
root@debian:~# cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default2-ssl.conf
root@debian:~# vim /etc/apache2/sites-available/default-ssl.conf
ServerName www.skills.com		##域名
DocumentRoot /web/www	##网页根目录
<Directory /web/www>
Options Indexes FollowSymLinks
AllowOverride None
Require all granted
</Directory>		
SSLCertificateFile      /CA/apache2.crt		##CA证书
SSLCertificateKeyFile /CA/apache2.key		##CA密钥
root@debian:~# vim /etc/apache2/sites-available/default2-ssl.conf
ServerName ftp.skills.com		##域名
DocumentRoot /web/ftp	##网页根目录
<Directory /web/ftp>
Options Indexes FollowSymLinks
AllowOverride None
Require all granted
</Directory>
SSLCertificateFile      /CA/apache2.crt		##CA证书
SSLCertificateKeyFile /CA/apache2.key		##CA密钥
```

### 3.创建目录与网页文件

```bash
root@debian:~# mkdir -p /web/{www,ftp}		##创建网页目录
root@debian:~# echo "I LOVE XLHZ" > /web/www/index.html
root@debian:~# echo "this is FTP web" > /web/ftp/index.html
```

### 4.启用SSL和网页模板

```bash
root@debian:~# a2enmod ssl 
root@debian:~# a2dissite 000-default.conf ##禁用默认模板
root@debian:~# a2ensite default-ssl.conf	##启用模板
root@debian:~# a2ensite default2-ssl.conf
```
### 5.修改默认主页文件

```bash
root@debian:~# vim /etc/apache2/mods-available/dir.conf 
<IfModule mod_dir.c>
        DirectoryIndex index.html index.cgi index.pl index.php index.xhtml index.htm
</IfModule>
```

### 6.重启服务

```bash
root@debian:~# systemctl restart apache2.service 
```
## openssl服务端

### 1.安装openssl软件包

```bash
root@debian:~# apt install -y openssl
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/ssl/openssl.cnf 
dir = /CA  
```

### 3.创建目录

```bash
root@debian:~# mkdir /CA
root@debian:~# cp -a /etc/ssl/* /CA
root@debian:~# cd /CA
```

### 4.配置证书

![image-20230614001550514](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001550514.png)

- 创建根证书

```bash
root@debian:/CA# openssl genrsa -out cakey.pem 2048
root@debian:/CA# openssl req -x509  -new -key cakey.pem -out cacsr.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Inc
Organizational Unit Name (eg, section) []:www.skills.com
Common Name (e.g. server FQDN or YOUR name) []:Skill Global Root CA
Email Address []:
```

![image-20230614001558969](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001558969.png)

- 请求证书

  注意申请证书不加 -x509

```bash
root@debian:/CA# openssl genrsa -out apache2.key
root@debian:/CA# openssl req -new -key apache2.key -out apache2.crs
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:China
Locality Name (eg, city) []:ShangDong
Organization Name (eg, company) [Internet Widgits Pty Ltd]:skills
Organizational Unit Name (eg, section) []:Operations Departments
Common Name (e.g. server FQDN or YOUR name) []:*.skills.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

- 颁发证书

```bash
root@debian:/CA# openssl x509 -req -in apache2.crs -CA cacsr.pem  -CAkey cakey.pem  -CAcreateserial -out apache2.crt
Signature ok
subject=C = CN, ST = China, L = ShangDong, O = skills, OU = Operations Departments, CN = *.skills.com
Getting CA Private Key
```

### 5.导入CA根证书

- 测试端

```bash
root@debian:~# mkdir /CA		##创建一个目录
```

- CA服务器

```bash
root@debian:~# scp /CA/cacsr.pem 192.168.100.20:/CA
root@192.168.100.20's password: 
cacsr.pem                                      100% 1342   609.1KB/s   00:00
```

- 测试端
```bash
root@debian:~# ls /CA
cacsr.pem
```

![image-20230614001616029](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001616029.png)

![image-20230614001623168](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001623168.png)

![image-20230614001631800](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001631800.png)

![image-20230614001640081](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001640081.png)

## 搭建主从DNS服务

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

## https重定向

> 重定向语句参考 http://t.csdn.cn/ToXTl

## 方法一

### 1.修改配置文件

```bash
root@debian:~# vim /etc/apache2/apache2.conf 
<Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All		##改为All
        Require all granted
</Directory>
```

### 2.定义重定向规则

```bash
root@debian:~# vim /var/www/html/.htaccess 
RewriteEngine on		##开启重定向
RewriteCond %{SERVER_PORT} !^443$		##当端口不为443时
RewriteRule ^(.*)$ https://%{SERVER_NAME}%{REQUEST_URI} [R=301]	
```

### 3.激活重定向模块

```bash
root@debian:~# a2enmod rewrite 
Enabling module rewrite.
To activate the new configuration, you need to run:
  systemctl restart apache2
```

### 4.重启服务

```bash
root@debian:~#systemctl restart apache2
```

## 方法二

```bash
root@debian:~# vim /etc/apache2/sites-available/000-default.conf 
redirect 301 "/" "https://www.dbw.com"
```

### 访问测试

![image-20230614001730027](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20230614001730027.png)

## 用户认证

### 1.修改配置文件

```bash
root@debian:~# vim /etc/apache2/apache2.conf 
<Directory /web/www/> ##指定对那一个目录进行认证
AuthType Basic				##加密方式
AuthName "password"		    ##显示在密码对话框中的提示
AuthUserFile /etc/apache2/peak		##//要创建的密码文件
Require user dbw		##添加认证用户
</Directory>
root@debian:~# cat /etc/apache2/sites-available/default2-ssl.conf 
<Directory /web/ftp/>
AuthType Basic
AuthNAme "password"
AuthUserFile /etc/apache2/peak
Require user dbw
</Directory>
```
### 2.添加用户及密码

```bash
root@debian:~# htpasswd -c /etc/apache2/peak dbw		
New password: 
Re-type new password: 
Adding password for user dbw
```

### 运行用户

```bash
root@debian:~# vim /etc/apache2/apache2.conf 
User webuser
Group webuser
```

