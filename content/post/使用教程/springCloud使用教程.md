+++
date = '2026-10-21T10:00:00+08:00'
draft = true
title = 'springCloud使用教程'
tags = ["springCloud","使用教程"]
series = ["使用教程"]
categories = ["微服务", "springCloud"]
+++

# springCloud使用教程

## 1 选型2024年教程学习

| 功能模块 | 推荐方案 | 备选方案 |
|---------|---------|---------|
| 分布式配置管理 | Nacos | Consul, Config Bus |
| 服务注册与发现 | Consul (主角) | Zookeeper, Eureka (降低), Nacos, Etcd |
| 服务网关 | Gate Way | Zuul |
| 服务链路追踪 | Micrometer Tracing (主角) | Sleuth + Zipkin |
| 服务调用和负载均衡 | OpenFeign (主角) | Ribbon (取消), LoadBalancer |
| 服务熔断和降级 | Circuit Breaker (主角) | Hystrix, Resilience4J, Sentinel |
| 分布式事务 | Seata (主角) | LCN, Hmily |

## 2 注意事项

健康检查失败的consul服务，是无法识别的
