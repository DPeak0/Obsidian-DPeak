# Debian CA证书服务

- **openssl** ：这是用于创建和管理OpenSSL证书，密钥和其他文件的基本命令行工具。
- **req** ：此子命令指定我们要使用X.509证书签名请求(CSR)管理。 “ X.509”是SSL和TLS对其密钥和证书管理所遵循的公共密钥基础结构标准。 我们要创建一个新的X.509证书，因此我们正在使用此子命令。
- **-x509** ：通过告诉实用程序我们要制作自签名证书而不是像通常那样生成证书签名请求，从而进一步修改了先前的子命令。
- **-days 365** ：此选项设置证书被视为有效的时间长度。 我们在这里设置了一年。
- **-newkey rsa：2048** ：这指定我们要同时生成一个新证书和一个新密钥。 我们没有在上一步中创建签名证书所需的密钥，因此我们需要将其与证书一起创建。 `rsa:2048`部分告诉它制作一个2048位长的RSA密钥。
- **-keyout** ：此行告诉OpenSSL在何处放置我们正在创建的生成的私钥文件。
- **-out** ：这告诉OpenSSL在哪里放置我们要创建的证书。

```bash
openssl genrsa -out 根密钥名.key 2048     ##创建根密钥
openssl req -x509 -new -key 根密钥名.key -out 根证书名.pem ##创建根证书

openssl genrsa -out 密钥名.key		##创建ssl密钥
openssl req -new -key 密钥名.key -out 证书名.csr	##证书请求

openssl x509 -req -in 请求证书名.csr -CA 根证书名.pem -CA 根密钥名.key -CAcreateserial -out 证书名.crt ##颁发证书
```
- **C  国家 = CN**
- ST = China
- L = ShangHai
- **O  单位 = Inc**
- **OU  组织机构 = www.skills.com**
- **CN  （域）=  \*.skills.com**

<hr>

### 1.安装软件包

```bash
root@debian:~# apt install -y openssl 
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/ssl/openssl.cnf
dir = /CA           ##修改证书目录
```

### 3.创建目录

```bash
root@debian:~# mkdir /CA		##创建目录
root@debian:~# cp -a /etc/ssl/* /CA		##将ssl目录复制进/CA
root@debian:~# cd /CA		##进入目录
```

### 4.签发根证书

- 新建一个名为cakey.pem的根密钥

```bash
root@debian:/CA# openssl genrsa -out cakey.pem 2048
Generating RSA private key, 2048 bit long m人们odulus (2 primes)
...........+++++
...+++++
e is 65537 (0x010001)
root@debian:/CA# ls
cakey.pem  certs  openssl.cnf  private
```

- 签发一个基于cakey.pem根密钥生成的名为cacrt.pem的根证书

```bash
root@debian:/CA# openssl req -x509 -new -key cakey.pem -out cacrt.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN		#国家	C
State or Province Name (full name) [Some-State]:Hubei	#省
Locality Name (eg, city) []:Wuhan	#市
Organization Name (eg, company) [Internet Widgits Pty Ltd]:HBGT #单位 O     
Organizational Unit Name (eg, section) []:jiwang2104	#组织 OU
Common Name (e.g. server FQDN or YOUR name) []:DBW	#域 CN
Email Address []:                
root@debian:/CA# 
```

### 5.创建证书申请

- 创建证书密钥

```bash
root@debian:/CA# openssl genrsa -out miyao.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
..........................................+++++
...+++++
e is 65537 (0x010001)
```

- 创建申请证书(与根证书相同)

```bash
root@debian:/CA# openssl req  -new -key miyao.key -out miyao.crs
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Hubei
Locality Name (eg, city) []:Wuhan
Organization Name (eg, company) [Internet Widgits Pty Ltd]:HBGT
Organizational Unit Name (eg, section) []:jiwang2104
Common Name (e.g. server FQDN or YOUR name) []:DBW
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
root@debian:/CA# 
```

### 6.签发证书

```bash
root@debian:/CA# openssl x509 -req -in miyao.crs -CA cacrt.pem -CAkey cakey.pem -CAcreateserial -out zhengshu.crt
Signature ok
subject=C = CN, ST = Hubei, L = Wuhan, O = HBGT, OU = jiwang2104, CN = DBW
Getting CA Private Key
```

### 7.查看证书信息

```bash
root@debian:/CA# openssl x509 -in zhengshu.crt -text 
```

