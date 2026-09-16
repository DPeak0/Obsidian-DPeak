
> [!abstract] 分享目标
> 
> - 理解数据从设备到互联网的路径
>     
> - 掌握光猫、路由器、交换机、MAC、IP、子网掩码、DHCP、DNS、网关等基础概念
>     
> - 会查看与配置网卡 IP：`ifconfig`、`ip`、`nmtui`、`nmcli`
>     
> - 会用 `ping` 做连通性测试
>     
> - 会用 `ss` / `netstat` 查看端口与连接
>     
> - 理解主机名、DNS 解析与常见排查思路
>     
---
## 大纲
- [[#1. 网络基础]]
- [[#2. 查看网络信息]]
- [[#3. IP 地址配置与网卡管理]]
- [[#4. ping 连通性测试]]
- [[#5. ss / netstat 端口查看]]
- [[#6. 主机名与 DNS 解析]]
- [[#7. 命令速查表]]
- [[#8. 常见问题与排查]]

---
## 1. 网络基础

### 1.1 网络接入拓扑：数据流向
**有线接入路径：**
> 电脑（有线网卡） → 网线 → 路由器 → 网线 → 光猫 → 光纤 → 运营商网络（互联网）

**无线接入路径：**
> 手机/笔记本（无线网卡） → Wi-Fi 信号 → 路由器 → 网线 → 光猫 → 光纤 → 运营商网络（互联网）

> [!note] 讲解重点
> 
> - 光猫是运营商提供的“上网大门”，负责光电信号转换。
>     
> - 路由器负责网络中转与分发，是家庭网络核心。
>     
> - 交换机只扩展有线端口，不具备路由功能。
>     

---

### 1.2 核心网络设备功能

|设备名称|核心功能|备注|
|---|---|---|
|**光猫**|**光电信号转换**，把光纤中的光信号转换为路由器/电脑能识别的电信号|由运营商提供，需要工作人员上门开通配置|
|**路由器**|**网络中转与分发**，通过有线和无线方式分发给多个设备|性能通常比光猫自带无线更强，是家庭网络核心|
|**交换机**|**扩展有线端口**，连接多台有线上网设备|可理解为路由器有线接口的“扩展坞”，不具备路由功能|

---

### 1.3 网络通信关键概念
#### 1.3.1 MAC 地址
- **定义**：每个网卡出厂时烧录的唯一硬件地址，类似“身份证号”。
- **作用**：在同一个局域网内，精确标识和寻找具体物理设备。
- **查看命令**：
```bash
ip link show
ifconfig
```
#### 1.3.2 IP 地址
- **定义**：计算机在网络上的逻辑标识，类似“门牌号”。
- **计算机必须配置 IP 地址才能与其他设备通信。**

**版本：**

| 版本   | 说明                    | 示例                         |                              |
| ---- | --------------------- | -------------------------- | ---------------------------- |
| IPv4 | 最常见，地址数量有限            | `192.168.1.1`              | 2^32=4294967296 <br>40亿个IP地址 |
| IPv6 | 解决 IPv4 地址枯竭问题，地址数量极大 | `fe80::20c:29ff:fe0d:6202` |                              |
|      |                       |                            |                              |

**按访问范围分类：**

|类型|说明|
|---|---|
|公网地址|全球互联网唯一且可访问|
|私网地址|仅在内部局域网使用，互联网不能直接访问|

常见私网地址段：

- `10.0.0.0/8`
    
- `172.16.0.0/12`
    
- `192.168.0.0/16`
    

**NAT 技术：**

- 作用：将私网 IP 转换为公网 IP。
    
- 允许多个设备共享一个公网 IP 上网，缓解 IPv4 地址短缺。
    

#### 1.3.3 子网掩码

- **定义**：用于划分 IP 地址的网络号和主机号，确定设备所在“网段”。
    
- **作用**：判断两台设备是否在同一个局域网内。
    
- 常见写法：
    
    - `255.255.255.0`
        
    - 等价于 `/24`
        
    - 表示该网段有 256 个 IP 地址，其中可用主机地址为 254 个。
        

#### 1.3.4 DHCP

- **定义**：动态主机配置协议。
    
- **作用**：自动给网络设备分配 IP 地址、子网掩码、网关、DNS 等参数。
    
- 优点：无需手动配置。
    

#### 1.3.5 DNS

- **定义**：互联网的“电话簿”，将域名转换为 IP 地址。
    
- 示例：
    
    - `www.baidu.com` → `110.242.68.66`
        

**解析类型：**

|类型|方向|用途|
|---|---|---|
|正向解析|域名 → IP|最常用|
|反向解析|IP → 域名|邮件服务器反垃圾邮件等|

**公共 DNS 举例：**

- `223.5.5.5`、`223.6.6.6` 阿里云 DNS
    
- `114.114.114.114` 国内通用 DNS
    
- `8.8.8.8` 谷歌 DNS
    

#### 1.3.6 网关

- **定义**：一个网络连接到另一个网络的“关口”。
    
- **作用**：如果两台设备不在同一个网段，通信必须通过网关转发。
    
- 网关通常是路由器。
    

---

## 2. 查看网络信息

### 2.1 `ifconfig`

bash

[root@xym ~]# ifconfig
ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.128  netmask 255.255.255.0  broadcast 192.168.2.255
        inet6 fe80::20c:29ff:fe0d:6202  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:0d:62:02  txqueuelen 1000  (Ethernet)
        RX packets 87124  bytes 121841725 (116.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 30752  bytes 1681374 (1.6 MiB)
        TX errors 0  dropped 0  overruns 0  carrier 0  collisions 0
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:7c:56:07  txqueuelen 1000  (Ethernet)

**常见网卡说明：**

|网卡|说明|
|---|---|
|`lo`|本地回环网卡，固定 IP `127.0.0.1`，无法被外部主机访问|
|`virbr0`|虚拟网卡，固定 IP `192.168.122.1`，主要作为 KVM 虚拟机网关|
|`ens160`|物理网卡设备，在 VMware 虚拟机中就是虚拟机网卡|

> [!warning] 注意  
> `ifconfig` 属于 `net-tools` 包，新系统可能默认未安装。  
> 推荐使用 `ip` 命令替代。

---

### 2.2 `ip`

bash

ip address show
ip addr
ip a

示例：

bash

[root@xym ~]# ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:0d:62:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.128/24 brd 192.168.2.255 scope global dynamic noprefixroute ens160
       valid_lft 1567sec preferred_lft 1567sec
3: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:0d:62:0c brd ff:ff:ff:ff:ff:ff
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:7c:56:07 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0

> [!tip] 如果一张网卡存在多个 IP 地址  
> 使用 `ip address show` 或 `ip addr` 查看，比 `ifconfig` 更清晰。

---

### 2.3 网卡命名规则详解

#### 传统命名方式：`ethX`

- 格式：`eth0`、`eth1`、`eth2`
    
- 优点：命名有规律，方便自动化工具。
    
- 缺点：根据开机时加载设备顺序命名，可能变化。
    

#### 基于固件特征的命名方式：`ens160`、`ens192`、`eno166667` 等

|字段|含义|
|---|---|
|`en`|Ethernet 以太网设备|
|`s`|支持热插拔|
|`o`|板载设备，网卡镶嵌在主板上|
|`wl`|无线网卡设备|
|`p`|PCIe 插槽设备|

---

## 3. IP 地址配置与网卡管理

### 3.1 临时修改 IP

> [!warning] 临时修改  
> 重启或重新激活网卡后会失效，适合测试。

bash

# 使用 ifconfig 临时设置 IP
ifconfig ens192 192.168.2.100/24
# 使用 ip 添加 IP
ip address add 192.168.2.102/24 dev ens192
# 删除 IP
ip addr del 192.168.2.102/24 dev ens192
# 启用/禁用网卡
ip link set ens192 up
ip link set ens192 down

---

### 3.2 永久修改：通过配置文件

> [!note] RHEL/CentOS 传统路径  
> `/etc/sysconfig/network-scripts/ifcfg-*`  
> RHEL 9 开始也常用 NetworkManager keyfile：`/etc/NetworkManager/system-connections/`。  
> 实际生产建议优先使用 `nmcli` / `nmtui`。

示例：`/etc/sysconfig/network-scripts/ifcfg-ens160`

bash

TYPE=Ethernet
BOOTPROTO=static
NAME=ens160
DEVICE=ens160
ONBOOT=yes
IPADDR=192.168.2.128
PREFIX=24
# 或者 NETMASK=255.255.255.0
GATEWAY=192.168.2.1
DNS1=223.5.5.5
DNS2=114.114.114.114

如果是 DHCP：

bash

BOOTPROTO=dhcp
ONBOOT=yes

重载配置：

bash

nmcli connection reload
nmcli connection up ens160
# 或
systemctl restart NetworkManager

---

### 3.3 永久修改：`nmtui`

`nmtui` 是伪图形化网络管理工具，适合不熟悉命令行的场景。

bash

nmtui

常见菜单：

- Edit a connection：编辑连接
    
- Activate a connection：激活连接
    
- Set system hostname：设置主机名
    

> [!tip] 分享演示建议  
> 现场打开 `nmtui`，展示如何修改 IPv4 地址、网关、DNS，然后保存并激活。

---

### 3.4 永久修改：`nmcli`

bash

# 查看设备状态
nmcli device status
# 查看连接
nmcli connection show
# 查看指定连接详情
nmcli connection show ens160

**配置静态 IP：**

bash

nmcli connection modify ens160 \
  ipv4.method manual \
  ipv4.addresses 192.168.2.100/24 \
  ipv4.gateway 192.168.2.1 \
  ipv4.dns "223.5.5.5 114.114.114.114"
nmcli connection up ens160

**配置 DHCP：**

bash

nmcli connection modify ens160 ipv4.method auto
nmcli connection up ens160

**修改主机名：**

bash

nmcli general hostname web01

---

### 3.5 修改网卡设备命名方式

#### 方式一：修改 GRUB

bash

vim /etc/default/grub

找到：

bash

GRUB_CMDLINE_LINUX="rhgb quiet"

改为：

bash

GRUB_CMDLINE_LINUX="rhgb quiet net.ifnames=0 biosdevname=0"

执行：

bash

grub2-mkconfig -o /boot/grub2/grub.cfg

然后重启。

#### 方式二：修改 BLS 条目

进入 `/boot/loader/entries/`，找到一个不是 rescue 的文件：

bash

/boot/loader/entries/ffffffffffffffffffffffffffffffff-5.14.0-70.13.1.el9_0.x86_64.conf

在 `options` 行后面添加：

bash

net.ifnames=0 biosdevname=0

> [!danger] 生产环境慎用  
> 修改网卡命名后，网卡可能从 `ens160` 变成 `eth0`。  
> 需要同步修改网卡配置文件，否则网络可能起不来。

---

## 4. ping 连通性测试

### 4.1 基本用法

bash

ping -c 4 www.baidu.com
ping -c 4 192.168.2.1
ping -c 4 223.5.5.5

常用参数：

|参数|作用|
|---|---|
|`-c 4`|发送 4 个包后停止|
|`-i 1`|间隔 1 秒|
|`-W 2`|等待超时时间|
|`-s 1400`|指定包大小|
|`-I ens160`|指定网卡/源接口|
|`-6`|使用 IPv6|

示例：

bash

ping -c 4 -I ens160 223.5.5.5
ping -6 -c 4 ::1

---

### 4.2 输出解读

bash

64 bytes from 223.5.5.5: icmp_seq=1 ttl=117 time=8.32 ms

|字段|含义|
|---|---|
|`icmp_seq`|ICMP 包序号|
|`ttl`|生存时间|
|`time`|往返延迟|
|`packet loss`|丢包率|

---

### 4.3 连通性排查顺序

> [!tip] 推荐顺序
> 
> 1. `ping 127.0.0.1`：本机协议栈是否正常
>     
> 2. `ping 本机IP`：本机网卡是否正常
>     
> 3. `ping 网关`：局域网到网关是否通
>     
> 4. `ping 223.5.5.5`：公网 IP 是否通
>     
> 5. `ping www.baidu.com`：DNS 是否正常
>     

常见结果：

|现象|可能原因|
|---|---|
|`Destination Host Unreachable`|路由/网关/链路问题|
|`Request timeout`|防火墙、目标不可达、丢包|
|`Unknown host`|DNS 解析失败|
|`ping 通 IP，ping 不通域名`|DNS 问题|

---

## 5. ss / netstat 端口查看

### 5.1 `ss` 常用命令

`ss` 是 `netstat` 的现代替代品，速度更快。

bash

# 查看所有 TCP/UDP 监听端口
ss -tulnp
# 只看 TCP 监听
ss -lntp
# 查看所有 TCP 连接
ss -tanp
# 查看连接统计
ss -s

参数说明：

|参数|含义|
|---|---|
|`-t`|TCP|
|`-u`|UDP|
|`-l`|listening，监听状态|
|`-n`|不解析服务名，显示数字端口|
|`-p`|显示进程|
|`-a`|所有连接|

---

### 5.2 `netstat` 常用命令

bash

netstat -tulnp
netstat -anp

> [!warning] 注意  
> `netstat` 属于 `net-tools` 包，新系统可能默认未安装。  
> 推荐优先使用 `ss`。

---

### 5.3 输出字段

bash

State      Recv-Q Send-Q Local Address:Port   Peer Address:Port   Process
LISTEN     0      128    0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=1234,fd=3))

|字段|含义|
|---|---|
|State|连接状态|
|Recv-Q|接收队列|
|Send-Q|发送队列|
|Local Address:Port|本地地址和端口|
|Peer Address:Port|对端地址和端口|
|Process|进程信息|

常见状态：

|状态|说明|
|---|---|
|`LISTEN`|正在监听|
|`ESTABLISHED`|已建立连接|
|`TIME_WAIT`|等待关闭|
|`CLOSE_WAIT`|被动关闭等待|

---

### 5.4 常用端口

|端口|服务|
|---|---|
|22|SSH|
|80|HTTP|
|443|HTTPS|
|53|DNS|
|3306|MySQL|
|6379|Redis|

示例：

bash

ss -lntp | grep :22
ss -lntp | grep :80
lsof -i :80

---

## 6. 主机名与 DNS 解析

### 6.1 主机名管理

查看主机名：

bash

hostname
hostnamectl
cat /etc/hostname

永久修改：

bash

hostnamectl set-hostname web01

或：

bash

nmcli general hostname web01

临时修改：

bash

hostname web01

> [!tip] 建议  
> 修改后重新登录或执行 `bash`，让提示符显示新主机名。

---

### 6.2 DNS 配置文件

#### `/etc/resolv.conf`

记录 DNS 地址：

bash

nameserver 223.5.5.5
nameserver 114.114.114.114
search example.com

> [!warning] 注意  
> 在 NetworkManager 管理的系统中，`/etc/resolv.conf` 可能被自动覆盖。  
> 推荐通过 `nmcli` 配置 DNS。

bash

nmcli connection modify ens160 ipv4.dns "223.5.5.5 114.114.114.114"
nmcli connection modify ens160 ipv4.ignore-auto-dns yes
nmcli connection up ens160

#### `/etc/hosts`

本地静态解析，优先级通常高于 DNS：

bash

127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain
192.168.2.128 web01

---

### 6.3 DNS 解析命令

bash

getent hosts www.baidu.com
nslookup www.baidu.com
dig www.baidu.com +short
host www.baidu.com

反向解析：

bash

dig -x 110.242.68.66

查看系统 DNS 状态：

bash

resolvectl status

> [!note] 正向解析与反向解析
> 
> - 正向解析：域名 → IP，最常用。
>     
> - 反向解析：IP → 域名，常用于邮件服务器反垃圾邮件。
>     

---

### 6.4 DNS 排查思路

bash

cat /etc/resolv.conf
ping -c 2 223.5.5.5
dig @223.5.5.5 www.baidu.com
getent hosts www.baidu.com

> [!example] 演示流程
> 
> 1. `ip addr` 查看 IP
>     
> 2. `ping 网关`
>     
> 3. `ping 223.5.5.5`
>     
> 4. `dig www.baidu.com`
>     
> 5. `ss -tulnp` 查看 22/80 端口
>     
> 6. `hostnamectl set-hostname` 修改主机名
>     

---

## 7. 命令速查表

|目的|命令|
|---|---|
|查看 IP|`ip addr show` / `ifconfig`|
|查看路由|`ip route`|
|临时加 IP|`ip addr add 192.168.2.100/24 dev ens160`|
|删除 IP|`ip addr del 192.168.2.100/24 dev ens160`|
|启用/禁用网卡|`ip link set ens160 up/down`|
|图形化配置网络|`nmtui`|
|命令行配置网络|`nmcli`|
|查看连接|`nmcli connection show`|
|连通性测试|`ping -c 4 目标`|
|查看监听端口|`ss -tulnp`|
|查看 TCP 连接|`ss -tanp`|
|查看 DNS|`cat /etc/resolv.conf`|
|解析域名|`dig` / `nslookup` / `getent hosts`|
|查看主机名|`hostname` / `hostnamectl`|
|修改主机名|`hostnamectl set-hostname 新名字`|

---

## 8. 常见问题与排查

> [!faq] 常见问题
> 
> - `ifconfig` 找不到：安装 `net-tools`，或改用 `ip`。
>     
> - 修改 IP 不生效：检查 `ONBOOT=yes`，执行 `nmcli connection reload` 或 `nmcli connection up`。
>     
> - DNS 不生效：`/etc/resolv.conf` 可能被 NetworkManager 覆盖，推荐用 `nmcli` 配置。
>     
> - 端口被占用：`ss -lntp | grep :端口`。
>     
> - `ping` 通 IP 但 ping 不通域名：DNS 解析问题。
>     
> - 不同网段无法通信：检查网关和路由。
>     
> - 网卡改名后网络异常：同步修改网卡配置文件，或恢复 `net.ifnames` 设置。
>     

---

## 9. 分享总结

> [!success] 一句话总结  
> 网络管理核心就是：**看懂拓扑、认准 IP、配好网卡、测通链路、查清端口、解析域名。**

重点掌握：

- 数据流向：设备 → 路由器 → 光猫 → 运营商
    
- 核心概念：MAC、IP、子网掩码、DHCP、DNS、网关、NAT
    
- 网卡管理：`ifconfig`、`ip`、`nmtui`、`nmcli`
    
- 连通性：`ping` 分层排查
    
- 端口连接：`ss -tulnp` / `netstat -tulnp`
    
- 主机名与 DNS：`hostnamectl`、`/etc/resolv.conf`、`/etc/hosts`、`dig`、`getent`
    

相关笔记：

- [[网络管理]]
    
- [[查看网络信息]]
    
- [[Linux命令选项解析]]
    
- [[Linux配置文件解析]]
    

#Linux #网络管理 #运维基础 #技术分享