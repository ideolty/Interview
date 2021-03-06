

# 综述

目前为应付面试，记录一下介绍性质的文字、一些面试题，以及一些比较好的资料，待之后再整理。

视频资料来自于极客时间《flink核心技术与实战》



> [Flink面试题](https://zhuanlan.zhihu.com/p/138101642)
>
> [Flink面试题大全(建议收藏)](https://blog.csdn.net/weixin_44439549/article/details/109012515)
>
> 



# 核心特性

- 统一数据处理组件栈，streaming dataflow runtime
- 支持事件时间(event time)、接入时间(ingestion time)、处理时间(processing time)
- 基于轻量级分布式快照(snapshot)实现容错，checkpoint，exact once
- 支持有状态计算
- 窗口
- 反压
- 基于jvm实现的独立内存管理



# 集群架构

整体架构图，分为三个部分。

![image-20210508230737429](截图/Flink/整体架构图.png)



## JobManager

<img src="截图/Flink/JobManager架构图.png" alt="image-20210508231027183"  />

负责整个集群资源的管理以及相应的调度工作，他会负责

1. taskManager之间checkpoint的协调执行。
2. 对于客户端提交的jobGraph（逻辑执行图）生成对应的执行图（物理执行逻辑图）。
3. 等等，对应上图几点





## TaskManager

![image-20210508232326317](截图/Flink/TaskManager架构图.png)

TaskManager节点之间的数据交互有提供network manager模块，他基于netty来实现网络交互。节点与节点之间进行RPC通信是使用Actor



## Client

![image-20210508232718606](截图/Flink/Client架构图.png)

### JobGraph

![image-20210508232949174](截图/Flink/JobGraph结构图.png)



# 集群运行模式

![image-20210509084040478](截图/Flink/集群运行模式.png)

## Session

![image-20210509084130391](截图/Flink/session模式.png)

## Per-Job

![image-20210509084201954](截图/Flink/per-job模式.png)

## Application mode

![image-20210509084345060](截图/Flink/application mode.png)



在集群的部署模式中，除了以上三种集群运行的模式支持部署外，还可以进行Native集群部署，他的特点是根据job的资源申请，动态的启动TaskManager满足计算需求。	



# Flink On Yarn

![image-20210509084853160](截图/Flink/yarn集群架构原理.png)



session模式

![image-20210509085042635](截图/Flink/flink on yarn session.png)



Per-job模式

![image-20210509085307048](../../../Library/Application Support/typora-user-images/image-20210509085307048.png)



# 时间窗口

![image-20210509220739461](截图/Flink/时间类型.png)



## WaterMark

![image-20210509221019470](截图/Flink/WaterMark.png)



作用、生成、更新、传递等



![image-20210509222114956](截图/Flink/watermark使用.png)

对于window来说，通过watermark来判断窗口统计的时机。上图，假设5分钟一个窗口，最大的延时时间为10分钟，使用的append模式。

![image-20210509223133750](../../../Library/Application Support/typora-user-images/image-20210509223133750.png)





![image-20210509223236623](截图/Flink/watermark使用总结.png)



**生成策略**

**Periodic WaterMarks**

根据最大的envent time 减去一个最大的延迟时间



**Punctuated WaterMarks**

在事件流中，基于一个固定的事件来生成



## Windows 窗口

![image-20210510215730933](截图/Flink/window抽象概念.png)



### Window Assigner

窗口分配器：用来将数据流中的元素分配到不同的窗口。

![image-20210510220246418](截图/Flink/窗口类型.png)



#### Sliding Windows

![image-20210510220420238](截图/Flink/sliding windows.png)



#### Tumbling Windows

![tumbling windows](截图/Flink/tumbling windows.png)



#### Session Windows

![session window](截图/Flink/session window.png)



### Window Trigger

![windows trigger](截图/Flink/windows trigger.png)



### Window Evictor

![image-20210510222235457](截图/Flink/windows evictor.png)



### Window Function

![image-20210510222824428](截图/Flink/window function.png)



## Windows 多流合并

多流合并类型

- window join
- Interval join





## Process function