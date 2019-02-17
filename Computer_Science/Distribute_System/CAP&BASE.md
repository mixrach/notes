# CAP理论
CAP定理，指在一个分布式系统中，Consistency、Availability、Partion tolerance三者不可得兼。
三个部分的解释如下：
> - Consistency: Every read receives the most recent write or an error 
> - Availability: Every request receives a (non-error) response – without guarantee that it contains the most recent write
> - Partition tolerance: The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes

CAP理论说在分布式系统中，最多能实现上面两点，而由于网络肯定会出现延迟丢包等问题，所以P是必选项，所以我们只能在A和C之间权衡。

# CAP 的简单证明
我们可以想象两台计算机a、b，每台计算机有一个数据库，两个数据库中的数据是同步的。
- CA：即任何时候访问两个数据库得到的结果是一样的，这就要求我们写入a的数据必须瞬间同步到b，可见这是非常理想的情况。在这种理想情况下，一定不可以发生分区，即我们不能容忍分区，因为分区就会造成在一个时间点两个数据库的数据可能会不一致。
- CP：即我们承认网络延迟带来的分区是必然存在的，那么我们在P的基础上实现数据的一致性，这就需要在发生P的时候我们停止对外服务转而进行数据的同步，当同步完成后才能恢复对外服务。显然我们这里必须牺牲可用性。
- AP：同样我们承认分区的必然性，我们想要保证系统一直运行，那么在网络延迟或故障的情况下，必然存在数据不一致的时刻，即这里我们牺牲掉了一致性。
