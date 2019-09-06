---
layout:     post
title:      Centos7_配置SFTP
subtitle:   SFTP服务部署
date:       2019-09-06
author:     Baru
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Linux
    - SFTP
    - 运维
---


# Centos7 配置SFTP

[TOC]

## 环境检查

### 验证ssh版本

~~~shell
ssh -V
~~~

> **NOTE:** `openssh`的版本必须大于`4.8p1`，如果低于这个版本需要升级

### 验证`Selinux`

~~~shell
getenforce
## 如果不是：Disabled,请使用如下命令关闭:
setenforce 0
~~~

## 创建组：

创建sftp组

~~~shell
#创建
groupadd sftp
#检查
cat /etc/group
~~~

## 创建用户

~~~shell
#创建一个sftp用户mysftp并加入到创建的sftp组中
useradd -g sftp -s /bin/false mysftp  
#同时修改mysftp用户的密码
passwd mysftp 
~~~

## 创建用户家目录

~~~shell
#创建目录
mkdir -p /data/sftp/mysftp 
#指定其为mysftp用户家目录
usermod -d /data/sftp/mysftp mysftp
~~~

## 配置ssh文件

### 备份sshd_config文件

~~~shell
cp /etc/ssh/sshd_config /etc/ssh/sshd_config_bak201909061013
~~~

### 编辑配置文件

~~~shell
vim /etc/ssh/sshd_config
~~~

变更如下：

~~~shell
#注释：Subsystem      sftp    /usr/libexec/openssh/sftp-server，修改如下：
# Subsystem      sftp    /usr/libexec/openssh/sftp-server
#文末添加以下内容：
Subsystem       sftp    internal-sftp    
Match Group sftp    
ChrootDirectory /data/sftp/%u    
ForceCommand    internal-sftp    
AllowTcpForwarding no    
X11Forwarding no  
~~~

## 设置Chroot目录权限

~~~shell
chown root:sftp /data/sftp/mysftp  
chmod 755 /data/sftp/mysftp
~~~

## 新建上传目录

新建一个目录供stp用户mysftp上传文件；

这个目录所有者为mysftp所有组为sftp；

所有者有写入权限，所有组无写入权限。

~~~shell
mkdir /data/sftp/mysftp/Suning  
chown mysftp:sftp /data/sftp/mysftp/Suning 
chmod 755 /data/sftp/mysftp/Suning
~~~

### 重启sshd

~~~shell
systemctl restart sshd.service
~~~

## 验证

~~~shell
sftp mysftp@xxx.xxx.xxx.xxx(IP)
~~~



