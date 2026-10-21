+++
date = '2025-12-12T10:00:00+08:00'
draft = true
title = 'Zookeeper源码分析'
tags = ["Zookeeper","源码分析"]
series = ["使用教程"]
categories = ["分布式", "Zookeeper"]
+++

# Zookeeper源码分析

本文尚硅谷平台，仅学习使用。

## 1 算法基础

### 1.1 拜占庭将军问题

### 1.2 Paxos算法

### 1.3 ZAB协议

#### 1.3.1 什么是ZAB算法

#### 1.3.2 ZAB协议内容

## 2 源码详解

### 2.1 辅助源码

#### 2.1.1 持久化源码

#### 2.1.2 序列化源码

### 2.2 ZK服务端初始化源码解析

#### 2.2.1 ZK服务端启动脚本分析

#### 2.2.2 ZK服务端启动入口

#### 2.2.3 解析参数zoo.cfg和myid

#### 2.2.4 过期快照删除

#### 2.2.5 初始化通信组件

### 2.3 ZK服务端加载数据源码解析

#### 2.3.1 冷启动数据恢复快照数据

#### 2.3.2 冷启动数据恢复编辑日志

### 2.4 ZK选举源码解析

#### 2.4.1 选举准备

#### 2.4.2 选举执行

### 2.5 Follower和Leader状态同步源码

#### 2.5.1 Leader.lead()等待接受follower的状态同步申请

#### 2.5.2 Follower.lead()查找并连接Leader

#### 2.5.3 Leader.lead()创建LearnerHandler

#### 2.5.4 Follower.leader()创建registerWithLeader

#### 2.5.5 Leader.lead()接收Follower状态

#### 2.5.6 Follower.lead()应答Leader同步结果

#### 2.5.7 Leader.lead()应答Follower

### 2.6 服务端Leader启动

### 2.7 服务端Follower启动

### 2.8 客户端启动

#### 2.8.1 创建ZookeeperMain

#### 2.8.2 初始化监听器

#### 2.8.3 解析连接地址

#### 2.8.4 创建通信

#### 2.8.5 执行run()
