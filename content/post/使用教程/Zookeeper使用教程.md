+++
date = '2026-10-21T10:00:00+08:00'
draft = true
title = 'Zookeeper使用教程'
tags = ["Zookeeper","使用教程"]
series = ["使用教程"]
categories = ["分布式", "Zookeeper"]
+++

# Zookeeper使用教程

本文内容整理自尚硅谷平台课程，仅供学习使用。

## 1 Zookeeper入门

### 1.1 概述

Zookeeper 是一个开源的分布式框架，为分布式应用提供协调服务，属于 Apache 旗下项目。

**Zookeeper 工作机制**

从设计模式角度理解，Zookeeper 是一个基于**观察者模式**设计的分布式服务管理框架：它负责存储和管理大家都关心的数据，接受观察者的注册；一旦这些数据的状态发生变化，Zookeeper 就负责通知已经在它上面注册的那些观察者做出相应反应。

可以用一句话概括其定位：

> Zookeeper = 文件系统 + 通知机制

前者体现在它的数据模型（ZNode 树），后者体现在它的 Watcher 机制。

### 1.2 Zookeeper特点

1. Zookeeper 由一个领导者（Leader）、多个跟随者（Follower）组成集群。
2. 集群中只要有**半数以上**节点存活，Zookeeper 集群就能正常服务，因此 Zookeeper 适合部署在奇数台服务器上。
3. **全局数据一致**：每个 Server 保存一份相同的数据副本，Client 无论连接到哪个 Server，读到的数据都是一致的。
4. **更新请求顺序执行**：来自同一个 Client 的更新请求按其发送顺序依次执行。
5. **数据更新原子性**：一次数据更新要么成功，要么失败。
6. **实时性**：在一定时间范围内，Client 能读到最新数据。

### 1.3 数据结构

Zookeeper 数据模型的结构与 Unix 文件系统很类似，整体上可以看做是一棵树，每个节点称作一个 **ZNode**。每一个 ZNode 默认能够存储 1MB 的数据，每个 ZNode 都可以通过其路径唯一标识。

ZNode 的两种类型：

| 类型 | 说明 |
|------|------|
| **持久化（Persistent）** | 客户端断开连接后节点仍然保留，是默认类型 |
| **临时（Ephemeral）** | 客户端会话失效后节点自动删除，常用于服务注册与心跳检测 |

两者都可以附加**顺序（Sequential）**属性，创建时路径末尾会自动追加一个单调递增的序号，常用于实现公平锁与全局唯一 ID。

### 1.4 应用场景

提供的服务包括：统一命名服务、统一配置管理、统一集群管理、服务器节点动态上下线、软负载均衡等。

- **统一命名服务**：在分布式环境下，经常需要对应用/服务进行统一命名，便于识别。
- **统一配置管理**：分布式环境下，配置文件同步非常常见。可将配置信息写入 Zookeeper 上的一个 ZNode，各个客户端服务器监听这个 ZNode，配置变更时自动收到通知。
- **统一集群管理**：实时掌握每个节点的状态是必要的。可以将节点信息写入 Zookeeper 上的一个 ZNode，监听这个 ZNode 可获取它的实时状态变化。
- **服务器动态上下线**：客户端能实时洞察到服务器上下线的变化。
- **负载均衡**：在 Zookeeper 中记录每台服务器的访问数，让访问数最少的服务器去处理最新的客户端请求。

### 1.5 下载地址

