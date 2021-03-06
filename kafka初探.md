> 项目所需，所以简单了解下。简单及实用，kafka也不是个上手很难的东西，你可以说它是个消息队列（sender/receiver），也可是是一个sub/pub模型，这无所谓，只是思想的侧重点有所不同。

kafka是什么：`A distributed streaming platform`，分布式流平台。分布式意味着它以集群的方式运行，且*易于拓展，弹性较好*。官网首页提到三点。

***PUBLISH & SUBSCRIBE***

to streams of data like a messaging system。意为你可以简单的把它当做为一个简单的订阅者和发布者模型。在前端很常用。

***PROCESS***

streams of data efficiently and in real time。高效，实时（谁也不能说自己不好吧。。）。stream意味着它在服务之间建设了类似`pipelines`的机制，可获得类似`read.pipe(write)`的好处，使对应的效率、可靠程度和资源使用都降低。

***STORE***

streams of data safely in a distributed replicated cluster。说明它有安全保障特性。

上面几点的大体意思就是，*简单，高效，安全*。

##

存在即合理，kafka解决的问题是？

***降低后端系统的连接复杂度***

比如最开始你的web服务器可能只需要依赖一个数据库服务，当业务越来越复杂，每个附属服务都需要与web服务器单独连接。这样做会有一些问题。***消息冗余***，这条消息需要传达给两个附属服务，直连发送冗余消息。**数据同步的挑战**（暂时无法理解）。那么把kafka加到web服务器和附属服务之间则解决了上面两个问题。总之来讲是**解耦**。

***防止数据丢失***

应对消息激增时服务丢弃消息的情况，相当于做了一层缓冲。

##

那么应用场景是？

***实时监控与分析***

事实上团队内也在搭建实时计算系统中使用kafka。第一足够快，第二降低了消息丢失率，第三服务间解耦了。

***传统的消息系统***

上面也说了，***PUBLISH & SUBSCRIBE***。搭配websocket类似的服务器推技术实现实时性更好的应用。

***分布式应用程序或平台的构件***

网上说的，总之被看中对于kafka也是好事。

##

核心概念

***topic***

用于将记录进行分类。topic中的每个record只有一个元信息，那就是offset。

***partition***

parition是物理上的概念，每个topic包含一个或多个partition，创建topic时可指定parition数量。每个partition对应于一个文件夹，该文件夹下存储该partition的数据和索引文件。每个partition有它的start值，存了某一段offset的record。这样是要提高查询效率。分partition的策略可以使单个topic获得水平拓展的能力。如果一个topic对应一个文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈。

逻辑上还是只需要关心topic的概念，producer复制将数据发往不同的分区。

***segment***

partition又分为多个段（segment），每次文件操作都是对一个小文件进行操作，所以非常轻便，而且也增加了并行处理的能力。 

***record***

记录，由一个key，value，和时间戳组成。（为什么跟我在公司的kafka admin上看到的不一样）。会根据时间戳来判断是否过期。

##

五种角色。第一个是kafka集群，下面四种都需要在集群之上工作。

![](/images/1492846976xr.png)

***cluster***

kafka集群。集群中的每台机器分别为一个*Broker*。各个broker之间是没有主从关系的，可以随意的增加或删除任何一个broker节点。

***connector***


***stream processor***

node中传输流中的processor的概念，在消费之前做预处理。

***producer***

产生record流。需要指定分区，如果要均衡到各个分区采用简单的round-robin即可。

***consumer***

消费record记录。consumer使用组的概念标识自身，消费进程可能在不同机器上，同组之间负载均衡，不同组之间广播。同组的几个消费进程称为一个“逻辑上的订阅者”。

消费需要指定topic和起始的offset。指定partition则在该partition消费，不指定则会将partition分散到customer。集群会自动帮你记录当前消费到的位置，返回你当前消费的位置，确认消费是帮助消费者重启时不重复消费，集群只保证不少消费，不保证重复消费。

一个partition只能被一个消费者消费（一个消费者可以同时消费多个partition），因此，如果设置的partition的数量小于consumer的数量，就会有消费者消费不到数据。所以，推荐partition的数量一定要大于同时运行的consumer的数量。

##

关键策略

***过期策略设计***

不会想想象那样消费即删除，会有相应的过期策略。kafka是否删除record跟消息是否被消费一点关系都没有。有两种过期策略。一是基于时间，二是基于partition文件大小。

***消费者策略***

Kafka提供了两套consumer api，分为high-level api和sample-api。Sample-api 是一个底层的API，它维持了一个和单一broker的连接，并且这个API是完全无状态的，每次请求都需要指定offset值，因此，这套API也是**最灵活**的。

很多时候，客户程序只是希望从Kafka读取数据，不太关心消息offset的处理。同时也希望提供一些语义，例如同一条消息只被某一个Consumer消费（单播）或被所有Consumer消费（广播）。因此，Kafka Hight Level Consumer提供了一个从Kafka消费数据的高层抽象，从而屏蔽掉其中的细节并提供丰富的语义。但需要Consumer Rebalance 管理 consumers 和 partitions，而且无法提供原子性的消费确认。

10版本之后建议不使用high-level了。

http://www.jasongj.com/2015/08/09/KafkaColumn4/
http://zqhxuyuan.github.io/2016/02/20/Kafka-Consumer-New/

***消费确认***

使用集群做消息确认很难做到原子性，所以在consumer重启的时候可能会出现重复消费的情况，所以需要将消费和消费确认做成原子性的操作。http://blog.csdn.net/chunlongyu/article/details/52663090（关闭消息确认）。这样获取重启前的offset就可以更精确的控制重复消费了。

***消息副本***

一般会至少设置副本（Replications）数为2，这样一台机器挂了之后还有另一台机器。现在的话每个partition会存放在不同的机器上，这样就需要一个leader partition。

***压缩***

Kafka还支持对消息集合进行压缩，Producer端可以通过GZIP或Snappy格式对消息集合进行压缩。Producer端进行压缩之后，在Consumer端需进行解压。压缩的好处就是减少传输的数据量，减轻对网络传输的压力，在对大数据处理上，瓶颈往往体现在网络上而不是CPU（压缩和解压会耗掉部分CPU资源）。

## 

在公司中？

***公共服务***

上面那么多在实际工作中或许统统用不上。一般在大厂都会标配的数据研发组，他们帮前端工程师解决了很多的问题，让工程师能投身于业务本身，快速的获得数据服务带来的收益。

这个过程仿佛十分简单，申请个topic，再使用对应语言的sdk接入就ok了。
