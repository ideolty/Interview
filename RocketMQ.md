# 综述

1. 版本使用release-4.7.0
2. [官网地址](http://rocketmq.apache.org/)
3. [GitHub地址](https://github.com/apache/rocketmq/)



## 结构图

![Architecture](截图/RocketMQ/Architecture.png)

先看一下官方给出的结构图，图中分为了4个大的模块，producer、namesrv、broker、consumer，这4个模块都是可以集群部署。



# NameServer

引入：

1. 为什么要单独写一个注册中心，为什么不采用网上现有的解决方案，例如zk，eureka
2. nameServer有什么优点？架构是怎么样的？与zk，eureka差异。
3. 启动流程
4. client的注册流程，nameServer故障发现，故障排除
5. 集群的实现方式，各节点间关系，怎样同步各个节点间数据
6. nameServer中储存了一些什么信息，储存结构是怎么样的，生命周期如何维护
7. nameServer提供了broker discovery功能，存储了routing info信息，相关功能是怎么用的





# Producer

引入

- 启动流程
- 消息发送流程
- 消息重发
- 事务消息



#  Broker

引入

- broker物理、逻辑结构，broker与queue的关系
- 启动流程
- 消息的存储结构与发送消息的接受流程
- 请求路由、负载均衡、动态扩容
- 消息的持久化机制
- 集群结构，节点间关系、节点间高可用



# Consumer

引入

- 启动流程
- 消息拉取过程
  - 主动拉取
  - 被动拉取
- 消费的具体流程
- 消费队列的负载均衡与rebalance
- 定时消费的具体实现
- 如果需要有严格消费顺序，该怎么做？