---
title: spring cloud
description: 微服务概念学习
date: 2018-08-19 13:12:12
keywords: java
categories : [spring,spring cloud]
tags : [java, spring cloud,网关,docker]
comments: true
---

# 基础知识

<img src="/images/spring-cloud-architecture.png">

## 四大神器
- auto-configuration
- starters
	<img src="/images/spring-cloud-stater.png">
- cli：Spring Boot Commad Line	
- Autuator监控

# 服务注册与发现

## Eureka

<img src="/images/aws-eureka.png">

[参考链接](https://luyiisme.github.io/2017/04/22/spring-cloud-service-discovery-products/?utm_source=tuicool&utm_medium=referral)

| Feature | Consul | zookeeper | etcd | Eureka |
| :-: | :-: | :-: | :-: | :-: |
| 服务健康检查 | 服务状态，内存，硬盘等 | (弱)长连接，keepalive | 连接心跳 | 可配支持 |
| 多数据中心 | 支持 | — | — | — |
| kv存储服务 | 支持 | 支持 | 支持	 | — |
| 一致性 | raft | paxos | raft | — |
| cap | ca | cp | cp | ap |
| 使用接口(多语言能力) | 支持http和dns | 客户端 | http/grpc | http（sidecar） |
| watch支持 | 全量/支持long polling	支持 | 支持 | long polling | 支持 long polling/大部分增量 |
| 自身监控 | metrics | — | metrics | metrics |
| 安全 | acl /https | acl	https支持（弱） | — |
| spring cloud集成 | 已支持 | 已支持 | 已支持 | 已支持 |


## CAP理论与BASE思想
## 服务发现的方式

- 客户端发现
<img src="/images/client-find.png">
- 服务器端发现
<img src="/images/server-find.png">

# 负载均衡

## LB方案

[参考链接1](https://www.cnblogs.com/mindwind/p/5339657.html)

[参考链接2](https://www.cnblogs.com/LittleHann/p/3963255.html)

<img src="/images/lb-whole.jpg">

- 硬负载
- 软负载
- DNS负载
- CDN负载

## Ribbon
**核心组件**

- ServerList	
	用于获取地址列表。它既可以是静态的(提供一组固定的地址)，也可以是动态的(从注册中心中定期查询地址列表)	
- ServerListFilter	
	仅当使用动态ServerList时使用，用于在原始的服务列表中使用一定策略过虑掉一部分地址	
- IRule	
	选择一个最终的服务地址作为LB结果。选择策略有轮询、根据响应时间加权、断路器(当Hystrix可用时)等	

# rest调用	

## Feign

# 容错处理

[参考链接1](http://blog.51cto.com/developerycj/1950881)
[参考链接2](https://www.cnblogs.com/leeSmall/p/8847652.html)

## Hystrix

- 限流
- 降级
- 熔断

## Dashboard 服务监控
## Turbine 聚合监控

# 微服务网关

## Zuul

### 过滤器机制
<img src="/images/zuul-filter.png">

### 标准过滤器类型

- PRE
	- 鉴权
	- 流量转发
- ROUTING
- POST
	- 跨域
	- 统计
- ERROR

### request生命周期
<img src="/images/zuul-filter-lifecyle.png">

[参考链接1](http://www.scienjus.com/api-gateway-and-netflix-zuul/#Netflix-Zuul)

[参考链接2](https://www.cnblogs.com/lexiaofei/p/7080257.html)

### 稳定性

- 隔离机制
- 重试机制

### 主要功能

- 验证与安全保障
- 审查与监控
- 动态路由
- 压力测试
- 负载分配
- 静态响应处理
- 多区域弹性

# 微服务配置

## Spring Cloud Config

<img src="/images/spring-cloud-config.png">

## Spring Cloud Bus

<img src="/images/configbus2.jpg">

[参考链接](https://www.cnblogs.com/ityouknow/p/6931958.html)

- 提交代码触发post请求给bus/refresh
- server端接收到请求并发送给Spring Cloud Bus
- Spring Cloud bus接到消息并通知给其它客户端
- 其它客户端接收到通知，请求Server端获取最新配置
- 全部客户端均获取到最新的配置

# 微服务跟踪

## Spring Cloud Sleuth

## ZipKin

# Docker

## 入门

## Docker Compose

