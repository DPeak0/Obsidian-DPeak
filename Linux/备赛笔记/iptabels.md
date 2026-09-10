> 
>
> https://www.zsythink.net/archives/1199
> 此博客有iptable整体的详细讲解多翻几页(理论知识)
>
> https://developer.aliyun.com/article/472310	
> 本次项目的命令可借鉴该教程(实战知识)
>
> https://www.jianshu.com/p/586da7c8fd42	 
> iptable防火 墙命令详解
>
> http://blog.sina.com.cn/s/blog_bc0d4b730102wv35.html	
> iptable进阶用法
>
> https://www.cnblogs.com/yyxianren/p/10910462.html
> (iptables的几种状态) 
>
> http://www.noobyard.com/article/p-sbnfinjo-mb.html
> (iptables超详细介绍)

#### **iptables常用命令：**

- **iptables -A 将一个规则添加到链末尾**
- **iptables -D 将指定的链中删除规则**
- **iptables -F 将指定的链中删除所有规则**
- iptables -I 将在指定链的指定编号位置插入一个规则
- iptables -L 列出指定链中所有规则
- iptables -t nat -L 列出所有NAT链中所有规则
- iptables -N 建立用户定义链
- iptables -X 删除用户定义链
- **iptables -P 修改链的默认设置，如将iptables -P INPUT DROP (将INPUT链设置为DROP)**

#### **常见设置参数介绍：**

- **--dport 指定目标TCP/IP端口 如 –dport 80**

- **--sport 指定源TCP/IP端口 如 –sport 80**

- **-p tcp 指定协议为tcp**

- **-p icmp 指定协议为ICMP**

- **-p udp 指定协议为UDP**

- **-j DROP 拒绝**

- **-j ACCEPT 允许**

- -j REJECT 拒绝并向发出消息的计算机发一个消息

- -j LOG 在/var/log/messages中登记分组匹配的记录

- -m mac –mac 绑定MAC地址

- -m limit –limit 1/s 1/m 设置时间策列

- **-s 10.10.0.0或10.10.0.0/16 指定源地址或地址段**

- **-d 10.10.0.0或10.10.0.0/16 指定目标地址或地址段**

- -s ! 10.10.0.0 指定源地址以外的

  <hr>

  ![在这里插入图片描述](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/2021042911014859.png)

![img](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/32929bc05b324fda9e3e1d25efb2fec5.png)

![img](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/3e512eb602784903ad7302fffc32022e.png)

<hr>

### 查看服务端口

#### 方法一 nmap

```bash
root@debian:~# apt install -y nmap
root@debian:~# nmap 127.0.0.1
```

#### 方法二 netstat

```bash
root@debian:~# apt install -y net-tools 
root@debian:~# netstat -tlnp     
```

<hr>

### 默认阻挡所以流量

```bash
root@debian:~# iptables -P INPUT DROP
```

### 开机自动加载iptables

#### 安装软件包

```bash
root@debian:~# apt install -y iptables-persistent 
```

#### 保存规则到/etc/iptables/rules.v4 

```bash
root@debian:~# iptables-save >/etc/iptables/rules.v4 
```

