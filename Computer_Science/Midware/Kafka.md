# 简介
A streaming platform has three key capabilities:

- Publish and subscribe to streams of records, similar to a message queue or enterprise messaging system.
- Store streams of records in a fault-tolerant durable way.
- Process streams of records as they occur.

Kafka is generally used for two broad classes of applications:

- Building real-time streaming data pipelines that reliably get data between systems or applications
- Building real-time streaming applications that transform or react to the streams of data

To understand how Kafka does these things, let's dive in and explore Kafka's capabilities from the bottom up.

First a few concepts:

- Kafka is run as a cluster on one or more servers that can span multiple datacenters.
- The Kafka cluster stores streams of records in categories called topics.
Each record consists of a key, a value, and a timestamp.

Kafka has four core APIs:

- The Producer API allows an application to publish a stream of records to one or more Kafka topics.
- The Consumer API allows an application to subscribe to one or more topics and process the stream of records produced to them.
- The Streams API allows an application to act as a stream processor, consuming an input stream from one or more topics and producing an output stream to one or more output topics, effectively transforming the input streams to output streams.
- The Connector API allows building and running reusable producers or consumers that connect Kafka topics to existing applications or data systems. For example, a connector to a relational database might capture every change to a table.

Features

- 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间复杂度的访问性能。
- 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条以上消息的传输。
- 支持Kafka Server间的消息分区，及分布式消费，同时保证每个Partition内的消息顺序传输。
- 同时支持离线数据处理和实时数据处理。
- Scale out：支持在线水平扩展。

## Kafka拓扑结构
![](/img/kafka/kafka_diagram.png)

## 高性能关键技术
### 宏观架构层面
- 利用Partition实现并行处理

> Topic是一个逻辑概念，每个Topic都包含一个或多个Partition，不同的Partition可位于不同的节点，同时Patition在物理上对应一个本地文件夹，每个Partition包含一个或多个Segment，每个Segment包含一个数据文件和一个与之对应的索引文件。可以把Partition当做一个非常长的数组，可通过这个“数组”的索引去访问数据,partition可以位于不同的机器，可以充分利用集群优势
可以通过配置让统一节点上不同的Partition置于不同的disk drive上，从而实现磁盘间的并行处理，充分发挥多磁盘的优势

![](/img/kafka/partition.png)


- 基于ISR的数据复制方案

> kafka数据复制接近Master-Slave方案，不同的是，其既不是完全的同步复制，也不是完全的异步复制，而是基于ISR的动态复制方案。ISR，也即In-sync Repilca.每个Partition的leader都会维护这样一个列表，该列表中，包含了所有与之同步的Replica。每次数据写入时，只有ISR中的所有的Replica都复制完，Leader才会将其置位Commit，他才能被Consumer消费。

![ISR复制方案 ](/img/kafka/ISR复制方案.png)

## 具体实现层面
### 高效使用磁盘

- 顺序写磁盘 - 一些场景下顺序写磁盘快于随机写内存
- 批量删除数据 - 不会直接删除被消费的消息，而是等待一个Segment一起删除
- 充分利用Page Cache: I/O Scheduler会将连续的小块写组装成大块的物理写从而提高性能；I/O Scheduler会将一些写操作排序，从而减少磁头的移动时间；充分利用所有空闲内存（非JVM内存）；读操作可以直接在Page Cache内进行；如果重启，JVM内的Cache会失效，但是Page Cache仍然可用;如果数据消费速度与生产速度相当，甚至不需要通过物理磁盘交换数据，而是直接通过Page Cache交换数据。
- 支持多Disk Drive
- 零拷贝

### 减少网络开销
- 批处理
- 数据压缩
- 高效的序列化方式

## 数据一致性保证

Kafka producer的ack有3中机制，初始化producer时的producerconfig可以通过配置request.required.acks不同的值来实现。

- 0：这意味着生产者producer不等待来自broker同步完成的确认继续发送下一条（批）消息。

此选项提供最低的延迟但最弱的耐久性保证（当服务器发生故障时某些数据会丢失，如leader已死，但producer并不知情，发出去的信息broker就收不到）。

- 1：这意味着producer在leader已成功收到的数据并得到确认后发送下一条message。

此选项提供了更好的耐久性为客户等待服务器确认请求成功（被写入死亡leader但尚未复制将失去了唯一的消息）。

- -1：这意味着producer在follower副本确认接收到数据后才算一次发送完成。

此选项提供最好的耐久性，我们保证没有信息将丢失，只要至少一个同步副本保持存活。

三种机制，性能依次递减 (producer吞吐量降低)，数据健壮性则依次递增。

# 部署架构
![](/img/kafka/kafka_architecture.png)


