| 服务器   | 公网 IP                  | 私网 IP               |
| ----- | ---------------------- | ------------------- |
| node1 | VMnet8： 192.168.200.10 | VMnet1：192.168.0.10 |
| node2 | 无                      | 192.168.0.20        |
| node3 | 无                      | 192.168.0.30        |

```bash
[root@node1 ~]# hostnamectl set-hostname node1
[root@node1 ~]# echo -e '192.168.0.10 node1\n192.168.0.20 node2\n192.168.0.30 node3' >> /etc/hosts
[root@node1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3

[root@node1 ~]# mkdir /etc/yum.repos.d/bak
[root@node1 ~]# mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak
[root@node1 ~]# cat > /etc/yum.repos.d/dvd.repo <<EOF
> [BaseOS]
> name=BaseOS
> baseurl=file:///media/BaseOS
> gpgcheck=0
>
> [AppStream]
> name=AppStream
> baseurl=file:///media/AppStream
> gpgcheck=0
> EOF
[root@node1 ~]# mount /dev/sr0 /media/
 yum makecachemount: /media: WARNING: device write-protected, mounted read-only.
[root@node1 ~]# yum clean all ; yum makecache
13 files removed
BaseOS                                                 37 MB/s | 2.5 MB     00:00
AppStream                                              31 MB/s | 7.1 MB     00:00
Last metadata expiration check: 0:00:01 ago on Mon 19 Oct 2026 08:17:06 PM CST.
Metadata cache created.

```

