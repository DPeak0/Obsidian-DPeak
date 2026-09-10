# Debian 搭建DHCP服务

## 服务端

### 1.安装软件包

```bash
root@debian:~# apt install -y isc-dhcp-server
```

### 2.修改配置文件

```bash
root@debian:~# vim /etc/default/isc-dhcp-server
DHCPDv4_CONF=/etc/dhcp/dhcpd.conf		##DHCP配置文件位置
INTERFACESv4="ens33"	##绑定网卡
#INTERFACESv6=""
root@debian:~# vim /etc/dhcp/dhcpd.conf 
# A slightly different configuration for an internal subnet.
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.30 192.168.100.40;		##dhcp地址池
  option domain-name-servers 192.168.100.100;		##dns
  option domain-name "dbw.com";			##域
  option routers 192.168.100.254;		##网关
#  option broadcast-address 10.5.5.31;		##广播地址
  default-lease-time 600;
  max-lease-time 7200;
##############################
  host WSC-Client1 {				
  hardware ethernet 00:0c:29:8D:4F:B1;
  fixed-address 192.168.100.33;}
###############################绑定指定客户端的MAC地址使其固定分配IP
}
```

### 3.重启服务

```bash
root@debian:~# systemctl restart isc-dhcp-server
```

## 客户端

### 修改dhcp模式

```bash
root@debian:~# vim /etc/network/interfaces
auto ens33
iface ens33 inet dhcp
```

<hr>
