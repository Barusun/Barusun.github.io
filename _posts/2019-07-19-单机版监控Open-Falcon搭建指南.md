---
layout:     post
title:      单机版监控Open-Falcon搭建指南
subtitle:   实现邮件告警
date:       2019-07-19
author:     Baru
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - Linux
    - 监控Open-Falcon
    - 运维
---

# 单机版监控Open-Falcon搭建指南

[TOC]

## 环境准备

### 操作系统

~~~shell
[root@linux-node2 redis-cluster]# cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core)
~~~

### 安装git

~~~shell
yum -y install git
~~~



### 安装redis

>  使用自己的方法安装好一个单节点redis即可

检查服务启动是否正常

~~~shell
[root@linux-node2 redis-cluster]# systemctl status redis_6379.service
● redis_6379.service - Redis In-Memory Data Store
   Loaded: loaded (/etc/systemd/system/redis_6379.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2019-07-18 16:09:53 CST; 7s ago
  Process: 16683 ExecStart=/usr/local/redis-cluster/6379/redis-server /usr/local/redis-cluster/6379/redis.conf (code=exited, status=0/SUCCESS)
 Main PID: 16684 (redis-server)
    Tasks: 4
   Memory: 6.3M
   CGroup: /system.slice/redis_6379.service
           └─16684 /usr/local/redis-cluster/6379/redis-server *:6379

Jul 18 16:09:53 linux-node2.kubernetes.com systemd[1]: Starting Redis In-Memory Data Store...
Jul 18 16:09:53 linux-node2.kubernetes.com systemd[1]: Started Redis In-Memory Data Store.


~~~

检查端口启动是否正常

~~~shell
[root@linux-node2 redis-cluster]# ss -ntlp|grep 6379
LISTEN     0      128          *:6379                     *:*                   users:(("redis-server",pid=16684,fd=7))
LISTEN     0      128         :::6379                    :::*                   users:(("redis-server",pid=16684,fd=6))
~~~

### 安装mysql

~~~shell
wget http://repo.mysql.com//mysql57-community-release-el7-8.noarch.rpm
yum -y install mysql57-community-release-el7-8.noarch.rpm
yum -y install mysql-community-server.x86_64
systemctl start mysqld.service
systemctl status mysqld.service
systemctl enable mysqld.service
#查看默认密码
cat /var/log/mysqld.log | grep 'password'
#连接数据库修改密码
mysql -uroot -p
#修改用户密码
set password for 'root'@'localhost'=password('xxxxxxxxxx');
#刷新权限
flush privileges;
~~~

### 初始化表结构

~~~shell
cd /tmp/ && git clone https://github.com/open-falcon/falcon-plus.git 
cd /tmp/falcon-plus/scripts/mysql/db_schema/
mysql -h 127.0.0.1 -u root -p < 1_uic-db-schema.sql
mysql -h 127.0.0.1 -u root -p < 2_portal-db-schema.sql
mysql -h 127.0.0.1 -u root -p < 3_dashboard-db-schema.sql
mysql -h 127.0.0.1 -u root -p < 4_graph-db-schema.sql
mysql -h 127.0.0.1 -u root -p < 5_alarms-db-schema.sql
rm -rf /tmp/falcon-plus/
#如果数据库设置密码的话，需要密码
~~~

### 下载二进制文件

下载二进制文件：[Open-Falcon二进制下载](https://github.com/open-falcon/falcon-plus/releases/download/v0.3/open-falcon-v0.3.tar.gz)  版本为V0.3



## 安装后端

### 创建工作目录

~~~shell
export FALCON_HOME=/opt/falcon
export WORKSPACE=$FALCON_HOME/open-falcon
mkdir -p $WORKSPACE
~~~

### 解压二进制包

~~~shell
tar -xzvf open-falcon-v0.3.tar.gz -C $WORKSPACE
~~~

### 启动后端组件

#### 1.确认配置文件中数据库账号密码与实际相同

~~~shell
cd $WORKSPACE
grep -Ilr 3306  ./ | xargs -n1 -- sed -i 's/root:/root:xxxxxxxxxx/g'
~~~

#### 2.启动

~~~shell
cd $WORKSPACE
./open-falcon start

# 检查所有模块的启动状况
./open-falcon check
~~~

##### 提示如下：

~~~shell
# ./open-falcon check
        falcon-graph         UP          124351 
          falcon-hbs         UP          124364 
        falcon-judge         UP          124377 
     falcon-transfer         UP          124386 
       falcon-nodata         UP          124397 
   falcon-aggregator         UP          124402 
        falcon-agent         UP          124422 
      falcon-gateway         UP          124458 
          falcon-api         UP          124469 
        falcon-alarm         UP          124485 
~~~

#### 命令

~~~shell
# ./open-falcon [start|stop|restart|check|monitor|reload] module
./open-falcon start agent

./open-falcon check
        falcon-graph         UP           53007
          falcon-hbs         UP           53014
        falcon-judge         UP           53020
     falcon-transfer         UP           53026
       falcon-nodata         UP           53032
   falcon-aggregator         UP           53038
        falcon-agent         UP           53044
      falcon-gateway         UP           53050
          falcon-api         UP           53056
        falcon-alarm         UP           53063

For debugging , You can check $WorkDir/$moduleName/log/logs/xxx.log
~~~

## 安装前端

### 准备工作目录

**安装的单机版，上面已经创建**

~~~shell
export FALCON_HOME=/opt/falcon
export WORKSPACE=$FALCON_HOME/open-falcon
mkdir -p $WORKSPACE
~~~

### 克隆前端组件代码

~~~shell
cd $WORKSPACE
git clone https://github.com/open-falcon/dashboard.git
~~~

### 安装依赖包

~~~shell
yum install -y python-virtualenv
yum install -y python-devel
yum install -y openldap-devel
yum install -y mysql-devel
yum groupinstall -y "Development tools"


cd $WORKSPACE/dashboard/
virtualenv ./env

./env/bin/pip install -r pip_requirements.txt -i https://pypi.douban.com/simple
~~~

### 修改dashboard配置

~~~shell
dashboard的配置文件为： 'rrd/config.py'，请根据实际情况修改

## API_ADDR 表示后端api组件的地址
API_ADDR = "http://127.0.0.1:8080/api/v1" 

## 根据实际情况，修改PORTAL_DB_*, 默认用户名为root，默认密码为""
## 根据实际情况，修改ALARM_DB_*, 默认用户名为root，默认密码为""
~~~

### 启动

#### 生产环境启动

~~~shell
启动
bash control start
停止
bash control stop
查看日志
bash control tail
~~~

> ```
> open http://127.0.0.1:8081 in your browser.
> ```

#### 开发者模式启动

~~~shell
./env/bin/python wsgi.py

open http://127.0.0.1:8081 in your browser.
~~~



>**NOTE:**
>
>```
>dashbord没有默认创建任何账号包括管理账号，需要你通过页面进行注册账号。
>想拥有管理全局的超级管理员账号，需要手动注册用户名为root的账号（第一个帐号名称为root的用户会被自动设置为超级管理员）。
>超级管理员可以给普通用户分配权限管理。
>
>小提示：注册账号能够被任何打开dashboard页面的人注册，所以当给相关的人注册完账号后，需要去关闭注册账号功能。只需要去修改api组件的配置文件cfg.json，将signup_disable配置项修改为true，重启api即可。当需要给人开账号的时候，再将配置选项改回去，用完再关掉即可。
>```
>
>

## 安装mail组件

### 准备二进制包

​	需要在网上找这个`falcon-mail-provider`组件并下载，不过网上有的mail组件有问题。就算配置好，可能也发不出去。这个需要注意。

### 配置

​	该组件也可统一放在`WORKSPACE`目录（/opt/falcon/open-falcon）下

~~~json
vim mail-provider/cfg.json
{
    "debug": true,
    "http": {
        "listen": "0.0.0.0:4000",
        "token": ""
    },
    "smtp": {
        "addr": "smtp.qq.com:465",
        "username": "baru_sun@qq.com",
        "password": "xxxxx",
        "from": "baru_sun@qq.com",
        "tls":true,
        "anonymous":false,
        "skipVerify":true
    }
}
~~~

> 以上为QQ邮箱设置，注意QQ邮箱密码需要授权码才可登陆

### 启动

~~~shell
cd mail-provider/
#启动
./control start
#停止
./control stop
#日志
./control tail
~~~

### 测试

~~~shell
curl http://127.0.0.1:4000/sender/mail -d "tos=baru_sun@qq.com&subject=OpenFalcon测试&content=Hello,xiaoxiao"
~~~

### 配置Alarm

​	将Alarm的`config/cfg.json`文件配置修改如下

~~~json
"mail": "http://127.0.0.1:10086/mail"
>"mail": "http://127.0.0.1:4000/sender/mail"
~~~

#### 重启Alarm

~~~shell
./open-falcon restart alarm
~~~

## 总结

​	以上为单机版部署步骤，在这个平台之上就可以实现告警设置了。dashboard配置参考[官网手册进行配置](<http://book.open-falcon.com/zh_0_2/usage/getting-started.html>). 此处不在赘述。

