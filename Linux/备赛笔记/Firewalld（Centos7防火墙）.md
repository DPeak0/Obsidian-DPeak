# Firewalld（Centos7防火墙）

>参考链接
>http://t.csdn.cn/rzmQX		简介
>
>http://t.csdn.cn/e2Fdj		配置

### 一、linux防火墙简介

#### 防火墙技术种类：
> （1）包过滤防火墙 packet filtering

> （2）应用代理防火墙 application proxy

> （3）状态检测防火墙 stateful inspection

（firewalld是包过滤防火墙，所以这里只讲包过滤防火墙）

#### 包过滤防火墙概述：
> （1）netfilter： 位于Linux内核中的包过滤功能体系，成为Linux防火墙的“内核态”

> （2）firewalld： CentOS7默认的管理防火墙规则的工具，成为Linux防火墙的“用户态”

(上面的两种称呼都可以表示为Linux防火墙)

#### 包过滤的工作层次：
> （1）主要是网络层，针对IP数据包、检查源IP

> （2）体现在对包内的IP地址、端口等信息的处理上

(Linux服务器主要用于对互联网提供某种服务或作为内部局域网的网关服务器)

![包过滤的工作层次](https://dbw-typora.oss-cn-hangzhou.aliyuncs.com/typora/20210104095245139.png)

### 二、firewalld简介

#### 概述：
防火墙是Linux系统的 主要的安全工具 ，可以提供基本的安全防护，在Linux历史上已经使用过的防火墙工具包括：ipfwadm、ipchains、iptables （即Centos6就是使用的iptables） ，而在firewalld中新引入了 区域（Zone） 这个概念

#### 特点：
以前的iptables防火墙是静态的，每次修改都要求防火墙完全重启，这个过程包括内核netfilter防火墙模块的卸载和新配置所需模块的装载等，而模块的卸载将会破坏状态防火墙和确立建立的连接，现在firewalld可以动态管理防火墙，firewalld把netfilter的过滤功能集于一身
总结成几个小点：

- firewalld提供了支持 网络/防火墙区域（zone）定义网络链接以及接口安全等级的动态防火墙管理工具。
- 支持 IPv4，IPv6 的防火墙设置以及以太网桥接
- 支持服务或者应用程序直接添加防火墙规则的接口
- 拥有运行时配置和永久配置两种选项。
  运行时配置——服务或系统重启后失效
  永久配置——服务或系统关机、重启后生效

#### 网络区域：
firewalld预定义的九种网络区域：

trusted、public、work、home、internal、external、dmz、block、drop

————默认情况就有一些有效的区域，由firewalld提供的区域按照从不信任到信任的顺序排序

> （1）丢弃区域（Drop Zone）： 如果使用丢弃区域，任何进入的数据包将被丢弃，这个类似于Centos6上的 iptables -j drop ，使用丢弃规则意味着将不存在相应。

> （2）阻塞区域（Block Zone）： 阻塞区域会拒绝进入的网络连接，返回icmp-host-prohibited，只有服务器已经建立的连接会被通过，即只允许由该系统初始化的网络连接。

> （3）公共区域（Public Zone）： 只接受那些被选中的连接，默认只允许ssh和dhcpv6-client，这个zone是缺省zone （缺省就是默认的意思，所以公共区域也是默认区域，在没有任何配置的情况下走的是公共区域）

> （4）外部区域（External Zone）： 这个区域相当于路由器的启动伪装（masquerading）选项，只有指定的连接会被接受，即ssh，而其他的连接将被丢弃或者不被接受

> （5）隔离区域（DMZ Zone）： 如果想要只允许给部分服务能被外部访问，可以在DMZ区域中定义，它也拥有只通过被选中连接的特性，即ssh， 这个区域又叫做非军事化区域

> （6）工作区域（Work Zone）： 在这个区域中，我们只能定义内部网络，比如私有网络通信才被允许， 只允许ssh、ipp-client和dhcpv6-client

> （7）家庭区域（Home Zone）： 这个区域专门用于家庭环境，它同样只允许被选中的连接， 即ssh、ipp-client、mdns、samba-client和dhcpv6-client

> （8）内部区域（Internal Zone）： 这个区域和 工作区域（Work Zone） 类似，只允许通过被选中的连接，与 家庭区域（Home Zone） 相同

> （9）信任区域（Trusted Zone）： 信任区域允许所有网络通信通过，因为 信任区域（Trusted Zone）是最被信任的，即使没有设置任何的服务，那么也是被允许的，因为 信任区域（Trusted Zone）是允许所有连接的

(以上是系统定义的所有的区域（Zone），但是，不是所有的区域（Zone）都在使用，只有活跃的区域（Zone）才有实际操作意义)

注意：因为默认区域只允许ssh和dhcp，所以在没有任何配置的情况下默认是拒绝ping包的

#### 检查原则：
如果一个客户端访问服务器，服务器根据以下原则决定使用哪个区域（zone）的策略去匹配：

> （1）如果一个客户端数据包的源IP地址匹配Zone的 来源（sources） 也就是匹配区域的规则，那么该 Zone的规则就适用这个客户端，一个源只能属于一个 Zone，不能同时属于多个区域（Zone）

> （2）如果一个客户端数据包进入服务器的某一个接口（如ens33网卡接口）匹配了 Zone的 接口（interfaces），则该Zone的规则就适用这个客户端，一个接口只能属于一个 Zone ，不能同时属于多个Zone

> （3）如果上述两个原则都不满足，那么默认的 Zone将被应用 firewalld数据处理流程 ，检查数据来源的源地址，

#### firewalld数据处理流程：
检查数据来源的源地址：

> （1）若源地址关联到特定的区域，则执行该区域所制定的规则

> （2）若源地址未关联到特定的区域，则使用关联网络接口的区域并执行该区域所制定的规则

> （3）若网络接口未关联到特定的区域，则使用默认区域并执行该区域所制定的规则

#### 数据包处理原则：
检查源地址的处理规则：

- 匹配源地址所在区域
- 匹配入站接口所在区域
- 匹配默认区域

#### firewalld防火墙的配置方法：

有三种配置方法，分别是：

- firewall-config图行化工具
- firewall-cmd命令行工具
- /etc/firewalld/中的配置文件

### 三、配置firewalld防火墙

#### 使用firewall-cmd命令进行配置

```bash
# 启动防火墙
[root@Firewalld ~]# systemctl start firewalld
# 设置防火墙为开机自启
[root@Firewalld ~]# systemctl enable firewalld
# 查看防火墙运行状态
[root@Firewalld ~]# systemctl status firewalld
[root@Firewalld ~]# firewall-cmd --state

# 查看防火墙可用区域
[root@Firewalld ~]# firewall-cmd --get-zones
# 查看防火墙默认区域
[root@Firewalld ~]# firewall-cmd --get-default-zone
# 查看防火墙可用服务
[root@Firewalld ~]# firewall-cmd --get-service
# 查看防火墙可用的icmp阻塞类型
[root@Firewalld ~]# firewall-cmd --get-icmptypes 


# 临时加一个端口到public区域
[root@zcwyou ~]# firewall-cmd --zone=public --add-port=8080/tcp

-----------------------华丽分割线-----------------------
若要永久生效方法加参数–permanent

拒绝所有包：firewall-cmd --panic-on    
取消拒绝状态： firewall-cmd --panic-off 
查看是否拒绝： firewall-cmd --query-panic 
-------------------------------------------------------

```

##### 如何清空防火墙规则：

当防火墙配置了很多规则想一次性清空怎么办，firewalld默认没有命令来清空规则，但是我们可以通过编辑配置文件来清除规则。
**`/etc/firewalld/zones/ 此处为配置生效后保存的配置文件，建议修改前先备份`**

```bash
## 此目录会显示已经配置的规则
[root@Firewalld ~]# ll /etc/firewalld/zones/
总用量 8
-rw-r--r--. 1 root root 315 7月   2 2020 public.xml
-rw-r--r--. 1 root root 315 7月   2 2020 public.xml.old

[root@Firewalld zones]# vim /tmp/firewalld.bak/public.xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>                 # 这两条service开头的就是允许的服务
  <service name="dhcpv6-client"/>
</zone>

# 在修改之前先创建备份目录
[root@Firewalld ~]# mkdir /tmp/firewalld.bak  （备份目录为firewalld.bak）
[root@Firewalld ~]# mv /etc/firewalld/zones/* /tmp/firewalld.bak/

[root@Firewalld ~]# firewall-cmd --set-default-zone=public   （设置默认区域为public）

[root@Firewalld ~]# systemctl restart firewalld

```

##### 区域管理选项说明

```bash
（1）--get-default-zone                          显示网络连接或接口的默认区域
（2）--set-default-zone=区域名称                  设置网络连接或接口的默认区域
（3）--get-active-zones                          显示已激活的所有区域
（4）--get-zone-of-interface=网卡名称             显示指定接口绑定的区域
（5）--zone=区域名称 --add-interface=网卡名称      为指定接口绑定区域
（6）--zone=区域名称 --change-interface=网卡名称   为指定的区域更改绑定的网络接口
（7）--zone=区域名称 --remove-interface=网卡名称   为指定的区域删除绑定的网络接口
（8）--list-all-zones                             显示所有区域及其规则
（9）--zone=区域名称 --list-all                    显示指定区域的所有规则
（10）--list-all                                  显示默认区域的所有规则
```

- 示例

```bash
# 显示当前系统的默认区域
[root@Firewalld ~]# firewall-cmd --get-default-zone 
public

# 显示默认区域的所有规则
[root@Firewalld ~]# firewall-cmd --list-all
public (active)                #表示public这个区域是活动区域即可用区域，如果是默认区域会多一个defaults
  target: default                  
  icmp-block-inversion: no         
  interfaces: ens33            #表示public这个区域的网卡接口是ens33
  sources:             #列出了public这个区域的源，现在这里没有，但是如果有的话，格式是xxx.xxx.xxx.xxx/xx
  services: dhcpv6-client ssh    #表示public区域允许通过的服务类型
  ports:                       #表示public区域允许通过的端口
  protocols:                   #允许的通过的协议
  masquerade: no               #表示这个区域不允许ip伪装，如果允许的话也同时会允许IP转发，即开启路由功能
  forward-ports:               #列出转发的端口
  source-ports:                   
  icmp-blocks:                #列出阻塞icmp流量的黑名单
  rich rules:                 #在public区域中优先处理的高级配置
  
-------------------------华丽丽的分割线-------------------------
target的作用：
当一个区域处理它的源或接口上的一个包时，但是没有处理该包的显式规则时，这个时候区域的目标target决定了该行为
（1）ACCEPT ： 通过这个包
（2）%%REJECT%% ： 拒绝这个包，并且返回一个拒绝的回复
（3）DROP ： 丢弃这个包，不回复任何信息
（4）default ： 不做任何事情，该区域不再管他，把它提到“楼上”
--------------------------------------------------------------

# 显示网络接口ens33的对应区域
[root@Firewalld ~]# firewall-cmd --get-zone-of-interface=ens33
public                           （说明ens33的区域是public）

# 更改ens33的区域为internal
[root@Firewalld ~]# firewall-cmd --zone=internal --change-interface=ens33 
success                          （更改成功）

[root@Firewalld ~]# firewall-cmd --get-zone-of-interface=ens33
internal                         （再次查看发现已经更改为internal区域）

# 查看全部活动区域
[root@Firewalld ~]# firewall-cmd --get-active-zones 
internal
  interfaces: ens33

```

##### 服务管理选项说明

```bash
# 服务存放在/usr/lib/firewalld/services目录中，通过单个的xml配置文件来指定
# xml文件：service-name.xml
（1）--zone=区域名称 --list-services                         显示指定区域内允许访问的所有服务
（2）--zone=区域名称 --add-service=服务名称                   为指定区域设置允许访问的某项服务
（3）--zone=区域名称 --remove-service=服务名称                删除指定区域已设置的允许访问的某项服务
（4）--zone=区域名称 --list-ports                            显示指定区域内允许访问的所有端口号
（5）--zone=区域名称 --add-port=端口号-端口号/协议名          为指定区域设置允许访问的某个或某段端口号并指定协议名（中间的-表示从多少到多少端口号， / 和后面跟端口的协议）
（6）--zone=区域名称 --remove-port=端口号-端口号/协议名        删除指定区域已设置的允许访问的某个端口号或某段端口号并且指定协议名（中间的-表示从多少到多少端口号， / 和后面跟端口的协议）
（7）--zone=区域名称 --list-icmp-blocks                      显示指定区域内拒绝访问的所有ICMP类型
（8）--zone=区域名称 --add-icmp-block=icmp类型               为指定区域设置拒绝访问的某项ICMP类型
（9）--zone=区域名称 --remove-icmp-block=icmp类型            删除指定区域已设置的拒绝访问的某项ICMP类型，省略 --zone=区域名称 时表示对默认区域操作
```

- 示例

```bash
# 显示默认区域允许访问的所有服务
[root@Firewalld ~]# firewall-cmd --list-services
You're performing an operation over default zone ('public'),
but your connections/interfaces are in zone 'internal' (see --get-active-zones)
You most likely need to use --zone=internal option.

dhcpv6-client ssh    （说明允许dhcp和ssh）

# 设置默认区域允许访问http和https服务 （不加--zone指定的话就是配置默认区域）
[root@Firewalld ~]# firewall-cmd --add-service=http
You're performing an operation over default zone ('public'),
but your connections/interfaces are in zone 'internal' (see --get-active-zones)
You most likely need to use --zone=internal option.

success

[root@Firewalld ~]# firewall-cmd --add-service=https
You're performing an operation over default zone ('public'),
but your connections/interfaces are in zone 'internal' (see --get-active-zones)
You most likely need to use --zone=internal option.

success

[root@Firewalld ~]# firewall-cmd --list-services   （再次查看）
You're performing an operation over default zone ('public'),
but your connections/interfaces are in zone 'internal' (see --get-active-zones)
You most likely need to use --zone=internal option.

dhcpv6-client http https ssh  （多了http和https）
```

```bash
# 预定义的服务可以使用服务名配置，同时其对应端口会自动打开，非预定义的服务只能手动指定端口给指定区域添加tcp443端口

[root@Firewalld ~]# firewall-cmd --zone=internal --add-port=443/tcp
success

[root@Firewalld ~]# firewall-cmd --zone=internal --list-all 
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources: 
  services: dhcpv6-client mdns samba-client ssh
  ports: 443/tcp  （发现添加成功）
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

# 删除指定区域的443/tcp端口
[root@Firewalld ~]# firewall-cmd --zone=internal --remove-port=443/tcp
success

[root@Firewalld ~]# firewall-cmd --zone=internal --list-all 
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources: 
  services: dhcpv6-client mdns samba-client ssh
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

##### 两种配置方式

- 运行时模式（runtime mode）： 当前内存中运行，系统或防火墙服务重启或停止，配置失效
- 永久模式（permanent mode）： 永久存储在配置文件中，但是配置完成要重启系统或防火墙

```bash
#相关选项：（配置时添加此选项即可）
--reload 将永久配置应用为运行时配置 
--permanent 设置永久性规则。服务重启或重新加载时生效

--runtime-to-permanent 将当前的运行时配置写入规则，成为永久性配置 
```

## firewall-cmd配置命令

### 4.基本配置

```bash
4.1 安装firewalld
[root@zcwyou ~]# yum install firewalld firewall-config

4.2 启动服务
[root@zcwyou ~]# systemctl start firewalld

4.3 开机自动启动服务
[root@zcwyou ~]# systemctl enable firewalld

4.4 查看状态
[root@zcwyou ~]# systemctl status firewalld
[root@zcwyou ~]# firewall-cmd --state

4.5 关闭服务
[root@zcwyou ~]# systemctl stop firewalld

4.6 取消开机启动
[root@zcwyou ~]# systemctl disable firewalld

4.7 弃用FirewallD防火墙，改用iptables （你也可以关闭目前还不熟悉的FirewallD防火墙，而使用iptables,但不建议:）
[root@zcwyou ~]# yum install iptables-services
[root@zcwyou ~]# systemctl start iptables
[root@zcwyou ~]# systemctl enable iptables

4.8 查看版本
[root@zcwyou ~]# firewall-cmd --version

4.9 查看帮助
[root@zcwyou ~]# firewall-cmd --help

4.10 显示状态
[root@zcwyou ~]# firewall-cmd --state

4.11 查看活动区域信息
[root@zcwyou ~]# firewall-cmd --get-active-zones

4.12 查看XX接口所属区域
[root@zcwyou ~]# firewall-cmd --get-zone-of-interface=XX

4.13 拒绝所有包
[root@zcwyou ~]# firewall-cmd --panic-on

4.14 取消拒绝状态
[root@zcwyou ~]# firewall-cmd --panic-off

4.15 查看是否拒绝
[root@zcwyou ~]# firewall-cmd --query-panic

4.16 查看firewalld是否开启
[root@zcwyou ~]# systemctl is-enabled firewalld

4.17 重启加载防火墙（以 root 身份输入以下命令，重新加载防火墙，并不中断用户连接，即不丢失状态信息：)
[root@zcwyou ~]# firewall-cmd --reload

