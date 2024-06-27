---
layout: post
title: RuoYi-Cloud架构浅析
subtitle: 以一个服务请求为例观察RuoYi-Cloud各组件协作
date: 2021-07-19
author: HuK
header-img: img/bg-material.jpg
catalog: true
tags:
  - RuoYi-Cloud
  - 架构图
---

### 框架设计

RuoYi-Cloud 是一个 Java EE 分布式微服务架构平台，基于经典技术组合（Spring Boot、Spring Cloud & Alibaba、Vue、Element），内置模块如：部门管理、角色用户、菜单及按钮授权、数据权限、系统参数、日志管理、代码生成等。在线定时任务配置；支持集群，支持多数据源。

RuoYi-Cloud 的框架设计图如下所示：

![framework](/img/1.png)

其中一些核心模块如下所示：

- `SpringBootAdmin Server` 监控和管理 Spring Boot 应用程序
- `Nacos` 服务发现和配置管理中心。
- `ruoyi-gateway` API 网关，负责路由请求和负载均衡。
- `Service-A 和 Service-B` 微服务组件，实现具体的业务逻辑。
- `Sidecar` 用于与异构服务的集成和代理。
- `quartz 2.3.2` 任务调度：管理和执行定时任务。
- `Minio/FastDFS` 用于提供对象存储服务。
- `ElasticSearch` 全文搜索和分析引擎，
- `Redis` 缓存数据库。
- `MySQL 5.7.x` 关系型数据库。
- `NIFI` 数据流管理和集成工具。
- `ruoyi-auth、ELK` 认证服务以及日志收集分析组件
- `Zipkin/SkyWalkingr` 分布式追踪系统。
- `Spring Cloud Stream` 消息驱动微服务框架。

### 模块联系

以一个具体的用户信息查询请求为例，详细描述每一步的处理过程和各个组件之间的互动。

1、 用户发起请求

用户在浏览器中访问 API 以查询用户信息，URL 为 http://<gateway-host>/api/user/{id}。

2、 请求到达网关（ruoyi-gateway）

- ruoyi-gateway 接收到用户请求后，首先进行路由、身份验证和负载均衡。
- Sentinel/Ribbon 在网关中启用，确保请求的限流和熔断机制。
- ruoyi-gateway 通过与 Nacos 通信，查询到具体处理该请求的服务实例列表（Service-A 或 Service-B）

```
1. 用户浏览器 -> ruoyi-gateway: 发起 HTTP 请求
2. ruoyi-gateway -> Nacos: 查询 Service-A 实例
3. Nacos -> ruoyi-gateway: 返回 Service-A 实例列表
4. ruoyi-gateway -> Sentinel/Ribbon: 执行负载均衡
5. ruoyi-gateway -> Service-A: 转发用户请求
```

3、请求到达 Service-A

- Service-A 收到请求后，首先通过 ruoyi-auth 进行认证和授权验证，以确保用户有权限访问该 API。
- ruoyi-auth 验证通过后，Service-A 开始处理具体的业务逻辑。

```
6. Service-A -> ruoyi-auth: 进行身份验证
7. ruoyi-auth -> Service-A: 验证通过

```

4、处理业务逻辑

- Service-A 从 Redis 缓存中查询用户信息。如果缓存命中，则直接返回结果；如果缓存未命中，则从 MySQL 数据库中查询。
- 如果从数据库中查询到数据，Service-A 会将数据写入 Redis 缓存，以便下次请求快速响应。

```
8. Service-A -> Redis: 查询用户信息
9. Redis -> Service-A: 如果缓存命中，返回用户信息
10. Redis: 如果缓存未命中，Service-A查询MySQL
11. Service-A -> MySQL: 查询用户信息
12. MySQL -> Service-A: 返回用户信息
13. Service-A -> Redis: 将用户信息写入缓存
```

5、分布式追踪

- 在请求处理的每一步，Sleuth 都会记录链路数据，并将这些数据发送到 Zipkin 或 SkyWalking，以便对请求链路进行追踪和分析。
- 这样可以监控和分析整个请求的执行时间、路径和可能的瓶颈。

```
14. Sleuth: 记录请求链路数据
15. Sleuth -> Zipkin/SkyWalking: 发送链路数据
```

6、日志记录和监控

- Service-A 在处理请求时，会记录日志，包括请求的接收、处理和响应情况。这些日志数据通过 ELK 系统进行集中化管理和分析。
- SpringBootAdmin Server 实时监控 Service-A 的运行状态，包括健康状况、性能指标等，确保服务稳定运行。

```
16. Service-A -> ELK: 记录日志数据
17. ELK: 分析和存储日志数据
18. Service-A -> SpringBootAdmin Server: 提供健康状态数据
19. SpringBootAdmin Server: 监控服务健康状况
```

7、任务调度

- 如果请求涉及到定时任务或周期性操作（例如，定时刷新缓存），这些任务会由 quartz 调度执行。
- quartz 与 Nacos 集成，确保任务调度的动态配置和管理。

```
20. quartz -> Nacos: 注册和管理定时任务
```

8、返回响应

- Service-A 处理完成后，将结果返回给 ruoyi-gateway。
- ruoyi-gateway 再将响应结果返回给前端用户，完成整个请求-响应过程。

```
21. Service-A -> ruoyi-gateway: 返回处理结果
22. ruoyi-gateway -> 用户浏览器: 返回响应结果
```

### 总结

通过以上详细分析，可以看到 RuoYi-Cloud 框架中的各个组件是如何有效协作，完成一个前端用户请求的处理过程。每个组件都在其中扮演了特定的角色，从请求的接收、路由、负载均衡、身份验证、业务逻辑处理、数据存储与检索、分布式追踪、日志记录、任务调度，到最后的响应返回，形成了一个完整的微服务架构体系。这种协作方式不仅提高了系统的灵活性和可扩展性，也保证了高可用性和可靠性。
