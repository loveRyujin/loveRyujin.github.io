+++
date = '2026-01-22T10:00:00+08:00'
draft = false
title = 'Docker部署单机或集群kafka'
tags = ["杂谈"]
+++

## 基本概念
在部署 kafka 之前，我们需要了解 kafka 集群中节点的角色（单机可视为单节点集群）、常见的配置项等等，方便后续部署。

- broker：集群中的工作节点，也就是我们通常说的消息队列，负责**存储消息和处理消息**，各个 broker 都是一个单独的服务器，运行在不同的机器上。
- controller：集群中的控制节点，负责**管理集群元数据和状态**。
- 混合节点：同时充当 broker 和 controller。

kafka主要存在两种模式，**ZooKeeper**和**KRaft**。

### ZooKeeper模式
该模式下，kafka 集群由两部分组成：Kafka Brokers 和 ZooKeeper Ensemble。

- 外部依赖：kafka 自身不存储元数据（如 Topic 信息，分区分配，Controller 选举等），而是讲相关信息存放在 ZooKeeper 中。
- Controller角色：kafka 集群中会有一个 broker 被选为 controller，负责和 ZooKeeper 通信，监听它的数据变化，并将相应指令同步给其它 broker。

### KRaft模式
KRaft 模式下彻底移除了集群对 ZooKeeper 的依赖，在集群内部实现了共识协议。
- 自管理：kafka 集群使用类 Raft 算法管理内部元数据。
- 角色划分：部分 broker 被指定为 controller，负责管理元数据。
- 元数据快照：元数据被存放在一个特殊的 topic 下。

### ZooKeeper 与 KRaft 对比

| 特性 | ZooKeeper 模式 | KRaft 模式 |
| --- | --- | --- |
| 依赖性 | 必须安装并运行 ZooKeeper 集群 | 原生内置，无需外部组件 |
| 元数据存储 | 存储在外部 ZooKeeper 节点 | 存储在内部隔离的元数据 Topic 中 |
| Controller 选举 | 依赖 ZooKeeper 抢占临时节点 | 基于 Raft 协议的多数派投票 |
| 可扩展性 | 分区数量受限（通常约 20 万个） | 支持百万级分区 |
| 故障恢复 | Controller 切换后需从 ZK 加载所有元数据（慢） | Controller 备机实时同步内存状态（极快） |
| 运维复杂度 | 需维护两套不同的分布式系统 | 仅需维护 Kafka 一套系统 |


## 单机部署
首先需要拉取 kafka 镜像
```bash
docker pull apache/kafka:4.0.1
```

我们选用更现代的 KRaft 模式部署，由于只有一个节点，该节点为混合节点。

```bash
docker run -d --name kafka-kraft \
    -p 9092:9092 \
    -e KAFKA_CFG_NODE_ID=1 \
    -e KAFKA_CFG_PROCESS_ROLES=controller,broker \
    -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
    -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
    -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
    -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
    -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
    -e KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT \
    -e KAFKA_KRAFT_CLUSTER_ID=my-custom-id \
    apache/kafka:4.0.1
```

解释一下上述注入的相关环境：

1、节点角色定义
- KAFKA_CFG_NODE_ID：节点 ID，ID 必须在集群中唯一且为非负整数。
- KAFKA_CFG_PROCESS_ROLES：节点角色，比如 broker、controller。
- KAFKA_CFG_CONTROLLER_QUORUM_VOTERS：具有投票权的控制器节点，格式为 id@host:port。

2、监听器配置
- KAFKA_CFG_LISTENERS：定义 broker 绑定在哪些网络接口和端口上，比如格式 PLAINTEXT://:9092,CONTROLLER://:9093。表示9092端口负责数据，9093端口负责控制命令。
- KAFKA_CFG_ADVERTISED_LISTENERS：客户端访问 broker 的连接地址。
- KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP：定义自定义监听器的安全协议。CONTROLLER:PLAINTEXT告诉 kafka 这个监听器走的明文传输协议。
- KAFKA_CFG_CONTROLLER_LISTENER_NAMES：明确告诉告诉 kafka 哪些监听器是供 controller 使用的
- KAFKA_CFG_INTER_BROKER_LISTENER_NAME：指定 Broker 之间相互同步数据时使用哪个监听器。通常使用 PLAINTEXT 或内部专用的网络平面。

3、集群初始化
- KAFKA_KRAFT_CLUSTER_ID：用于格式化存储目录的唯一集群 ID。KRaft 启动前必须有一个 meta.properties 文件，记录了 Cluster ID。如果 ID 不匹配，Broker 将拒绝加入集群。这能有效防止节点误加入错误的集群。

