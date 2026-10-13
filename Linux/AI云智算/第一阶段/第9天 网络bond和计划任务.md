---
时间: 2026-09-07
---
# 网络bond（双网卡绑定）
## 基本概念
- 定义：将两张物理网卡绑定为一张虚拟网卡，对外呈现为一个逻辑接口。
- 目的：
	- 提升网络冗余性（高可用）
	- 增加网络带宽（负载均衡）
## bond级别

| bond级别 | bond名称                      | bond特点                                     | 对交换机要求                        |
| :----: | :-------------------------- | :----------------------------------------- | :---------------------------- |
|   0    | round-robin<br>（轮询）         | 按照设备顺序依次传输数据包。提供负载均衡和容错能力                  | 交换机需要配置trunking               |
|   1    | active-backup<br>（主备）       | 只有一个设备处理数据，当它宕机的时候就会由备份代替，仅提供容错能力          | 交换机不需要配置trunking              |
|   2    | load-balancing(xor)         | 根据MAC地址异或运算的结果来选择传输设备，提供负载均衡和容错能力          | 交换机需要配置trunking               |
|   3    | fault-tolerance (broadcast) | 通过全部设备来传输所有数据，提供容错能力                       | 交换机需要配置trunking               |
|   4    | lacp<br>（链路聚合）              | 通过创建聚合组来共享相同的传输速度，需要交换机也支持802.3ad模式，提供容错能力 | 需要交换机支持802.3d、交换机需要配置trunking |
|   5    | transmit load balancing     | 由负载最轻的网口发送，由当前使用的网口接收。提供负载均衡和容错能力          | 交换机不需要配置trunking              |
|   6    | adaptive load balancing     | 用负载最轻的网口进行发送和接收。提供负载均衡和容错能力。               | 交换机不需要配置trunking              |
## bond配置
- 准备两张相同网络模式的网卡设备


![](https://oss.bwihz.cn/%20PicGo/20260907101302649.png)

1、清空物理网卡的配置文件
```bash
[root@dpeak ~]# nmcli connection delete ens160
Connection 'ens160' (f2ed3493-4b6f-4bc0-8d92-6dce41dc5dd8) successfully deleted.
```
2、添加一个虚拟网卡br1
```bash
[root@dpeak ~]# nmcli connection add type bond mode active-backup  con-name br1 ifname br1 autoconnect yes 
Connection 'br1' (44510346-24bd-4cc6-92c7-ba9211bc6e3c) successfully added.
```
3、将两个物理网卡设备加入到虚拟网卡设备中
```bash
[root@dpeak ~]# nmcli connection add type bond-slave con-name ens160-br1   ifname ens160 master br1 autoconnect yes 
Connection 'ens160-br1' (e22ef16c-b80b-4327-8fed-a7136a185690) successfully added.
[root@dpeak ~]# nmcli connection add type bond-slave con-name ens192-br1   ifname ens192 master br1 autoconnect yes 
Connection 'ens192-br1' (98029f3a-cd27-4c91-93e6-f5ae8bddb6ff) successfully added.
```
4、修改虚拟网卡的网络信息
```bash
[root@dpeak ~]# nmcli con modify br1 ipv4.method manual ipv4.addresses 192.168.200.100/24 ipv4.dns 223.5.5.5 ipv4.gateway 192.168.200.2
[root@dpeak ~]# nmcli connection up br1 
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)

```
5、查看虚拟网卡br1的详细信息
```bash
[root@dpeak ~]# cat /proc/net/bonding/br1 
Ethernet Channel Bonding Driver: v4.18.0-305.3.1.el8_4.x86_64

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: ens160
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0
Peer Notification Delay (ms): 0

Slave Interface: ens160
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:a5:13:f3
Slave queue ID: 0

Slave Interface: ens192
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:a5:13:fd
Slave queue ID: 0

```

## 三种网络模式
### 仅主机模式
- **特点**：纯局域网，无上行链路，不能访问外网。
![](https://oss.bwihz.cn/%20PicGo/20260907193726982.png)

### NAT模式
- **特点**：虚拟机通过宿主机物理网卡进行网络地址转换，可以上网。外部用户无法主动访问虚拟机，只有宿主机可以。
![](https://oss.bwihz.cn/%20PicGo/20260907193737852.png)

### 桥接模式
- **特点**：虚拟机网卡直接桥接到宿主机物理网卡（相当于插入同一交换机），与 NAT 一样可以上网。桥接模式下，外部网络可直接访问该虚拟机。
  ![](https://oss.bwihz.cn/%20PicGo/20260907194102454.png)

- **配置注意**：需在 VMware 中指定桥接到哪块宿主机物理网卡（如无线网卡或有线网卡）。![](https://oss.bwihz.cn/%20PicGo/20260907194042157.png)
#### 配置Linux系统的桥接网络
![](https://oss.bwihz.cn/%20PicGo/20260907194226996.png)
把ens192作为桥接模式的网卡，作为虚拟交换机进行使用
配置流程：
1. 删除掉ens192对应的网卡配置文件
2. 添加一张虚拟网卡br0，将br0虚拟网卡桥接到ens192（把ens192当作了虚拟交换机），br0虚拟网卡就是我们的网络模式
3. 创建KVM虚拟机，设置网络模式为br0（桥接模式）

# 系统主机名修改、端口
## `hostnamectl`
![[2.Linux命令选项解析#hostnamectl 管理系统主机名]]
## netstat
![[2.Linux命令选项解析#`netstat`]]

# 计划任务

## `at`：一次性计划任务
![[2.Linux命令选项解析#`at` 一次性计划任务]]

## `crontab` 周期性计划任务
![[2.Linux命令选项解析#`crontab` 周期性计划任务]]