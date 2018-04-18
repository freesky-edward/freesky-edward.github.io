---
layout: post
title:  "linux配置cifs共享"
lang: zh
date: 2018-04-10
---

### 准备

```shell
sudo apt-get update
sudo apt-get upgrade
```

### 安装samba

安装samba提供共享服务a

```shell
sudo apt-get install samba samba-common
```

修改samba配置文件

```shell
sudo vim /etc/samba/smb.conf
```

1.增加用户认证配置——在下面内容处增加sercurity = user

```
#### Debugging/Accounting ####

# This tells Samba to use a separate log file for each machine
# that connects
   log file = /var/log/samba/log.%m

# Cap the size of the individual log files (in KiB).
   max log size = 1000
   security = user
```

2. 增加共享路径配置——在文件最后增加：

```
[openstack] #访问路径
    comment = this is my devstack work folder
    path = /opt/openstack
    browseable = yes
    writable = yes

[projects]  #访问路径
    comment = this is openstack projects source folder
    path = /opt/stack
    browseable = yes
    writable = yes
```

重启服务

```
sudo service smbd restart.
```

创建共享用户

```
smbpasswd -a <name> #回车后输入密码，其中<name>需要是系统已有账户
```

### 其它问题

报错：指定的网络文件夹目前是以其他用户名和密码进行映射的。要用其他用户名和密码进行连接，首先请断开所有现有的连接到网络共享的映射

原因：被映射的网络共享文件夹所在的机器给不同的共享文件夹设置了不同的用户访问权限，而目前连接的机器与被映射的机器已经用另一个用户建立了连接。

解决办法：运行 net use * /delete. 输入Y后，将原映射的路径删除掉用再连接
