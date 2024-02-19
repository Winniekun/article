# [Kafka 概览](https://github.com/Winniekun/article/issues/10)

## 简介
 Kafka 是一个分布式的基于发布/订阅模式的消息队列（Message Queue），主要应用于大数据实时处理领域。其主要设计目标如下：
- 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能
- 高吞吐率。即使在非常廉价的机器上也能做到单机支持每秒100K条消息的传输
- 支持Kafka Server间的消息分区，及分布式消费，同时保证每个partition内的消息顺序传输，同时支持离线数据处理和实时数据处理

## 为什么要用消息系统
Kafka 本质上是一个 MQ（Message Queue），使用消息队列的好处？
- **解耦**：允许我们独立修改队列两边的处理过程而互不影响。
- **冗余**：有些情况下，我们在处理数据的过程会失败造成数据丢失。消息队列把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险, 确保你的数据被安全的保存直到你使用完毕
- **峰值处理能力**：不会因为突发的流量请求导致系统崩溃，消息队列能够使服务顶住突发的访问压力, 有助于解决生产消息和消费消息的处理速度不一致的情况
- **异步通信**：消息队列允许用户把消息放入队列但不立即处理它, 等待后续进行消费处理。

## 基础知识
- Producer：即消息生产者，向 Kafka Broker 发消息的客户端。
- Consumer：即消息消费者，从 Kafka Broker 读消息的客户端。
- Consumer Group：即消费者组，消费者组内每个消费者负责消费不同分区的数据，以提高消费能力。一个分区只能由组内一个消费者消费，不同消费者组之间互不影响。
- Broker：一台 Kafka 机器就是一个 Broker。一个集群是由多个 Broker 组成的且一个 Broker 可以容纳多个 Topic。
- Topic：可以简单理解为队列，Topic 将消息分类，生产者和消费者面向的都是同一个 Topic。
- Partition：为了实现Topic扩展性，提高并发能力，一个非常大的 Topic 可以分布到多个 Broker 上，一个 Topic 可以分为多个 Partition 进行存储，每个 Partition 是一个有序的队列。
- Replica：即副本，为实现数据备份的功能，保证集群中的某个节点发生故障时，该节点上的 Partition 数据不丢失，且 Kafka 仍然能够继续工作，为此Kafka提供了副本机制，一个 Topic 的每个 Partition 都有若干个副本，一个 Leader 副本和若干个 Follower 副本。
- Leader：即每个分区多个副本的主副本，生产者发送数据的对象，以及消费者消费数据的对象，都是 Leader。
- Follower：即每个分区多个副本的从副本，会实时从 Leader 副本中同步数据，并保持和 Leader 数据的同步。Leader 发生故障时，某个 Follower 还会被选举并成为新的 Leader , 且不能跟 Leader 在同一个broker上, 防止崩溃数据可恢复。
- Offset：消费者消费的位置信息，监控数据消费到什么位置，当消费者挂掉再重新恢复的时候，可以从消费位置继续消费。
- ZooKeeper服务：Kafka 集群能够正常工作，需要依赖于 ZooKeeper，ZooKeeper 帮助 Kafka 存储和管理集群元数据信息。在最新版本中, 已经慢慢要脱离 ZooKeeper。


## References
- [Kafka](https://zh.wikipedia.org/zh-hans/Kafka#:~:text=Kafka%E6%98%AF%E7%94%B1Apache%E8%BD%AF%E4%BB%B6,%E5%BC%8F%E6%95%B0%E6%8D%AE%E9%9D%9E%E5%B8%B8%E6%9C%89%E4%BB%B7%E5%80%BC%E3%80%82)