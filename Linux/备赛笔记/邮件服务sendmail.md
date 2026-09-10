# sendmail邮件服务

`基于Centos 7.9`

>1. /etc/mail/sendmail.cf - Sendmail主配置文件
>2. /etc/mail/local-host-names - 包含本地主机名的列表，用于确定本地邮件地址
>3. /etc/mail/relay-domains - 指定允许中继邮件的域名列表
>4. /etc/mail/access - 指定允许或拒绝连接的主机和域名列表
>5. /etc/mail/virtusertable - 包含虚拟用户和真实用户之间映射关系的表格
>6. /etc/mail/trusted-users - 指定可信用户列表
>7. /etc/mail/aliases - 包含系统别名的列表，指定在邮件传递过程中将邮件发送给哪个用户
>8. /etc/mail/localnames - 包含本地邮件地址的列表，指定哪些邮件地址是本地邮件地址
>9. /etc/mail/service.switch - 用于指定Sendmail使用的名称服务
>10. /etc/mail/sendmail.mc - Sendmail配置文件的m4源文件
>
>(通常对sendmail.mc进行修改，然后使用m4编译器生成sendmail.cf文件)



### 案例

![image-20230330175805237](https://oss.bwihz.cn/typora/image-20230330175805237.png)

#### 创建用户

```bash
[root@mail ~]# useradd winmail
[root@mail ~]# useradd linmail
[root@mail ~]# echo "winmail:winmail" | chpasswd
[root@mail ~]# echo "linmail:linmail" | chpasswd
```

#### 配置两个区域邮件服务

```bash
[root@mail ~]# cat /etc/mail/access
Connect:2018skills.com		RELAY
Connect:lin.2018skills.com			RELAY
```

```bash
[root@mail ~]# cat /etc/mail/local-host-names
# local-host-names - include all aliases for your machine here.
2018skills.com
lin.2018skills.com
```

#### 配置监听地址


```bash
[root@mail ~]# vim /etc/mail/sendmail.mc
118 DAEMON_OPTIONS(`Port=smtp,Addr=0.0.0.0, Name=MTA')dnl
## 默认为127.0.0.1改为0.0.0.0为任何主机都可以访问Sendmail服务，或者改为192.168.10.0/24使一个特定的网段可以访问
```

#### 配置用户发邮件大小5MB

```bash
## 使用m4编译器生成sendmail.cf文件
[root@mail ~]# m4 /etc/mail/sendmail.mc > /etc/mail/sendmail.cf
```

```bash
[root@mail ~]# vim /etc/mail/sendmail.cf
 188 # maximum message size
 189 O MaxMessageSize=5242880
 #（限制整个邮件大小包括标题与内容的大小，单位为字节）
```

#### 测试

```bash
[root@mail ~]# yum install -y mailx
[root@mail ~]# echo "欢迎大家" | mail -s "你好" -r winmail@2018skills.com linmail@lin.2018skills.com
[root@mail ~]# echo "欢迎大家" | mail -s "你好" -r linmail@lin.2018skills.com winmail@2018skills.com
```

![image-20230330180957981](https://oss.bwihz.cn/typora/image-20230330180957981.png)

![image-20230330181016234](https://oss.bwihz.cn/typora/image-20230330181016234.png)

#### 创建邮件列表

```bash
[root@mail mail]# useradd everyone
[root@mail mail]# vim /etc/aliases
everyone:       winmail,linmail
```

## 邮件使用命令

### mailx

- 发送邮件

```bash
[root@mail ~]# echo "内容" | mail -s "标题"  -r 发件人 收件人
```

- 发送附件

```bash
[root@mail ~]# echo "内容" | mail -s "标题" -a 文件路径 收件人
```

