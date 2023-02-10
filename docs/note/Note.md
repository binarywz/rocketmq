# RocketMQ源码笔记

### 启动类设置

```
# NameServer
org.apache.rocketmq.namesrv.NamesrvStartup

# Evironment variables
ROCKETMQ_HOME=$PROJECT_DIR$/config
```

![1-1](img\1-1.png)

```
# Broker
org.apache.rocketmq.broker.BrokerStartup

# 启动参数
-c $PROJECT_DIR$/config/conf/broker.conf

# Evironment variables
ROCKETMQ_HOME=$PROJECT_DIR$/config
```

![1-2](img\1-2.png)

### RocketMQ Reactor多线程设计

![2-1](img\2-1.png)

从上面的框图中可以大致了解RocketMQ中NettyRemotingServer的Reactor 多线程模型。一个 Reactor 主线程（eventLoopGroupBoss，即为上面的1）负责监听 TCP网络连接请求，建立好连接，创建SocketChannel，并注册到selector上。RocketMQ的源码中会自动根据OS的类型选择NIO和Epoll，也可以通过参数配置），然后监听真正的网络数据。拿到网络数据后，再丢给Worker线程池（eventLoopGroupSelector，即为上面的“N”，源码中默认设置为3），在真正执行业务逻辑之前需要进行SSL验证、编解码、空闲检查、网络连接管理，这些工作交给defaultEventExecutorGroup（即为上面的“M1”，源码中默认设置为8）去做。而处理业务操作放在业务线程池中执行，根据 RomotingCommand 的业务请求码code去processorTable这个本地缓存变量中找到对应的 processor，然后封装成task任务后，提交给对应的业务processor处理线程池来执行（sendMessageExecutor，以发送消息为例，即为上面的 “M2”）。从入口到业务逻辑的几个步骤中线程池一直再增加，这跟每一步逻辑复杂性相关，越复杂，需要的并发通道越宽。

| 线程数 | 线程名                         | 线程具体说明            |
| ------ | ------------------------------ | ----------------------- |
| 1      | NettyBoss_%d                   | Reactor 主线程          |
| N      | NettyServerEPOLLSelector_%d_%d | Reactor 线程池          |
| M1     | NettyServerCodecThread_%d      | Worker线程池            |
| M2     | RemotingExecutorThread_%d      | 业务processor处理线程池 |

### Broker启动流程源码解析

Broker主要负责消息的存储，投递和查询以及服务高可用保证，为了实现这些功能，Broker包含了以下重要子模块：

- Remoting Module：整个Broker的实体，负责处理来自Client端的请求
- Client Manager：负责管理客户端（Producer/Consumer）和维护Consumer的Topic订阅信息
- Store Service：提供方便的API接口处理消息存储到物理硬盘和查询功能
- HA Service：高可用服务，提供Master Broker和Slave Broker之间的数据同步功能
- Index Service：根据特定的Message Key对投递到Broker的消息进行索引服务，以提供消息的快速查询

### CommitLog文件简介

CommitLog文件是RocketMQ真正存储消息内容的地方，即消息主体及元数据的存储主体，存储Producer端写入的消息主体内容，消息内容是不定长的。

官方描述如下：单个文件大小默认1G，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息顺序写入日志文件，效率很高，当文件满了，写入下一个文件。

RocketMQ会进行commitlog文件预创建，如果启用了MappedFile（MappedFile类可以看作是commitlog文件在Java中的抽象）预分配服务，那么在创建MappedFile时会同时创建两个MappedFile，一个同步创建并返回用于本次实际使用，一个后台异步创建用于下次取用。这样的好处是避免等到当前文件真正用完了才创建下一个文件，目的同样是提升性能。

### ConsumeQueue文件简介

官方描述如下：消息消费队列（可以理解为Topic中的队列），引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。

Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值，以及ConsumeOffset（每个消费者组的消费位置）。

consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：topic/queue/file三层组织结构，具体存储路径为：

```
$ROCKETMQ_HOME/store/consumequeue/{topic}/{queueId}/{fileName}
```

同样consumequeue文件中的条目采取定长设计，每个条目共20字节，分别为8字节的commitlog物理偏移量、4字节的消息长度、8字节tag hashcode，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个ConsumeQueue文件大小约5.72M。

ConsumeQueue名字长度为20位，左边补零，剩余为起始偏移量；比如00000000000000000000代表第一个文件，起始偏移量为0，文件大小为600w，当第一个文件满之后创建的第二个文件的名字为00000000000006000000，起始偏移量为6000000，以此类推，消息存储的时候会顺序写入文件，当文件写满了，写入下一个文件。

### Index索引文件简介

索引文件`IndexFile`提供了一种可以通过key或事件区间来查询方法的方法。Index文件的存储位置是`$ROCKETMQ_HOME/store/index/{filename}`，文件名`filename`是以创建的时间戳命名的，固定的单个`IndexFile`文件大小约为400M，一个`IndexFile`可以保存2000w个索引，`IndexFile`的底层存储设计为在文件系统中实现`HashMap`结构，估`RocketMQ`的索引文件其底层实现为`hash`索引

