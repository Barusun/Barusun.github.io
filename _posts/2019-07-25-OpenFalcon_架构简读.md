---
layout:     post
title:      OpenFalcon 架构简读
subtitle:   架构简读
date:       2019-07-25
author:     Baru
header-img: img/404-bg.jpg
catalog: true
tags:
    - Linux
    - 监控Open-Falcon
    - 运维
---


# OpenFalcon 架构简读

[TOC]

## 架构图

![func_intro_1.png](https://i.loli.net/2019/07/25/5d3962e42b64988199.png)

## Agent

​	`agent`用于采集机器负载监控指标，比如`cpu.idle`、`load.1min`、`disk.io.util`等等，每隔60秒push给`Transfer`。`agent`与`Transfer`建立了长连接，数据发送速度比较快，`agent`提供了一个http接口`/v1/push`用于接收用户手工push的一些数据，然后通过长连接迅速转发给`Transfer`。

​	所以`agent`需要指向正确的`transfer`地址。

## Transfer

​	`transfer`是数据转发服务。它接收`agent`上报的数据，然后按照哈希规则进行数据分片、并将分片后的数据分别push给`graph`&`judge`等组件。

​	所以`transfer`的`cfg.json`配置文件里需要执行正确的`graph`和`judge`地址。

## Graph

​	`graph`是存储绘图数据的组件。`graph`组件 接收`transfer`组件推送上来的监控数据，同时处理`api`组件的查询请求、返回绘图数据。

​	`transfer`和`api`需要配置正确的`graph`地址。

## API

​	`api`组件，提供统一的`restAPI`操作接口。比如：`api`组件接收查询请求，根据一致性哈希算法去相应的`graph`实例查询不同metric的数据，然后汇总拿到的数据，最后统一返回给用户。

​	`dashboard`组件的配置要修改`api`的IP配置，使其能够正确寻址到`api`组件。

​	确保`api`组件的`graph`列表 与 `transfer`的配置 一致

## Dashborad

​	是OpenFalcon自带的web界面，可通过`api`去查询或者绘图相关数据。

## Heartbeat Server（HBS）

​	所有`agent`都会连到HBS，每分钟发一次心跳请求; `agent`需要配置正确的HBS地址；

​	除此之外，`HBS`去读取`Portal`的数据库，将需要采集的端口和进程信息返回给`agent`,然后agent来采集；

​	`Judge`需要获取所有的报警策略，让`Judge`去读取`Portal`的DB么？不太好。因为`Judge`的实例数目比较多，如果公司有几十万机器，`Judge`实例数目可能会是几百个，几百个`Judge`实例去访问`Portal`数据库，也是一个比较大的压力。既然HBS无论如何都要访问`Portal`的数据库了，那就让HBS去获取所有的报警策略缓存在内存里，然后`Judge`去向HBS请求。这样一来，对`Portal` DB的压力就会大大减小。

## Judge

​	`Judge`用于告警判断，`agent`将数据`push`给`Transfer`，`Transfer`不但会转发给Graph组件来绘图，还会转发给`Judge`用于判断是否触发告警。

## Alarm

​	`alarm`模块是处理报警event的，`judge`产生的报警event写入redis，`alarm`从redis读取处理，并进行不同渠道的发送。

## 参考

 	1.  [OpenFalcon官方资料](<https://book.open-falcon.org/zh_0_2/distributed_install/>)；
 	2.  [OpenFalcon架构解析](<http://www.jikexueyuan.com/course/1651.html>)。

