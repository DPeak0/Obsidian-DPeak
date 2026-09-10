---
时间: 2026-08-24
---
### 8.24 Linux系统安装

### Linux操作系统介绍

操作系统是介于计算机硬件与上层软件之间的基础软件，为上层软件提供运行所需的基础环境。

发展历程：Unix → Minix → Linux

分支体系：

| 系列             | 软件包格式 | 包管理工具 | 发行系统                                  |
| ---------------- | ---------- | ---------- | ----------------------------------------- |
| RHEL（红帽）系列 | .rpm       | yum        | Redhat、CentOS、RockyLinux、EulerOS、麒麟 |
| Debian系列       | .deb       | apt        | Debian、Ubuntu                            |

### Linux系统安装

![image-20260824195406350](https://oss.bwihz.cn/ytpora/image-20260824195406350.png)



![image-20260824195415159](https://oss.bwihz.cn/ytpora/image-20260824195415159.png)



![image-20260824195426976](https://oss.bwihz.cn/ytpora/image-20260824195426976.png)

![image-20260824195632151](https://oss.bwihz.cn/ytpora/image-20260824195632151.png)



![image-20260824195651976](https://oss.bwihz.cn/ytpora/image-20260824195651976.png)

![image-20260824200711854](https://oss.bwihz.cn/ytpora/image-20260824200711854.png)

![image-20260824200724271](https://oss.bwihz.cn/ytpora/image-20260824200724271.png)

![image-20260824200743261](https://oss.bwihz.cn/ytpora/image-20260824200743261.png)



![image-20260824200814674](https://oss.bwihz.cn/ytpora/image-20260824200814674.png)

![image-20260824200921721](https://oss.bwihz.cn/ytpora/image-20260824200921721.png)

![image-20260824200950966](https://oss.bwihz.cn/ytpora/image-20260824200950966.png)





![image-20260824201009161](https://oss.bwihz.cn/ytpora/image-20260824201009161.png)



![image-20260824201535122](https://oss.bwihz.cn/ytpora/image-20260824201535122.png)



![image-20260824202431548](https://oss.bwihz.cn/ytpora/image-20260824202431548.png)

![image-20260824203603590](https://oss.bwihz.cn/ytpora/image-20260824203603590.png)

![image-20260824204118585](https://oss.bwihz.cn/ytpora/image-20260824204118585.png)

![image-20260824204235951](https://oss.bwihz.cn/ytpora/image-20260824204235951.png)

![image-20260824204319654](https://oss.bwihz.cn/ytpora/image-20260824204319654.png)

![image-20260824204327503](https://oss.bwihz.cn/ytpora/image-20260824204327503.png)

![image-20260824204341133](https://oss.bwihz.cn/ytpora/image-20260824204341133.png)

![image-20260824204411696](https://oss.bwihz.cn/ytpora/image-20260824204411696.png)

<img src="https://oss.bwihz.cn/ytpora/image-20260824204427238.png" alt="image-20260824204427238"  />



#### 拍摄快照

![image-20260824204612690](https://oss.bwihz.cn/ytpora/image-20260824204612690.png)

### 简单实操

需求1：创建一个个人姓名缩写的普通用户，设置密码为1

```bash
[root@dpeak ~]# passwd dbw
Changing password for user dbw.
New password:
BAD PASSWORD: The password is a palindrome
Retype new password:
passwd: all authentication tokens updated successfully.
```

需求2：通过root用户身份切换到这个创建的普通用户，再通过这个普通用户切换到devops用户，最后退出到root用户身份

```bash
[root@dpeak ~]# su - dbw
[dbw@dpeak ~]$ su - devops
Password:
[devops@dpeak ~]$ exit
logout
[dbw@dpeak ~]$ exit
logout
```

需求3：切换到你创建的用户身份，编辑file.txt文件，内容写上你的个人姓名缩写，最后保存退出

```bash
[root@dpeak ~]# su - dbw
[dbw@dpeak ~]$ vim file.txt
[dbw@dpeak ~]$ cat file.txt
dbw
```

