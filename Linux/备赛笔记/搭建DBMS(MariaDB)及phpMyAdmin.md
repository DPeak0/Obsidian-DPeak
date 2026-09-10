# 基于MariaDB搭建phpMyAdmin

`基于deband10`

## 服务端

### 安装数据库软件包

```bash
root@debian:~# apt install -y mariadb-server
```

### 初始设置

```bash
root@debian:~# mysql_secure_installation 

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 	##输入当前的数据库root密码,刚安装密码为空，回车即可
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y		##是否要设置root密码
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y		##是否禁止匿名用户登录
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n		##是否禁止root用户远程登录
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

### 修改配置文件

```bash
root@debian:~# vim /etc/mysql/debian.cnf 
# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = root
password = 123456 
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = root
password = 123456
socket   = /var/run/mysqld/mysqld.sock
basedir  = /usr
```

### 配置mariadb远程root登录

```bash
root@debian:~# mysql -uroot -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 36
Server version: 10.3.23-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mysql]> update user set plugin='mysql_native_password' where User='root' ;
Query OK, 1 row affected (0.000 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [mysql]> flush privileges;		###生效
Query OK, 0 rows affected (0.001 sec)
```

### 安装PHP

```bash
root@debian:~# apt install php7.3 php7.3-cgi php-common libapache2-mod-php7.3 php7.3-mysql php7.3-mbstring 
```

### 启用php

```bash
root@debian:~# a2enconf php7.3-cgi 
Enabling conf php7.3-cgi.
To activate the new configuration, you need to run:
  systemctl reload apache2
root@debian:~# systemctl reload apache2
```

### 安装phpMyAdmin

> 将phpMyAdmin压缩包上传到服务器
>
> 链接：https://pan.baidu.com/s/1rgTmFqLk8VRWUt64dG5ItQ?pwd=ummx 
> 提取码：ummx

```bash
root@debian:~# cd /var/www/html/
root@debian:/var/www/html# tar -zxvf phpMyAdmin-5.0.4-all-languages.tar.gz
root@debian:/var/www/html# mv phpMyAdmin-5.0.4-all-languages/* /var/www/html/
```

### 访问测试

![image-20221011191735109](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/phpmyadmin.png)

### 问题解决

### 配置文件需要一个短语密码

> 在phpmyadmin目录下libraries/config.default.php的第111行 输入46个随机字符

```bash
root@debian:/var/www/html# vim libraries/config.default.php 
111	$cfg['blowfish_secret'] = '';		##在此处输入46个随机字符
```

### tmp无法访问

> 在phpmyadminn目录下新建一个tmp目录并给其777权限即可

```bash
root@debian:/var/www/html# mkdir tmp
root@debian:/var/www/html# chmod 777 tmp
```