4.18 完全重启防火墙 (以 root 身份输入以下命令，重新加载防火墙并中断用户连接，即丢弃状态信息：)
[root@zcwyou ~]# firewall-cmd --complete-reload

注意:通常在防火墙出现严重问题时，这个命令才会被使用。比如，防火墙规则是正确的，但却出现状态信息问题和无法建立连接。

firewall-cmd --reload与firewall-cmd --complete-reload两者的区别就是：

第一个无需断开连接，就是firewalld特性之一动态添加规则，第二个需要断开连接，类似重启服务

4.19 显示默认区域
[root@zcwyou ~]# firewall-cmd --get-default-zone

4.20 添加接口到区域（将接口添加到XX区域,如果不指定区域,则添加到默认区域）
[root@zcwyou ~]# firewall-cmd --zone=XX --add-interface=eth0

永久生效再加上–permanent 然后reload防火墙

4.21 设置默认区域，立即生效无需重启
[root@zcwyou ~]# firewall-cmd --set-default-zone=XX

4.22 查看XX区域打开的端口
[root@zcwyou ~]# firewall-cmd --zone=XX --list-ports

4.23 查看XX区域加载的服务
[root@zcwyou ~]# firewall-cmd --zone=XX --list-services

4.24 临时加一个端口到XX区域
[root@zcwyou ~]# firewall-cmd --zone=XX --add-port=8080/tcp