官网首页：[Apache ZooKeeper](https://zookeeper.apache.org/)

## 2 Zookeeper本地安装

### 2.1 本地模式安装

1. 安装前准备：安装 JDK。
2. 拷贝 `apache-zookeeper-3.5.7-bin.tar.gz` 安装包到 Linux 系统下。
3. 解压到指定目录。
4. 修改名称，便于后续脚本引用。

```bash
# 解压
tar -zxvf apache-zookeeper-3.5.7-bin.tar.gz -C /opt/module/
# 改名
mv /opt/module/apache-zookeeper-3.5.7-bin /opt/module/zookeeper-3.5.7
```

**配置修改**

1. 将 `/opt/module/zookeeper-3.5.7/conf` 这个路径下的 `zoo_sample.cfg` 修改为 `zoo.cfg`。
2. 打开 `zoo.cfg` 文件，修改 `dataDir` 路径。
3. 在 `/opt/module/zookeeper-3.5.7/` 这个目录上创建 `zkData` 文件夹。

```bash
cd /opt/module/zookeeper-3.5.7/conf
mv zoo_sample.cfg zoo.cfg

vi zoo.cfg
# 修改：dataDir=/opt/module/zookeeper-3.5.7/zkData

mkdir /opt/module/zookeeper-3.5.7/zkData
```

**操作 Zookeeper**

```bash
# 1. 启动 Zookeeper
bin/zkServer.sh start

# 2. 查看进程是否启动
jps    # 出现 QuorumPeerMain 即表示已启动

# 3. 查看状态
bin/zkServer.sh status   # 单机模式下输出 Mode: standalone

# 4. 启动客户端
bin/zkCli.sh

# 5. 退出客户端
quit

# 6. 停止 Zookeeper
bin/zkServer.sh stop
```

### 2.2 配置参数解读

Zookeeper 中的配置文件 `zoo.cfg` 中参数含义解读如下：

| 参数 | 含义 |
|------|------|
| `tickTime=2000` | 通信心跳时间，Zookeeper 服务器与客户端心跳时间，单位毫秒 |
| `initLimit=10` | LF 初始通信时限，Leader 和 Follower 初始连接时能容忍的最多心跳数（`tickTime` 的数量） |
| `syncLimit=5` | LF 同步通信时限，Leader 和 Follower 之间通信时间如果超过 `syncLimit × tickTime`，Leader 认为 Follower 已死掉，从服务器列表中删除该 Follower |
| `dataDir` | 保存 Zookeeper 中的数据。注意：默认的 tmp 目录容易被 Linux 系统定期删除，所以一般不用默认的 tmp 目录 |
| `clientPort=2181` | 客户端连接端口，通常不做修改 |

## 3 Zookeeper基础操作

### 3.1 集群操作

#### 3.1.1 集群安装

1. **集群规划**：在 hadoop102、hadoop103 和 hadoop104 三个节点上都部署 Zookeeper。
2. **解压安装**：把 Zookeeper 安装包拷贝到每台机器并解压。
3. **配置服务器编号**：在 `zkData` 目录下创建 `myid` 文件，添加与该 Server 对应的编号。
4. **配置 `zoo.cfg` 文件**。
5. **集群操作**：分别启动 Zookeeper，查看状态。

```bash
# 在每台机器的 zkData 目录下写入各自的编号（内容必须与集群规划一致）
echo 2 > /opt/module/zookeeper-3.5.7/zkData/myid    # hadoop102
echo 3 > /opt/module/zookeeper-3.5.7/zkData/myid    # hadoop103
echo 4 > /opt/module/zookeeper-3.5.7/zkData/myid    # hadoop104

# zoo.cfg 中追加集群配置
server.2=hadoop102:2888:3888
server.3=hadoop103:2888:3888
server.4=hadoop104:2888:3888
```

端口说明：`2888` 是 Follower 与 Leader 之间**信息交换**的端口；`3888` 是集群内机器**选举 Leader** 使用的端口。

```bash
# 分别在三台机器上启动并查看状态
bin/zkServer.sh start
bin/zkServer.sh status   # 其中一台为 leader，其余为 follower
```

#### 3.1.2 选举机制（面试重点）

**SID**：服务器 ID，用来唯一标识一台 Zookeeper 集群中的机器，每台机器不能重复，和 `myid` 一致。

**ZXID**：事务 ID，用来标识一次服务器状态的变更。

**Epoch**：每个 Leader 任期的代号。

**第一次启动选举流程：**

1. 服务器 1 启动，发起一次选举，投给自己一票。
2. 服务器 2 启动，服务器 1 发现服务器 2 的 `myid` 比自己大，更改选票为推举服务器 2。
3. 服务器 3 启动，服务器 1 和 2 都会更改选票为服务器 3，此时服务器 3 获得半数以上选票，当选 Leader。
4. 服务器 4、5 启动，由于多数票已经产生，它们只能服从多数，更改选票信息为服务器 3。

**非第一次启动选举规则**（按优先级依次比较）：

1. EPOCH 大的直接胜出；
2. EPOCH 相同，事务 ID（ZXID）大的胜出；
3. 事务 ID 相同，服务器 ID（SID）大的胜出。

### 3.2 客户端命令行操作

| 命令 | 功能描述 |
|------|----------|
| `help` | 显示所有操作命令 |
| `ls path` | 查看当前 znode 的子节点（可监听） |
| `create` | 普通创建（`-s` 含有序列，`-e` 临时） |
| `get path` | 获得节点的值（可监听） |
| `set` | 设置节点的具体值 |
| `stat` | 查看节点状态 |
| `delete` | 删除节点 |
| `deleteall` | 递归删除节点 |

### 3.3 客户端API操作

包括 IDEA 环境搭建、创建 Zookeeper 客户端、创建子节点、获取子节点并监听节点变化、判断 ZNode 是否存在等。

核心依赖与用法：

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.5.7</version>
</dependency>
```

```java
// 获取客户端连接
ZooKeeper zooKeeper = new ZooKeeper("hadoop102:2181", 5000, new Watcher() {
    @Override
    public void process(WatchedEvent watchedEvent) {
        System.out.println(watchedEvent.getPath() + " 发生变化：" + watchedEvent.getType());
    }
});

// 创建节点
zooKeeper.create("/zk-test", "0".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);

// 获取子节点并监听变化（第二个参数 true 表示注册 Watcher）
List<String> children = zooKeeper.getChildren("/", true);

// 判断 ZNode 是否存在
Stat stat = zooKeeper.exists("/zk-test", false);
```

### 3.4 客户端向服务端写数据流程

Zookeeper 的写操作必须由 Leader 统一发起，整体流程如下：

1. Client 将写请求发送给**任意一台** Server。
2. 如果接收请求的 Server 不是 Leader，它把请求**转发给 Leader**。
3. Leader 收到请求后，把该事务（Proposal）分发给集群中所有的 Follower，同时自己也写入事务日志。
4. 每个 Server 把事务日志**持久化到磁盘**，写成功后向 Leader 返回 ACK。
5. Leader 收到**集群中半数以上**节点的 ACK 后，向所有 Follower 发送 Commit 消息，正式提交该事务。
6. Follower 将事务应用到内存数据库，更新完成后，由与 Client 相连的那台 Server 向 Client 反馈写操作成功。

要点：

- 写请求是**串行**的，Leader 通过 ZXID 单调递增来保证全局顺序，这就是 ZAB 协议的核心。
- 只要半数以上节点 ACK，写入即算成功，因此 3 节点集群可容忍 1 个节点故障，5 节点可容忍 2 个。
- 读请求则不同：任意一台 Server 都可以直接从自己的内存副本返回，不经过 Leader，所以读性能随节点数增加而提升。

## 4 服务器动态上下线监听案例

**需求**：某分布式系统中，Server 可能动态上线或下线，Client 需要实时感知到 Server 列表的变化。

**设计思路**：

1. Zookeeper 中创建一个永久节点 `/servers` 作为服务器列表的父节点。
2. 每台 Server 启动时，在 `/servers` 下创建一个**临时节点**（如 `/servers/192.168.10.100:8080`）；Server 宕机或会话失效时，该临时节点被自动删除。
3. Client 启动时调用 `getChildren("/servers", true)` 获取当前可用服务器列表，并注册 Watcher。
4. 任一 Server 上下线都会导致 `/servers` 的子节点变化，Zookeeper 向 Client 推送 `NodeChildrenChanged` 事件；Client 在 Watcher 回调中**重新获取子节点列表**并更新本地缓存。

**关键点**：

- 必须使用**临时节点**，这是"下线即感知"的基础——不需要 Server 主动上报死亡。
- Zookeeper 的 Watcher 是**一次性的**，触发后需要重新注册，否则只能收到第一次变更通知。
- Client 本地缓存一份服务器列表，避免每次业务调用都访问 Zookeeper。

## 5 Zookeeper分布式锁案例

### 5.1 原生Zookeeper实现分布式锁案例

**基本实现**：多个客户端争抢创建同一个节点，`create` 在 Zookeeper 中是原子的，创建成功即获得锁；业务完成后 `delete` 该节点即释放锁。

```java
try {
    zooKeeper.create("/lock", "1".getBytes(),
            ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
    // 创建成功，获得锁，执行业务
} catch (NodeExistsException e) {
    // 节点已存在，抢锁失败，此处可注册 Watcher 等待锁释放后重试
} finally {
    // 释放锁
    zooKeeper.delete("/lock", -1);
}
```

**存在的问题**：

- **羊群效应**：抢锁失败的客户端都在监听同一个节点，锁释放时全部被唤醒，但只有一个能抢到，其余再次失败，造成大量无效通知与上下文切换。
- **不可重入**：同一线程再次获取锁时会被自己阻塞。
- **公平性无法保证**：唤醒顺序不代表申请顺序，后到的客户端可能先抢到锁。

**改良方案**：改用**临时顺序节点**。每个客户端在 `/lock` 下创建 `EPHEMERAL_SEQUENTIAL` 序号节点，序号最小者获得锁；其余客户端只监听**自己前一个**节点。这样既消除了羊群效应，又天然保证了 FIFO 公平性。

### 5.2 Curator框架实现分布式锁案例

Curator 是 Apache 官方的 Zookeeper 客户端框架，封装了重试策略与分布式锁等常见配方（recipes），生产环境一般直接使用它而不是裸写 Zookeeper API。

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.1.0</version>
</dependency>
```

Curator 提供的锁类型：

| 类 | 说明 |
|------|------|
| `InterProcessMutex` | 分布式**可重入**互斥锁 |
| `InterProcessSemaphoreMutex` | 分布式**不可重入**互斥锁 |
| `InterProcessReadWriteLock` | 分布式读写锁，读读不互斥、读写互斥、写写互斥 |
| `InterProcessSemaphoreV2` | 信号量，用于控制并发数量 |

```java
// 创建 Curator 客户端
CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString("hadoop102:2181,hadoop103:2181,hadoop104:2181")
        .sessionTimeoutMs(30000)
        .retryPolicy(new ExponentialBackoffRetry(1000, 3))
        .build();
client.start();

// 可重入分布式锁
InterProcessMutex lock = new InterProcessMutex(client, "/locks/order");
try {
    // 尝试加锁，最多等待 10 秒
    if (lock.acquire(10, TimeUnit.SECONDS)) {
        // 执行业务
    }
} finally {
    lock.release();   // 释放锁
}
```

## 6 企业面试真题（面试重点）

### 6.1 选举机制

参见 [3.1.2 选举机制](#312-选举机制面试重点)。回答时抓住三点：第一次启动靠 **SID** 比较、非第一次启动按 **EPOCH > ZXID > SID** 的优先级依次比较、以及**半数以上**通过才能当选。

### 6.2 生产集群安装多少zk合适？

**结论**：至少 3 台，且部署**奇数台**，常见规模为 3、5、7 台。

**理由**：

1. Zookeeper 需要半数以上节点存活才能对外服务，可用性要求下 1 台没有容错能力，2 台挂 1 台即不可用，因此最小可用集群是 3 台。
2. 集群能容忍的故障节点数为 `(n − 1) / 2`（向下取整）。4 台集群只能容忍 1 台故障，与 3 台相同，多出来的 1 台纯属浪费；而 5 台能容忍 2 台。既然偶数不带来额外容错能力，就应选奇数。
3. 奇数节点在**网络分区**场景下更容易避免出现"两个分区都凑够半数"的脑裂情况。
4. 节点越多，写请求需要等待的 ACK 越多，写延迟随之上升，因此不建议盲目扩容到很大规模；读吞吐不足时更适合加 Observer 而非 Follower。

### 6.3 常用命令

![](/images/Zookeeper使用教程1.png)

![](/images/Zookeeper使用教程2.png)
