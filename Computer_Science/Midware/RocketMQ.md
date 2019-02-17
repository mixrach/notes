# 简介
>Apache RocketMQ is a distributed messaging and streaming platform with low latency, high performance and reliability, trillion-level capacity and flexible scalability.

It offers a variety of features:

- Pub/Sub messaging model 发布订阅模式
- Scheduled message delivery 定时消息分发
- Message retroactivity by time or offset 根据时间或偏移量的消息回溯
- Log hub for streaming 流式日志处理
- Big data integration 大数据继承
- Reliable FIFO and strict ordered messaging in the same queue  可靠的先进先出与消息有序性
- Efficient pull&push consumption model 高校的推/拉消费模型
- Million-level message accumulation capacity in a single queue 单队列 亿级消息堆积能力
- Multiple messaging protocols like JMS and OpenMessaging 类似JMS与openMessaging的消息协议
- Flexible distributed scale-out deployment architecture 灵活的分布式、易扩展的部署架构
- Lightning-fast batch message exchange system 轻量-快速的批量消息交换系统
- Various message filter mechanics such as SQL and Tag 多样化的消息过滤机制
- Docker images for isolated testing and cloud isolated clusters 可以用于隔离测试以及云端隔离集群的Docker镜像
- Feature-rich administrative dashboard for configuration, metrics and monitoring 功能丰富的管理控制台，可以进行配置，指标查看和监控

# RocketMQ 术语

- Producer
> 生产者，用于将消息发送到RocketMQ，生产者本身既可以是生成消息，也可以对外提供接口，由外部来调用接口，再由生产者将受到的消息发送给MQ。

- Consumer 
> 消费者，从Broker拉取消息进行消费。从应用角度来说有两类消费者：PullConsumer：主动拉取消息，一旦拉取到消息，应用的消费进程进行初始化；PushConsumer：封装消息拉取，消费进程和内部

- Broker
> RocketMQ服务器，也是整个服务的核心，它实现了消息的存储、拉取功能。它通常以集群方式启动，并可配置主从，每个broker上提供对指定topic的服务。理解了broker的原理以及它和其他服务交互的过程，也就命令消息中间件的原理，其实都大同小异。它具有2中角色 - Master：能写、能读； Slave：只能读，不能写

- Topic 
> 消息的主题，由用于定义并在服务端配置，消费者可以按照主题进行订阅，也就是消息分类，通常一个应用一个Topic

- Message
> 在生产者、消费者、服务器之间传递的消息，一个message必须属于一个Topic

- Namesrv
> 一个无状态的名称服务，可以集群部署，每一个broker启动的时候都会向名称服务器注册，主要是接收broker的注册（broker每十秒就会向所有名称服务器发送心跳请求，同时注册topic信息到名称服务器），接收客户端的路由请求并返回路由信息，你可以理解为服务自动发现，就是相当于zookeeper在dubbo框架中的作用。
> - 生产者发消息时会根据Topic向名称服务器获取到指定broker的路由信息
> - 消费者根据Topic到名称服务器获取该Topic到broker的路由信息

- Group
>组名，一类消费者或者生产者的集合名称。消费者组，消费相同Topic内容的消费者，可以并行消费Topic中Partition中的消息。生产者组，生产相同Topic内容的生产者

- Offset
> 偏移量，消费者拉取消息时需要知道上一次消费到了什么位置，这一次从哪里开始

- Partition
> 分区，Topic物理上的分组，一个Topic可以分为多个分区，每个分区是一个有序的队列。分区中的每条消息都会给分配一个有序的ID，也就是偏移量。分区的目的：
> 1. 减缓日志文件占用磁盘空间，消息需要持久化到文件，分区可以将消息粒度细分，每个分区可以存放在不同的磁盘空间中
> 2. 不同消费者同时消费分区中的数据，一个分区仅由一个消费者组中的消费者消费，1个消费者可以同时消费多个分区。
> 3. 可以实现负载均衡，如果同一个Topic的消息都放在同一个Broker上，那消费的时候同一个Topic的消费者都去同一个Broker上消费，这样会带来压力，如果通过分区放在不同Broker上，这样就可以到不同的Broker上消费，当然同一个ID的消息只能存在一个分区上。你可以想象A这个topic的消息有10个那么每个消息有1个ID，如果分布10个消息分布在不同的分区上，比如3个，那就形成3-3-4，消费者去消费的时候消费10条消息时通过3个分区完成这样就提高了吞吐量。
> 4. Topic是消息的逻辑队列，分区是物理队列。可以通过配置文件来设置topic的默认分区数量，也可以在新建立topic的时候指定。建议分区数量和消费者数量一致，因为消费者数量多，多出来的不会去消费消息的，因为一个队列只能被一个消费者消费。如果消费者数量少则消费者就会比较繁忙。

- Tag 
> 用于对消息进行过滤，理解文件message的子主题，同一业务不同目的的message可以用相同的topic但是可以用不同的tag来区分，在队列中tag在消息的数据结构中被 转换为一个8byte的hashcode，这样节省空间。过滤分两步：
> 1. 在Broker端进行Message Tag对比，先遍历Consume Queue，如果存储的Message tag与订阅的tag不符合就跳过，符合则传输给Consumer，在队列中继续比对hashcode
> 2. Consumer收到消息后，对比真实的Message Tag字符串，而不是Hashcode，这样避免HASH冲突。

- key 
> 消息的KEY字段是为了唯一表示消息的，方便查问题，不是说必须设置，只是说设置为了方便开发和运维定位问题，这个KEY可以是订单ID等。

# [十分钟入门RocketMQ](http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/)

# RocketMQ 与 Kafka

#### 1. 去除Producer端多个小消息合并-批量发送功能
> RocketMQ为什么没有这么做？
> - Producer通常使用Java语言，缓存过多消息，GC是个很严重的问题
> - Producer调用发送消息接口，消息未发送到Broker，向业务返回成功，此时Producer宕机，会导致消息丢失，业务出错
> - Producer通常为分布式系统，且每台机器都是多线程发送，我们认为线上的系统单个Producer每秒产生的数据量有限，不可能上万。
> - 缓存的功能完全可以由上层业务完成。
#### 2. 增加单机的队列/分区个数

#### 3. 使用长轮询保证消息延时通常在几个毫秒

#### 4. 支持消息失败重试

#### 5. 严格的消息顺序

#### 6. 支持定时消息

#### 7. 支持消息查询

#### 8. Broker端消息过滤


# 部署架构
![](/img/RocketMQ/推荐架构.png)

# 相关概念

#### 短轮询
   
短轮询指的是在循环周期内，不断发起请求，每一次请求都立即返回结果，根据新旧数据对比决定是否使用这个结果。

#### 长轮询

是在请求的过程中，若是服务器端数据并没有更新，那么则将这个连接挂起，直到服务器推送新的数据，再返回，然后再进入循环周期