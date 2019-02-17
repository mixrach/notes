# 功能模式
## 1. Hello world

可以将RabbitMQ想象成一个邮局，生产者将消息发送给邮局，邮局将消息分发到对应的消费者。这里有一点需要注意，消息是被分派出去，也即所谓的push模式。

![](/img/rabbitmq/hello-world.png)

## 2. 任务分发 
![](/img/rabbitmq/work-queues.png)

#### 普通分发 
可以添加多个消费者监听同一个queue，这种情况下RMQ会轮询所有的Consumer，每次只发送给一个消费者，最终每个消费者将收到相同数量的消息。
#### 消息确认
在很多时候，我们需要知道我们的消息是否被成功的处理，而不是简单的将消息发送给消费者。比如我们需要执行一个较复杂的任务，如果消息队列只是简单的将消息发送给消费者就将消息从buffer中删除，如果任务执行失败或者消费者根本就没有收到消息，这种情况下在我们对消息的可靠性要求很高的场景下是不被允许的。比如目前我们的B2C中将交易消息发给C2C，让C2C根据交易消息添加保单信息，计算奖励等。如果消息在传递过程中丢失，或者C2C服务发生了某种异常，那么这笔交易信息就没有被处理，而我们却全然不知。

因此，RMQ支持消息确认机制来解决上述问题，如果消费者挂掉（channel is closed，connection is closed, or TCP connection is lost）没有发送ACK给RMQ，这时候RMQ就会重新分发消息，给其他的消费者或者等待当前消费者重启。
#### 消息持久化
消息确认机制保证消费者挂掉之后消息不丢失。而如果RMQ挂掉，消息仍然会丢失（这是因为默认情况下，RMQ将消息内容放在buffer中）。因此RMQ支持对消息进行持久化，事实上这里有两个部分-队列持久化与消息持久化。RMQ会将声明的持久化队列写入磁盘中，并且只写队列中声明为持久化的消息。当RMQ挂掉重启时，可以将消息从磁盘中恢复

#### publisher确认与事务
消息持久化仍然存在一点问题，如果RMQ在消息由cache写入磁盘中的时候机器挂掉，消息仍然会丢失。
AMQP协议中，唯一可以保证消息没有丢失的方式是使用事务，但是使用事务会将RMQ的性能降低250倍。RabbitMQ提供了一种publisher确认方式来实现一种高性能的保证消息不丢失的功能。

#### 公平分发
前面提到RMQ会公平的轮询每个消费者为其分发消息。但是这里存在一个问题，比如在上述两个消费者的情况，如果奇数消息都是繁重的任务，而偶数消息是轻量级的任务，那么如果仍然默认轮询方式，显然不合适。RMQ支持配置是的只有在上一个消息被确认后才会给他分发新的消息，否则将消息分发给其他的空闲的消费者。这里有点类似TCP的Nagle算法。


## 3. 发布/订阅
![](/img/rabbitmq/publish-subscribe.png)
#### 路由器
> The core idea in the messaging model in RabbitMQ is that the producer never sends any messages directly to a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.Instead, the producer can only send messages to an exchange. An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the exchange type.

#### 路由类型
路由类型|解释
:---|----:
direct| 只有这两个routingkey完全相同，exchange才会选择对应的binging进行消息路由
topic| 这里的routingkey可以有通配符：'\*','#'.其中'\*'表示匹配一个单词， '#'则表示匹配没有或者多个单词
fanout| 直接将消息路由到所有绑定的队列中，无须对消息的routingkey进行匹配操作
header| 通过header中的参数判断
![](/img/rabbitmq/route.png)

![](/img/rabbitmq/topic.png)

## RPC
![](/img/rabbitmq/RPC.png)
消息发送时，会设置一个reply_to的队列名称，当server收到消息后，会将执行结果push到这个队列，而client则监听这个队列获取结果。这里有个问题：如果准确的获取返回结果
- 为每一个RPC设置一个reply_to queue
- 使用correlation_id来判断返回结果

这里有一点很重要：即RMQ在发现一个不认识的correlation_id时，会安全的丢弃这个message，之所以会这样是基于这样一种可能：
> it is possible that the RPC server will die just after sending us the answer, but before sending an acknowledgment message for the request. If that happens, the restarted RPC server will process the request again. That's why on the client we must handle the duplicate responses gracefully, and the RPC should ideally be idempotent.


# 部署架构
![](/img/rabbitmq/architecture.png) 

# 相关概念
### 信道(channel)
RabbitMQ使用信道来进行消息交互。信道建立在真实TCP连接内的虚拟连接，每个信道都会被指派一个唯一ID（AMQP库会记住此ID），AMQP命令都是通过信道发送出去的。
通过使用信道，避免了大量TCP连接的建立与销毁，对于操作系统来说建立与销毁TCP连接是非常昂贵的开销。
### Erlang语言
- 函数式语言
- 针对分布式，天然支持进程间通信；erlang中所有节点都知道其他节点的存在