若要永久生效方法加参数–permanent

4.25 打开一个服务，类似于将端口可视化，服务需要在配置文件中添加，/etc/firewalld 目录下有services文件夹，查看其它的xml文件以及参考前面说方法
[root@zcwyou ~]# firewall-cmd --zone=work --add-service=smtp

4.26 移除服务
[root@zcwyou ~]# firewall-cmd --zone=work --remove-service=smtp

4.27 显示支持的区域列表
[root@zcwyou ~]# firewall-cmd --get-zones

4.28 列出全部区域启用的特性
[root@zcwyou ~]# firewall-cmd --list-all-zones

4.29 显示XX区域详情
[root@zcwyou ~]# firewall-cmd --zone=XX --list-all

4.30 查看当前活跃区域
[root@zcwyou ~]# firewall-cmd --get-active-zones

4.31 设置XX接口所属区域
[root@zcwyou ~]# firewall-cmd --get-zone-of-interface=XX

4.32 查询YY区域中是否包含XX接口
[root@zcwyou ~]# firewall-cmd --zone=YY --query-interface=XX

4.33 删除指定XX网卡所在的zone(以YY为例)
[root@zcwyou ~]# firewall-cmd --zone=YY --remove-interface=XX

4.34 临时修改XX接口为YY区域(永久修改加参数–permanent）
[root@zcwyou ~]# firewall-cmd --zone=YY --change-interface=XX

4.35 控制端口 / 服务

可以通过两种方式控制端口的开放：

1）一种是指定端口号，另一种是指定服务名。

虽然开放 http 服务就是开放了 80 端口，但是还是不能通过端口号来关闭，也就是说通过指定服务名开放的就要通过指定服务名关闭；

2）通过指定端口号开放的就要通过指定端口号关闭。

3）还有一个要注意的就是指定端口的时候一定要指定是什么协议，tcp 还是 udp。

4.36 富规则

[root@zcwyou ~]# firewall-cmd --permanent --add-rich-rule=“rule family=“ipv4” source address=“192.168.142.166” port protocol=“tcp” port=“5432” accept”
[root@zcwyou ~]# systemctl restart firewalld.service

```

### 5.firewalld服务管理

```bash

5.1 显示支持的服务
[root@zcwyou ~]# firewall-cmd --get-services

5.2 临时允许Samba服务通过600秒
[root@zcwyou ~]# firewall-cmd --add-service=samba --timeout=600

5.3 显示默认区域开启的服务,如果要查某区域,加参数–zone=XX
[root@zcwyou ~]# firewall-cmd --list-services

5.4 添加HTTP服务到内部区域（internal）,并保存到配置文件
[root@zcwyou ~]# firewall-cmd --permanent --zone=internal --add-service=http

5.5 在不改变状态的条件下重新加载防火墙
[root@zcwyou ~]# firewall-cmd --reload

5.6 开放mysql服务
[root@zcwyou ~]# firewall-cmd --add-service=mysql

5.7 阻止mysql服务
[root@zcwyou ~]# firewall-cmd --remove-service=mysql

5.8 端口管理，临时打开443/TCP端口,立即生效
[root@zcwyou ~]# firewall-cmd --add-port=443/tcp

5.9 永久打开3690/TCP端口
[root@zcwyou ~]# firewall-cmd --permanent --add-port=3690/tcp

5.10 永久打开端口需要reload一下，如果用了reload临时打开的端口就失效了
[root@zcwyou ~]# firewall-cmd --reload

5.11 查看防火墙所有区域的设置，包括添加的端口和服务
[root@zcwyou ~]# firewall-cmd --list-all

5.12 开放通过tcp访问3306
[root@zcwyou ~]# firewall-cmd --add-port=3306/tcp

5.13 阻止tcp80
[root@zcwyou ~]# firewall-cmd --remove-port=80/tcp

5.14 开放通过udp访问233
[root@zcwyou ~]# firewall-cmd --add-port=233/udp

5.15 查看开放的端口
[root@zcwyou ~]# firewall-cmd --list-ports

5.16 开放自定义的ssh端口号为12222 （–permanent参数可以将永久保存到配置文件）
[root@zcwyou ~]# firewall-cmd --add-port=12222/tcp --permanent

重启防火墙。永久打开端口需要reload一下，如果用了reload临时打开的端口就失效了
[root@zcwyou ~]# firewall-cmd --reload

5.16 添加端口范围
[root@zcwyou ~]# firewall-cmd --add-port=2000-4000/tcp

5.17 针对指定zone XX添加端口
[root@zcwyou ~]# firewall-cmd --permanent --zone=XX --add-port=443/tcp
```

### 6.管理区域中的对象

```bash
6.1 获取永久支持的区域
[root@zcwyou ~]# firewall-cmd --permanent --get-zones

6.2 启用区域中的服务（此举将永久启用区域中的服务。如果未指定区域，将使用默认区域。）
firewall-cmd --permanent [–zone=] --add-service=

6.3 临时开放mysql服务,立即生效
[root@zcwyou ~]# firewall-cmd --add-service=mysql

6.4 public区域,添加httpd服务,并保存,但不会立即生效,需要reload防火墙
[root@zcwyou ~]# firewall-cmd --permanent --zone=public --add-service=httpd

6.5 public区域,禁用httpd服务,并保存,但不会立即生效,需要reload防火墙
[root@zcwyou ~]# firewall-cmd --permanent --zone=public --remove-service=httpd
```

### 7.端口转发

端口转发可以将指定地址访问指定的端口时，将流量转发至指定地址的指定端口。

转发的目的如果不指定ip的话就默认为本机，如果指定了ip却没指定端口，则默认使用来源端口。

典型的做法:

1）NAT内网端口映射
2）SSH隧道转发数据

如果配置好端口转发之后不能用，可以检查下面两个问题：

比如我将 80 端口转发至 8080 端口，首先检查本地的 80 端口和目标的 8080 端口是否开放监听了

其次检查是否允许伪装 IP，没允许的话要开启伪装 IP

```bash
7.1 将80端口的流量转发至8080
[root@zcwyou ~]# firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080

7.2 将80端口的流量转发至192.168.0.1
[root@zcwyou ~]# firewall-cmd --add-forward-port=proto=80:proto=tcp:toaddr=192.168.1.0.1

7.3 将80端口的流量转发至192.168.0.1的8080端口
[root@zcwyou ~]# firewall-cmd --add-forward-port=proto=80:proto=tcp:toaddr=192.168.0.1:toport=8080

7.4 禁止区域的端口转发或者端口映射

firewall-cmd [–zone=] --remove-forward-port=port=[-]:proto= { :toport=[-] | :toaddr=| :toport=[-]:toaddr=}

7.5 查询区域的端口转发或者端口映射

firewall-cmd [–zone=] --query-forward-port=port=[-]:proto= { :toport=[-] | :toaddr=| :toport=[-]:toaddr=}

7.6 在区域中永久启用端口转发或映射

firewall-cmd --permanent [–zone=] --add-forward-port=port=[-]:proto= { :toport=[-] | :toaddr=| :toport=[-]:toaddr=}

端口可以映射到另一台主机的同一端口，也可以是同一主机或另一主机的不同端口。

端口号可以是一个单独的端口 或者是端口范围 。

协议可以为 tcp 或udp 。

目标端口可以是端口号 或者是端口范围 。

目标地址可以是 IPv4 地址。受内核限制，端口转发功能仅可用于IPv4。

7.7永久禁止区域的端口转发或者端口映射

firewall-cmd --permanent [–zone=] --remove-forward-port=port=[-]:proto= { :toport=[-] | :toaddr=| :toport=[-]:toaddr=}

7.8 查询区域的端口转发或者端口映射状态

firewall-cmd --permanent [–zone=] --query-forward-port=port=[-]:proto= { :toport=[-] | :toaddr=| :toport=[-]:toaddr=}

如果服务启用，此命令将有返回值。此命令没有输出信息。

7.9 将 home 区域的 ssh 服务转发到 127.0.0.2

[root@zcwyou ~]# firewall-cmd --permanent --zone=home --add-forward-port=port=22:proto=tcp:toaddr=127.0.0.2
```

### 8.伪装 IP

```bash
8.1 检查是否允许伪装IP
[root@zcwyou ~]# firewall-cmd --query-masquerade

8.2 允许防火墙伪装IP
[root@zcwyou ~]# firewall-cmd --add-masquerade

8.3 禁止防火墙伪装IP
[root@zcwyou ~]# firewall-cmd --remove-masquerade

8.4 永久启用区域中的伪装
firewall-cmd --permanent [–zone=] --add-masquerade

此举启用区域的伪装功能。私有网络的地址将被隐藏并映射到一个公有IP。

这是地址转换的一种形式，常用于路由。由于内核的限制，伪装功能仅可用于IPv4。

8.5 临时禁用区域中的 IP 伪装
firewall-cmd [–zone=] --remove-masquerade

8.6 永久禁用区域中的伪装
firewall-cmd --permanent [–zone=] --remove-masquerade

8.7 查询区域中的伪装的永久状态
firewall-cmd --permanent [–zone=] --query-masquerade

如果服务启用，此命令将有返回值。此命令没有输出信息。

8.8 查询区域的伪装状态
firewall-cmd [–zone=] --query-masquerade

如果启用，此命令将有返回值。没有输出信息。
```

### 9.ICMP控制

```bash
9.1 获取永久选项所支持的ICMP类型列表
[root@zcwyou ~]# firewall-cmd --permanent --get-icmptypes

9.2 获取所有支持的ICMP类型
[root@zcwyou ~]# firewall-cmd --get-icmptypes

9.3 永久启用区域中的ICMP阻塞,需要reload防火墙,
firewall-cmd --permanent [–zone=] --add-icmp-block=

此举将启用选中的 Internet 控制报文协议 （ICMP） 报文进行阻塞。ICMP 报文可以是请求信息或者创建的应答报文或错误应答报文。

9.4 永久禁用区域中的ICMP阻塞,需要reload防火墙,
firewall-cmd --permanent [–zone=] --remove-icmp-block=

9.5 查询区域中的ICMP永久状态
firewall-cmd --permanent [–zone=] --query-icmp-block=

如果服务启用，此命令将有返回值。此命令没有输出信息。

阻塞公共区域中的响应应答报文:
[root@zcwyou ~]# firewall-cmd --permanent --zone=public --add-icmp-block=echo-reply

9.6 立即启用区域的 ICMP 阻塞功能
firewall-cmd [–zone=] --add-icmp-block=

此举将启用选中的 Internet 控制报文协议 （ICMP） 报文进行阻塞。 ICMP 报文可以是请求信息或者创建的应答报文，以及错误应答。

9.7 立即禁止区域的 ICMP 阻塞功能
firewall-cmd [–zone=] --remove-icmp-block=

9.8 查询区域的 ICMP 阻塞功能
firewall-cmd [–zone=] --query-icmp-block=

如果启用，此命令将有返回值。没有输出信息。

例: 阻塞区域的响应应答报文:
[root@zcwyou ~]# firewall-cmd --zone=public --add-icmp-block=echo-reply
```

## 10.通过配置文件来使用Firewalld的方法

系统本身已经内置了一些常用服务的防火墙规则,存放在/usr/lib/firewalld/services/

注意!!!请勿编辑/usr/lib/firewalld/services/ ，只有 /etc/firewalld/services 的文件可以被编辑。

以下例子均以系统自带的public zone 为例子.
```bash

10.1 案例1: 如果想开放80端口供外网访问http服务,操作如下

Step1:将 http.xml复制到/etc/firewalld/services/下面,以服务形式管理防火墙,

系统会优先去读取 /etc/firewalld 里面的文件,读取完毕后,会去/usr/lib/firewalld/services/ 再次读取。为了方便修改和管理,强烈建议复制到/etc/firewalld

[root@zcwyou ~]# cp /usr/lib/firewalld/services/http.xml /etc/firewalld/services/

修改/etc/firewalld/zones/public.xml,加入http服务

vi /etc/firewalld/zones/public.xml

Public For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.

**加入这行,要匹配 /etc/firewalld/services/文件夹下的文件名**

以 root 身份输入以下命令，重新加载防火墙，并不中断用户连接，即不丢失状态信息：

[root@zcwyou ~]# firewall-cmd --reload

或者以 root 身份输入以下信息，重新加载防火墙并中断用户连接，即丢弃状态信息：

[root@zcwyou ~]# firewall-cmd --complete-reload

注意:通常在防火墙出现严重问题时，这个命令才会被使用。比如，防火墙规则是正确的，但却出现状态信息问题和无法建立连接。

10.2 案例2: SSH为非默认端口,要求能正常访问

[root@zcwyou ~]# cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/
[root@zcwyou ~]# vi /etc/firewalld/services/ssh.xml
把默认22修改为目前的SSH端口号

[root@zcwyou ~]# firewall-cmd --reload

10.3 案例3:修改区域配置文件只允许特定主机192.168.23.1连接SSH

[root@zcwyou ~]# cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/
[root@zcwyou ~]# vi /etc/firewalld/zones/public.xml

确保配置文件有以下内容

配置结束

重启防火墙后生效
[root@zcwyou ~]# firewall-cmd --reload
```

## 11.firewalld直接模式


对于最高级的使用，或 iptables 专家，FirewallD 提供了一个Direct接口，允许你给它传递原始 iptables 命令。

直接接口规则不是持久的，除非使用 --permanent。

直接选项主要用于使服务和应用程序能够增加规则。 规则不会被保存，在重新加载或者重启之后必须再次提交。传递的参数 与 iptables, ip6tables 以及 ebtables 一致。

选项 –direct 需要是直接选项的第一个参数。将命令传递给防火墙。参数 可以是 iptables, ip6tables 以及 ebtables 命令行参数。

firewall-cmd --direct --passthrough { ipv4 | ipv6 | eb }
```bash

11.1 为表增加一个新链 。

firewall-cmd --direct --add-chain { ipv4 | ipv6 | eb }

11.2 从表中删除链 。

firewall-cmd --direct --remove-chain { ipv4 | ipv6 | eb }

11.3 查询链是否存在与表如果是，返回0,否则返回1.

firewall-cmd --direct --query-chain { ipv4 | ipv6 | eb }

如果启用，此命令将有返回值。此命令没有输出信息。

11.4 获取用空格分隔的表中链的列表。

firewall-cmd --direct --get-chains { ipv4 | ipv6 | eb }

11.5 为表增加一条参数为 的链 ，优先级设定为 。

firewall-cmd --direct --add-rule { ipv4 | ipv6 | eb }

11.6 从表中删除带参数的链 。

firewall-cmd --direct --remove-rule { ipv4 | ipv6 | eb }

11.7 查询带参数的链 是否存在表中. 如果是，返回0,否则返回1.

firewall-cmd --direct --query-rule { ipv4 | ipv6 | eb }

如果启用，此命令将有返回值。此命令没有输出信息。

11.8 获取表中所有增加到链的规则，并用换行分隔。

firewall-cmd --direct --get-rules { ipv4 | ipv6 | eb }以iptables的命令允许端口号,重启生效

[root@zcwyou ~]# firewall-cmd --direct -add-rule ipv4 filter INPUT 0 -p tcp --dport 9000 -j ACCEPT
[root@zcwyou ~]# firewall-cmd --reload
```

## 12.添加富规则:

```bash

12.1 允许192.168.122.0/24主机所有连接。
[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule family=“ipv4” source address=“192.168.122.0” accept’

12.2 每分钟允许2个新连接访问ftp服务。
[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule service name=ftp limit value=2/m accept’

12.3 同意新的 IP v4 和 IP v6 连接 FT P ,并使用审核每分钟登录一次。
[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule service name=ftp log limit value=“1/m” audit accept’

12.4 允许来自192.168.122.0/24地址的新 IPv4连接连接TFTP服务,并且每分钟记录一次。
[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule family=“ipv4” source address=“192.168.122.0/24” service name=ssh log prefix=“ssh” level=“notice” limit value=“3/m” accept’

12.5 丢弃所有icmp包
[root@zcwyou ~]# firewall-cmd --permanent --add-rich-rule=‘rule protocol value=icmp drop’

12.6 当使用source和destination指定地址时,必须有family参数指定ipv4或ipv6。如果指定超时,规则将在指定的秒数内被激活,并在之后被自动移除。
[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule family=ipv4 source address=192.168.122.0/24 reject’ --timeout=10

12.7 拒绝所有来自2001:db8::/64子网的主机访问dns服务,并且每小时只审核记录1次日志。
[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule family=ipv6 source address=“2001:db8::/64” service name=“dns” audit limit value=“1/h” reject’ --timeout=300

12.8 允许192.168.122.0/24网段中的主机访问ftp服务
[root@zcwyou ~]# firewall-cmd --permanent --add-rich-rule=‘rule family=ipv4 source address=192.168.122.0/24 service name=ftp accept’

12.9 转发来自ipv6地址1:2:3:4:6::TCP端口4011,到1:2:3:4:7的TCP端口4012

[root@zcwyou ~]# firewall-cmd --add-rich-rule=‘rule family=“ipv6” source address=“1:2:3:4:6::” forward-port to-addr=“1::2:3:4:7” to-port=“4012” protocol=“tcp” port=“4011”’

12.10 允许来自主机 192.168.0.14 的所有 IPv4 流量。
[root@zcwyou ~]# firewall-cmd --zone=public --add-rich-rule ‘rule family=“ipv4” source address=192.168.0.14 accept’

12.11 拒绝来自主机 192.168.1.10 到 22 端口的 IPv4 的 TCP 流量。
[root@zcwyou ~]# firewall-cmd --zone=public --add-rich-rule ‘rule family=“ipv4” source address=“192.168.1.10” port port=22 protocol=tcp reject’

12.12 查看富规则
[root@zcwyou ~]# firewall-cmd --list-rich-rules
```