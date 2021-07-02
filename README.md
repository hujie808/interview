# 面试问题整理
## ZooKeeper
### CAP定理：

一个分布式系统不可能在满足分区容错性（P）的情况下同时满足一致性（C）和可用性（A）。在此ZooKeeper保证的是CP，ZooKeeper不能保证每次服务请求的可用性，在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果。另外在进行leader选举时集群都是不可用，所以说，ZooKeeper不能保证服务可用性。

### BASE理论

BASE理论是基本可用，软状态，最终一致性三个短语的缩写。BASE理论是对CAP中一致性和可用性（CA）权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的，它大大降低了我们对系统的要求。
1. 基本可用：基本可用是指分布式系统在出现不可预知故障的时候，允许损失部分可用性。但是，这绝不等价于系统不可用。比如正常情况下，一个在线搜索引擎需要在0.5秒之内返回给用户相应的查询结果，但由于出现故障，查询结果的响应时间增加了1~2秒。
2. 软状态：软状态指允许系统中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时。
3. 最终一致性：最终一致性强调的是系统中所有的数据副本，在经过一段时间的同步后，最终能够达到一个一致的状态。因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不需要实时保证系统数据的强一致性。

### ZooKeeper特点

1. 顺序一致性：同一客户端发起的事务请求，最终将会严格地按照顺序被应用到 ZooKeeper 中去。
2. 原子性：所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说，要么整个集群中所有的机器都成功应用了某一个事务，要么都没有应用。
3. 单一系统映像：无论客户端连到哪一个 ZooKeeper 服务器上，其看到的服务端数据模型都是一致的。
4. 可靠性：一旦一次更改请求被应用，更改的结果就会被持久化，直到被下一次更改覆盖。

### ZAB协议：

ZAB协议包括两种基本的模式：崩溃恢复和消息广播。当整个 Zookeeper 集群刚刚启动或者Leader服务器宕机、重启或者网络故障导致不存在过半的服务器与 Leader 服务器保持正常通信时，所有服务器进入崩溃恢复模式，首先选举产生新的 Leader 服务器，然后集群中 Follower 服务器开始与新的 Leader 服务器进行数据同步。当集群中超过半数机器与该 Leader 服务器完成数据同步之后，退出恢复模式进入消息广播模式，Leader 服务器开始接收客户端的事务请求生成事物提案（超过半数同意）来进行事务请求处理。

### 选举算法和流程：FastLeaderElection(默认提供的选举算法)

目前有5台服务器，每台服务器均没有数据，它们的编号分别是1,2,3,4,5,按编号依次启动，它们的选择举过程如下：
1. 服务器1启动，给自己投票，然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，服务器1的状态一直属于Looking。
2. 服务器2启动，给自己投票，同时与之前启动的服务器1交换结果，由于服务器2的编号大所以服务器2胜出，但此时投票数没有大于半数，所以两个服务器的状态依然是LOOKING。
3. 服务器3启动，给自己投票，同时与之前启动的服务器1,2交换信息，由于服务器3的编号最大所以服务器3胜出，此时投票数正好大于半数，所以服务器3成为leader，服务器1,2成为follower。
4. 服务器4启动，给自己投票，同时与之前启动的服务器1,2,3交换信息，尽管服务器4的编号大，但之前服务器3已经胜出，所以服务器4只能成为follower。
5. 服务器5启动，后面的逻辑同服务器4成为follower。

### zk中的监控原理

zk类似于linux中的目录节点树方式的数据存储，即分层命名空间，zk并不是专门存储数据的，它的作用是主要是维护和监控存储数据的状态变化，通过监控这些数据状态的变化，从而可以达到基于数据的集群管理，zk中的节点的数据上限时1M。

client端会对某个znode建立一个watcher事件，当该znode发生变化时，这些client会收到zk的通知，然后client可以根据znode变化来做出业务上的改变等。

### zk实现分布式锁

zk实现分布式锁主要利用其临时顺序节点，实现分布式锁的步骤如下：

1. 创建一个目录mylock
2. 线程A想获取锁就在mylock目录下创建临时顺序节点
3. 获取mylock目录下所有的子节点，然后获取比自己小的兄弟节点，如果不存在，则说明当前线程顺序号最小，获得锁
4. 线程B获取所有节点，判断自己不是最小节点，设置监听比自己次小的节点
5. 线程A处理完，删除自己的节点，线程B监听到变更事件，判断自己是不是最小的节点，如果是则获得锁

## Redis

### 应用场景

1. 缓存
2. 共享Session
3. 消息队列系统
4. 分布式锁

### 单线程的Redis为什么快

1. 纯内存操作
2. 单线程操作，避免了频繁的上下文切换
3. 合理高效的数据结构
4. 采用了非阻塞I/O多路复用机制（有一个文件描述符同时监听多个文件描述符是否有数据到来）

### Redis 的数据结构及使用场景

1. String字符串:字符串类型是 Redis 最基础的数据结构，首先键都是字符串类型，而且 其他几种数据结构都是在字符串类型基础上构建的，我们常使用的 set key value 命令就是字符串。常用在缓存、计数、共享Session、限速等。
2. Hash哈希:在Redis中，哈希类型是指键值本身又是一个键值对结构，哈希可以用来存放用户信息，比如实现购物车。
3. List列表（双向链表）:列表（list）类型是用来存储多个有序的字符串。可以做简单的消息队列的功能。
4. Set集合：集合（set）类型也是用来保存多个的字符串元素，但和列表类型不一 样的是，集合中不允许有重复元素，并且集合中的元素是无序的，不能通过索引下标获取元素。利用 Set 的交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。
5. Sorted Set有序集合（跳表实现）：Sorted Set 多了一个权重参数 Score，集合中的元素能够按 Score 进行排列。可以做排行榜应用，取 TOP N 操作。

### Redis 的数据过期策略

Redis 中数据过期策略采用定期删除+惰性删除策略
* 定期删除策略：Redis 启用一个定时器定时监视所有的 key，判断key是否过期，过期的话就删除。这种策略可以保证过期的 key 最终都会被删除，但是也存在严重的缺点：每次都遍历内存中所有的数据，非常消耗 CPU 资源，并且当 key 已过期，但是定时器还处于未唤起状态，这段时间内 key 仍然可以用。
* 惰性删除策略：在获取 key 时，先判断 key 是否过期，如果过期则删除。这种方式存在一个缺点：如果这个 key 一直未被使用，那么它一直在内存中，其实它已经过期了，会浪费大量的空间。
* 这两种策略天然的互补，结合起来之后，定时删除策略就发生了一些改变，不在是每次扫描全部的 key 了，而是随机抽取一部分 key 进行检查，这样就降低了对 CPU 资源的损耗，惰性删除策略互补了为检查到的key，基本上满足了所有要求。但是有时候就是那么的巧，既没有被定时器抽取到，又没有被使用，这些数据又如何从内存中消失？没关系，还有内存淘汰机制，当内存不够用时，内存淘汰机制就会上场。淘汰策略分为：
    1. 当内存不足以容纳新写入数据时，新写入操作会报错。（Redis 默认策略）
    2. 当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 Key。（LRU推荐使用）
    3. 当内存不足以容纳新写入数据时，在键空间中，随机移除某个 Key。
    4. 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 Key。这种情况一般是把 Redis 既当缓存，又做持久化存储的时候才用。
    5. 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 Key。
    6. 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 Key 优先移除。

### Redis的set和setnx

Redis中setnx不支持设置过期时间，做分布式锁时要想避免某一客户端中断导致死锁，需设置lock过期时间，在高并发时 setnx与 expire 不能实现原子操作，如果要用，得在程序代码上显示的加锁。使用SET代替SETNX ，相当于SETNX+EXPIRE实现了原子性，不必担心SETNX成功，EXPIRE失败的问题。

### Redis的LRU具体实现：

传统的LRU是使用栈的形式，每次都将最新使用的移入栈顶，但是用栈的形式会导致执行select *的时候大量非热点数据占领头部数据，所以需要改进。Redis每次按key获取一个值的时候，都会更新value中的lru字段为当前秒级别的时间戳。Redis初始的实现算法很简单，随机从dict中取出五个key,淘汰一个lru字段值最小的。在3.0的时候，又改进了一版算法，首先第一次随机选取的key都会放入一个pool中(pool的大小为16),pool中的key是按lru大小顺序排列的。接下来每次随机选取的keylru值必须小于pool中最小的lru才会继续放入，直到将pool放满。放满之后，每次如果有新的key需要放入，需要将pool中lru最大的一个key取出。淘汰的时候，直接从pool中选取一个lru最小的值然后将其淘汰。

### Redis如何发现热点key

1. 凭借经验，进行预估：例如提前知道了某个活动的开启，那么就将此Key作为热点Key。
2. 服务端收集：在操作redis之前，加入一行代码进行数据统计。
3. 抓包进行评估：Redis使用TCP协议与客户端进行通信，通信协议采用的是RESP，所以自己写程序监听端口也能进行拦截包进行解析。
4. 在proxy层，对每一个 redis 请求进行收集上报。
5. Redis自带命令查询：Redis4.0.4版本提供了redis-cli –hotkeys就能找出热点Key。（如果要用Redis自带命令查询时，要注意需要先把内存逐出策略设置为allkeys-lfu或者volatile-lfu，否则会返回错误。进入Redis中使用config set maxmemory-policy allkeys-lfu即可。）

### Redis的热点key解决方案

1. 服务端缓存：即将热点数据缓存至服务端的内存中.(利用Redis自带的消息通知机制来保证Redis和服务端热点Key的数据一致性，对于热点Key客户端建立一个监听，当热点Key有更新操作的时候，服务端也随之更新。)
2. 备份热点Key：即将热点Key+随机数，随机分配至Redis其他节点中。这样访问热点key的时候就不会全部命中到一台机器上了。

### 如何解决 Redis 缓存雪崩问题

1. 使用 Redis 高可用架构：使用 Redis 集群来保证 Redis 服务不会挂掉
2. 缓存时间不一致，给缓存的失效时间，加上一个随机值，避免集体失效
3. 限流降级策略：有一定的备案，比如个性推荐服务不可用了，换成热点数据推荐服务

### 如何解决 Redis 缓存穿透问题

1. 在接口做校验
2. 存null值（缓存击穿加锁,或设置不过期）
3. 布隆过滤器拦截： 将所有可能的查询key 先映射到布隆过滤器中，查询时先判断key是否存在布隆过滤器中，存在才继续向下执行，如果不存在，则直接返回。布隆过滤器将值进行多次哈希bit存储，布隆过滤器说某个元素在，可能会被误判。布隆过滤器说某个元素不在，那么一定不在。

### Redis的持久化机制

Redis为了保证效率，数据缓存在了内存中，但是会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件中，以保证数据的持久化。Redis的持久化策略有两种：

1. RDB：快照形式是直接把内存中的数据保存到一个dump的文件中，定时保存，保存策略。当Redis需要做持久化时，Redis会fork一个子进程，子进程将数据写到磁盘上一个临时RDB文件中。当子进程完成写临时文件后，将原来的RDB替换掉。
2. AOF：把所有的对Redis的服务器进行修改的命令都存到一个文件里，命令的集合。

使用AOF做持久化，每一个写命令都通过write函数追加到appendonly.aof中。aof的默认策略是每秒钟fsync一次，在这种配置下，就算发生故障停机，也最多丢失一秒钟的数据。
缺点是对于相同的数据集来说，AOF的文件体积通常要大于RDB文件的体积。根据所使用的fsync策略，AOF的速度可能会慢于RDB。
Redis默认是快照RDB的持久化方式。对于主从同步来说，主从刚刚连接的时候，进行全量同步（RDB）；全同步结束后，进行增量同步(AOF)。

### Redis的事务

1. Redis 事务的本质是一组命令的集合。事务支持一次执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序串行化执行队列中的命令，其他客户端提交的命令请求不会插入到事务执行命令序列中。总结说：redis事务就是一次性、顺序性、排他性的执行一个队列中的一系列命令。
2. Redis事务没有隔离级别的概念，批量操作在发送 EXEC 命令前被放入队列缓存，并不会被实际执行，也就不存在事务内的查询要看到事务里的更新，事务外查询不能看到。
3. Redis中，单条命令是原子性执行的，但事务不保证原子性，且没有回滚。事务中任意命令执行失败，其余的命令仍会被执行。

### Redis事务相关命令

1. watch key1 key2 ... : 监视一或多个key,如果在事务执行之前，被监视的key被其他命令改动，则事务被打断（类似乐观锁）
2. multi : 标记一个事务块的开始（queued）
3. exec : 执行所有事务块的命令（一旦执行exec后，之前加的监控锁都会被取消掉）
4. discard : 取消事务，放弃事务块中的所有命令
5. unwatch : 取消watch对所有key的监控

### Redis和 memcached 的区别

1. 存储方式上：memcache会把数据全部存在内存之中，断电后会挂掉，数据不能超过内存大小。redis有部分数据存在硬盘上，这样能保证数据的持久性。
2. 数据支持类型上：memcache对数据类型的支持简单，只支持简单的key-value，，而redis支持五种数据类型。
3. 用底层模型不同：它们之间底层实现方式以及与客户端之间通信的应用协议不一样。redis直接自己构建了VM机制，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求。
4. value的大小：redis可以达到1GB，而memcache只有1MB。

### Redis的几种集群模式

1. 主从复制
2. 哨兵模式
3. cluster模式

### Redis的哨兵模式

哨兵是一个分布式系统,在主从复制的基础上你可以在一个架构中运行多个哨兵进程,这些进程使用流言协议来接收关于Master是否下线的信息,并使用投票协议来决定是否执行自动故障迁移,以及选择哪个Slave作为新的Master。

每个哨兵会向其它哨兵、master、slave定时发送消息,以确认对方是否活着,如果发现对方在指定时间(可配置)内未回应,则暂时认为对方已挂(所谓的”主观认为宕机”)。

若“哨兵群“中的多数sentinel,都报告某一master没响应,系统才认为该master"彻底死亡"(即:客观上的真正down机),通过一定的vote算法,从剩下的slave节点中,选一台提升为master,然后自动修改相关配置。

### Redis的rehash

Redis的rehash 操作并不是一次性、集中式完成的，而是分多次、渐进式地完成的，redis会维护维持一个索引计数器变量rehashidx来表示rehash的进度。

这种渐进式的 rehash 避免了集中式rehash带来的庞大计算量和内存操作，但是需要注意的是redis在进行rehash的时候，正常的访问请求可能需要做多要访问两次hashtable（ht[0]， ht[1]），例如键值被rehash到新ht1，则需要先访问ht0，如果ht0中找不到，则去ht1中找。

### Redis的hash表被扩展的条件

1. 哈希表中保存的key数量超过了哈希表的大小.
2. Redis服务器目前没有在执行BGSAVE命令（rdb）或BGREWRITEAOF命令，并且哈希表的负载因子大于等于1.
3. Redis服务器目前在执行BGSAVE命令（rdb）或BGREWRITEAOF命令，并且哈希表的负载因子大于等于5.(负载因子=哈希表已保存节点数量 / 哈希表大小，当哈希表的负载因子小于0.1时，对哈希表执行收缩操作。)

### Redis并发竞争key的解决方案

1. 分布式锁+时间戳
2. 利用消息队列

### Redis与Mysql双写一致性方案

先更新数据库，再删缓存。数据库的读操作的速度远快于写操作的，所以脏数据很难出现。可以对异步延时删除策略，保证读请求完成以后，再进行删除操作。

### Redis的管道pipeline

对于单线程阻塞式的Redis，Pipeline可以满足批量的操作，把多个命令连续的发送给Redis Server，然后一一解析响应结果。Pipelining可以提高批量处理性能，提升的原因主要是TCP连接中减少了“交互往返”的时间。pipeline 底层是通过把所有的操作封装成流，redis有定义自己的出入输出流。在 sync() 方法执行操作，每次请求放在队列里面，解析响应包。

## Mysql

### 事务的基本要素

1. 原子性：事务是一个原子操作单元，其对数据的修改，要么全都执行，要么全都不执行
2. 一致性：事务开始前和结束后，数据库的完整性约束没有被破坏。
3. 隔离性：同一时间，只允许一个事务请求同一数据，不同的事务之间彼此没有任何干扰。
4. 持久性：事务完成后，事务对数据库的所有更新将被保存到数据库，不能回滚。

### Mysql的存储引擎

1. InnoDB存储引擎：InnoDB存储引擎支持事务，其设计目标主要面向在线事务处理（OLTP）的应用。其特点是行锁设计，支持外键，并支持非锁定锁，即默认读取操作不会产生锁。从Mysql5.5.8版本开始，InnoDB存储引擎是默认的存储引擎。
2. MyISAM存储引擎：MyISAM存储引擎不支持事务、表锁设计，支持全文索引，主要面向一些OLAP数据库应用。InnoDB的数据文件本身就是主索引文件，而MyISAM的主索引和数据是分开的。
3. NDB存储引擎：NDB存储引擎是一个集群存储引擎，其结构是share nothing的集群架构，能提供更高的可用性。NDB的特点是数据全部放在内存中（从MySQL 5.1版本开始，可以将非索引数据放在磁盘上），因此主键查找的速度极快，并且通过添加NDB数据存储节点可以线性地提高数据库性能，是高可用、高性能的集群系统。NDB存储引擎的连接操作是在MySQL数据库层完成的，而不是在存储引擎层完成的。这意味着，复杂的连接操作需要巨大的网络开销，因此查询速度很慢。如果解决了这个问题，NDB存储引擎的市场应该是非常巨大的。
4. Memory存储引擎：Memory存储引擎（之前称HEAP存储引擎）将表中的数据存放在内存中，如果数据库重启或发生崩溃，表中的数据都将消失。它非常适合用于存储临时数据的临时表，以及数据仓库中的纬度表。Memory存储引擎默认使用哈希索引，而不是我们熟悉的B+树索引。虽然Memory存储引擎速度非常快，但在使用上还是有一定的限制。比如，只支持表锁，并发性能较差，并且不支持TEXT和BLOB列类型。最重要的是，存储变长字段时是按照定常字段的方式进行的，因此会浪费内存。
5. Archive存储引擎：Archive存储引擎只支持INSERT和SELECT操作，从MySQL 5.1开始支持索引。Archive存储引擎使用zlib算法将数据行（row）进行压缩后存储，压缩比一般可达1∶10。正如其名字所示，Archive存储引擎非常适合存储归档数据，如日志信息。Archive存储引擎使用行锁来实现高并发的插入操作，但是其本身并不是事务安全的存储引擎，其设计目标主要是提供高速的插入和压缩功能。
6. Maria存储引擎：Maria存储引擎是新开发的引擎，设计目标主要是用来取代原有的MyISAM存储引擎，从而成为MySQL的默认存储引擎。它可以看做是MyISAM的后续版本。Maria存储引擎的特点是：支持缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事务和非事务安全的选项，以及更好的BLOB字符类型的处理性能。

### 事务的并发问题

1. 脏读：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
2. 不可重复读：事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。
3. 幻读：A事务读取了B事务已经提交的新增数据。注意和不可重复读的区别，这里是新增，不可重复读是更改（或删除）。select某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，此时就发生了幻读。

### MySQL事务隔离级别

| 事务隔离级别 | 脏读 | 不可重复读 | 幻读 |
|--------|----|-------|----|
| 读未提交   | 是  | 是     | 是  |
| 不可重复读  | 否  | 是     | 是  |
| 可重复读   | 否  | 否     | 是  |
| 串行化    | 否  | 否     | 否  |

### Mysql的逻辑结构

* 最上层的服务类似其他CS结构，比如连接处理，授权处理。
* 第二层是Mysql的服务层，包括SQL的解析分析优化，存储过程触发器视图等也在这一层实现。
* 最后一层是存储引擎的实现，类似于Java接口的实现，Mysql的执行器在执行SQL的时候只会关注API的调用，完全屏蔽了不同引擎实现间的差异。比如Select语句，先会判断当前用户是否拥有权限，其次到缓存（内存）查询是否有相应的结果集，如果没有再执行解析sql，检查SQL 语句语法是否正确，再优化生成执行计划，调用API执行。

### SQL执行顺序

SQL的执行顺序：from---where--group by---having---select---order by

### MVCC,redolog,undolog,binlog

* undoLog 也就是我们常说的回滚日志文件 主要用于事务中执行失败，进行回滚，以及MVCC中对于数据历史版本的查看。由引擎层的InnoDB引擎实现,是逻辑日志,记录数据修改被修改前的值,比如"把id='B' 修改为id = 'B2' ，那么undo日志就会用来存放id ='B'的记录”。当一条数据需要更新前,会先把修改前的记录存储在undolog中,如果这个修改出现异常,,则会使用undo日志来实现回滚操作,保证事务的一致性。当事务提交之后，undo log并不能立马被删除,而是会被放到待清理链表中,待判断没有事物用到该版本的信息时才可以清理相应undolog。它保存了事务发生之前的数据的一个版本，用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读。
* redoLog 是重做日志文件是记录数据修改之后的值，用于持久化到磁盘中。redo log包括两部分：一是内存中的日志缓冲(redo log buffer)，该部分日志是易失性的；二是磁盘上的重做日志文件(redo log file)，该部分日志是持久的。由引擎层的InnoDB引擎实现,是物理日志,记录的是物理数据页修改的信息,比如“某个数据页上内容发生了哪些改动”。当一条数据需要更新时,InnoDB会先将数据更新，然后记录redoLog 在内存中，然后找个时间将redoLog的操作执行到磁盘上的文件上。不管是否提交成功我都记录，你要是回滚了，那我连回滚的修改也记录。它确保了事务的持久性。每个InnoDB存储引擎至少有1个重做日志文件组（group），每个文件组下至少有2个重做日志文件，如默认的ib_logfile0和ib_logfile1。为了得到更高的可靠性，用户可以设置多个的镜像日志组（mirrored log groups），将不同的文件组放在不同的磁盘上，以此提高重做日志的高可用性。在日志组中每个重做日志文件的大小一致，并以循环写入的方式运行。InnoDB存储引擎先写重做日志文件1，当达到文件的最后时，会切换至重做日志文件2，再当重做日志文件2也被写满时，会再切换到重做日志文件1中。
* MVCC多版本并发控制是MySQL中基于乐观锁理论实现隔离级别的方式，用于读已提交和可重复读取隔离级别的实现。在MySQL中，会在表中每一条数据后面添加两个字段：最近修改该行数据的事务ID，指向该行（undolog表中）回滚段的指针。Read View判断行的可见性，创建一个新事务时，copy一份当前系统中的活跃事务列表。意思是，当前不应该被本事务看到的其他事务id列表。已提交读隔离级别下的事务在每次查询的开始都会生成一个独立的ReadView,而可重复读隔离级别则在第一次读的时候生成一个ReadView，之后的读都复用之前的ReadView。

### binlog和redolog的区别

1. redolog是在InnoDB存储引擎层产生，而binlog是MySQL数据库的上层服务层产生的。
2. 两种日志记录的内容形式不同。MySQL的binlog是逻辑日志，其记录是对应的SQL语句，对应的事务。而innodb存储引擎层面的重做日志是物理日志，是关于每个页（Page）的更改的物理情况。
3. 两种日志与记录写入磁盘的时间点不同，binlog日志只在事务提交完成后进行一次写入。而innodb存储引擎的重做日志在事务进行中不断地被写入，并日志不是随事务提交的顺序进行写入的。
4. binlog不是循环使用，在写满或者重启之后，会生成新的binlog文件，redolog是循环使用。
5. binlog可以作为恢复数据使用，主从复制搭建，redolog作为异常宕机或者介质故障后的数据恢复使用。

### Mysql读写分离以及主从同步

1. 原理：主库将变更写binlog日志，然后从库连接到主库后，从库有一个IO线程，将主库的binlog日志拷贝到自己本地，写入一个中继日志中，接着从库中有一个sql线程会从中继日志读取binlog，然后执行binlog日志中的内容，也就是在自己本地再执行一遍sql，这样就可以保证自己跟主库的数据一致。
2. 问题：这里有很重要一点，就是从库同步主库数据的过程是串行化的，也就是说主库上并行操作，在从库上会串行化执行，由于从库从主库拷贝日志以及串行化执行sql特点，在高并发情况下，从库数据一定比主库慢一点，是有延时的，所以经常出现，刚写入主库的数据可能读不到了，要过几十毫秒，甚至几百毫秒才能读取到。还有一个问题，如果突然主库宕机了，然后恰巧数据还没有同步到从库，那么有些数据可能在从库上是没有的，有些数据可能就丢失了。所以mysql实际上有两个机制，一个是半同步复制，用来解决主库数据丢失问题，一个是并行复制，用来解决主从同步延时问题。
3. 半同步复制：semi-sync复制，指的就是主库写入binlog日志后，就会将强制此时立即将数据同步到从库，从库将日志写入自己本地的relay log之后，接着会返回一个ack给主库，主库接收到至少一个从库ack之后才会认为写完成。
4. 并发复制：指的是从库开启多个线程，并行读取relay log中不同库的日志，然后并行重放不同库的日志，这样库级别的并行。（将主库分库也可缓解延迟问题）
                                                                                                                                                  
### Next-Key Lock

InnoDB 采用 Next-Key Lock 解决幻读问题。在`insert into test(xid) values (1), (3), (5), (8), (11);`后，由于xid上是有索引的，该算法总是会去锁住索引记录。现在，该索引可能被锁住的范围如下：(-∞, 1], (1, 3], (3, 5], (5, 8], (8, 11], (11, +∞)。Session A（`select * from test where id = 8 for update`）执行后会锁住的范围：(5, 8], (8, 11]。除了锁住8所在的范围，还会锁住下一个范围，所谓Next-Key。

### InnoDB的关键特性

1. 插入缓冲：对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入；若不在，则先放入到一个Insert Buffer对象中。然后再以一定的频率和情况进行Insert Buffer和辅助索引页子节点的merge（合并）操作，这时通常能将多个插入合并到一个操作中（因为在一个索引页中），这就大大提高了对于非聚集索引插入的性能。
2. 两次写：两次写带给InnoDB存储引擎的是数据页的可靠性，有经验的DBA也许会想，如果发生写失效，可以通过重做日志进行恢复。这是一个办法。但是必须清楚地认识到，如果这个页本身已经发生了损坏（物理到page页的物理日志成功页内逻辑日志失败），再对其进行重做是没有意义的。这就是说，在应用（apply）重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本来还原该页，再进行重做。在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是会通过memcpy函数将脏页先复制到内存中的doublewrite buffer，之后通过doublewrite buffer再分两次，每次1MB顺序地写入共享表空间的物理磁盘上，这就是doublewrite。
3. 自适应哈希索引：InnoDB存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引。
4. 异步IO：为了提高磁盘操作性能，当前的数据库系统都采用异步IO（AIO）的方式来处理磁盘操作。AIO的另一个优势是可以进行IO Merge操作，也就是将多个IO合并为1个IO，这样可以提高IOPS的性能。
5. 刷新邻接页：当刷新一个脏页时，InnoDB存储引擎会检测该页所在区（extent）的所有页，如果是脏页，那么一起进行刷新。这样做的好处显而易见，通过AIO可以将多个IO写入操作合并为一个IO操作，故该工作机制在传统机械磁盘下有着显著的优势。

### Mysql如何保证一致性和持久性

MySQL为了保证ACID中的一致性和持久性，使用了WAL(Write-Ahead Logging,先写日志再写磁盘)。Redo log就是一种WAL的应用。当数据库忽然掉电，再重新启动时，MySQL可以通过Redo log还原数据。也就是说，每次事务提交时，不用同步刷新磁盘数据文件，只需要同步刷新Redo log就足够了。

### InnoDB的行锁模式

* 共享锁(S)：用法lock in share mode，又称读锁，允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。
* 排他锁(X)：用法for update，又称写锁，允许获取排他锁的事务更新数据，阻止其他事务取得相同的数据集共享读锁和排他写锁。若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。在没有索引的情况下，InnoDB只能使用表锁。

### 为什么选择B+树作为索引结构

* Hash索引：Hash索引底层是哈希表，哈希表是一种以key-value存储数据的结构，所以多个数据在存储关系上是完全没有任何顺序关系的，所以，对于区间查询是无法直接通过索引查询的，就需要全表扫描。所以，哈希索引只适用于等值查询的场景。而B+ 树是一种多路平衡查询树，所以他的节点是天然有序的（左子节点小于父节点、父节点小于右子节点），所以对于范围查询的时候不需要做全表扫描
* 二叉查找树：解决了排序的基本问题，但是由于无法保证平衡，可能退化为链表。
* 平衡二叉树：通过旋转解决了平衡的问题，但是旋转操作效率太低。
* 红黑树：通过舍弃严格的平衡和引入红黑节点，解决了	AVL旋转效率过低的问题，但是在磁盘等场景下，树仍然太高，IO次数太多。
* B+树：在B树的基础上，将非叶节点改造为不存储数据纯索引节点，进一步降低了树的高度；此外将叶节点使用指针连接成链表，范围查询更加高效。

### B+树的叶子节点都可以存哪些东西

可能存储的是整行数据，也有可能是主键的值。B+树的叶子节点存储了整行数据的是主键索引，也被称之为聚簇索引。而索引B+ Tree的叶子节点存储了主键的值的是非主键索引，也被称之为非聚簇索引

### 覆盖索引

指一个查询语句的执行只用从索引中就能够取得，不必从数据表中读取。也可以称之为实现了索引覆盖。

### 查询在什么时候不走（预期中的）索引

1. 模糊查询 %like
2. 索引列参与计算,使用了函数
3. 非最左前缀顺序
4. where单列索引对null判断 
5. where不等于
6. or操作有至少一个字段没有索引
7. 需要回表的查询结果集过大（超过配置的范围）

### 为什么Mysql数据库存储不建议使用NULL

1. NOT IN子查询在有NULL值的情况下返回永远为空结果，查询容易出错。
2. 索引问题，单列索引无法存储NULL值，where对null判断会不走索引。
3. 如果在两个字段进行拼接（CONCAT函数），首先要各字段进行非null判断，否则只要任意一个字段为空都会造成拼接的结果为null
4. 如果有 Null column 存在的情况下，count(Null column)需要格外注意，null 值不会参与统计。
5. Null列需要更多的存储空间：需要一个额外的字节作为判断是否为NULL的标志位

### explain命令概要

1. id:select选择标识符
2. select_type:表示查询的类型。
3. table:输出结果集的表
4. partitions:匹配的分区
5. type:表示表的连接类型
6. possible_keys:表示查询时，可能使用的索引
7. key:表示实际使用的索引
8. key_len:索引字段的长度
9. ref:列与索引的比较
10. rows:扫描出的行数(估算的行数)
11. filtered:按表条件过滤的行百分比
12. Extra:执行情况的描述和说明

### explain 中的 select_type（查询的类型）

1. SIMPLE(简单SELECT，不使用UNION或子查询等)
2. PRIMARY(子查询中最外层查询，查询中若包含任何复杂的子部分，最外层的select被标记为PRIMARY)
3. UNION(UNION中的第二个或后面的SELECT语句)
4. DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)
5. UNION RESULT(UNION的结果，union语句中第二个select开始后面所有select)
6. SUBQUERY(子查询中的第一个SELECT，结果不依赖于外部查询)
7. DEPENDENT SUBQUERY(子查询中的第一个SELECT，依赖于外部查询)
8. DERIVED(派生表的SELECT, FROM子句的子查询)
9. UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)

### explain 中的 type（表的连接类型）

1. system：最快，主键或唯一索引查找常量值，只有一条记录，很少能出现
2. const：PK或者unique上的等值查询
3. eq_ref：PK或者unique上的join查询，等值匹配，对于前表的每一行(row)，后表只有一行命中
4. ref：非唯一索引，等值匹配，可能有多行命中
5. range：索引上的范围扫描，例如：between/in
6. index：索引上的全集扫描，例如：InnoDB的count
7. ALL：最慢，全表扫描(full table scan)

### explain 中的 Extra（执行情况的描述和说明）

1. Using where:不用读取表中所有信息，仅通过索引就可以获取所需数据，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示mysql服务器将在存储引擎检索行后再进行过滤
2. Using temporary：表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询，常见 group by ; order by
3. Using filesort：当Query中包含 order by 操作，而且无法利用索引完成的排序操作称为“文件排序”
4. Using join buffer：改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。
5. Impossible where：这个值强调了where语句会导致没有符合条件的行（通过收集统计信息不可能存在结果）。
6. Select tables optimized away：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行
7. No tables used：Query语句中使用from dual 或不含任何from子句

### 数据库优化指南

1. 创建并使用正确的索引
2. 只返回需要的字段
3. 减少交互次数（批量提交）
4. 设置合理的Fetch Size（数据每次返回给客户端的条数）


## 操作系统

### 进程和线程

1. 进程是操作系统资源分配的最小单位，线程是CPU任务调度的最小单位。一个进程可以包含多个线程，所以进程和线程都是一个时间段的描述，是CPU工作时间段的描述，不过是颗粒大小不同。
2. 不同进程间数据很难共享，同一进程下不同线程间数据很易共享。
3. 每个进程都有独立的代码和数据空间，进程要比线程消耗更多的计算机资源。线程可以看做轻量级的进程，同一类线程共享代码和数据空间，每个线程都有自己独立的运行栈和程序计数器，线程之间切换的开销小。
4. 进程间不会相互影响，一个线程挂掉将导致整个进程挂掉。
5. 系统在运行的时候会为每个进程分配不同的内存空间；而对线程而言，除了CPU外，系统不会为线程分配内存（线程所使用的资源来自其所属进程的资源），线程组之间只能共享资源。

### 多线程和单线程

线程不是越多越好，假如你的业务逻辑全部是计算型的（CPU密集型）,不涉及到IO，并且只有一个核心。那肯定一个线程最好，多一个线程就多一点线程切换的计算，CPU不能完完全全的把计算能力放在业务计算上面，线程越多就会造成CPU利用率（用在业务计算的时间/总的时间）下降。但是在WEB场景下，业务并不是CPU密集型任务，而是IO密集型的任务，一个线程是不合适，如果一个线程在等待数据时，把CPU的计算能力交给其他线程，这样也能充分的利用CPU资源。但是线程数量也要有个限度，一般线程数有一个公式：最佳启动线程数=[任务执行时间/(任务执行时间-IO等待时间)]*CPU内核数超过这个数量，CPU要进行多余的线程切换从而浪费计算能力，低于这个数量，CPU要进行IO等待从而造成计算能力不饱和。总之就是要尽可能的榨取CPU的计算能力。如果你的CPU处于饱和状态，并且没有多余的线程切换浪费，那么此时就是你服务的完美状态，如果再加大并发量，势必会造成性能上的下降。

### 进程的组成部分

进程由进程控制块（PCB）、程序段、数据段三部分组成。

### 进程的通信方式

1. 无名管道：半双工的，即数据只能在一个方向上流动，只能用于具有亲缘关系的进程之间的通信，可以看成是一种特殊的文件，对于它的读写也可以使用普通的read、write 等函数。但是它不是普通的文件，并不属于其他任何文件系统，并且只存在于内存中。
2. FIFO命名管道：FIFO是一种文件类型，可以在无关的进程之间交换数据，与无名管道不同，FIFO有路径名与之相关联，它以一种特殊设备文件形式存在于文件系统中。
3. 消息队列：消息队列，是消息的链接表，存放在内核中。一个消息队列由一个标识符（即队列ID）来标识。
4. 信号量：信号量是一个计数器，信号量用于实现进程间的互斥与同步，而不是用于存储进程间通信数据。
5. 共享内存：共享内存指两个或多个进程共享一个给定的存储区，一般配合信号量使用。

### 进程间五种通信方式的比较

1. 管道：速度慢，容量有限，只有父子进程能通讯。
2. FIFO：任何进程间都能通讯，但速度慢。
3. 消息队列：容量受到系统限制，且要注意第一次读的时候，要考虑上一次没有读完数据的问题。
4. 信号量：不能传递复杂消息，只能用来同步。
5. 共享内存区：能够很容易控制容量，速度快，但要保持同步，比如一个进程在写的时候，另一个进程要注意读写的问题，相当于线程中的线程安全，当然，共享内存区同样可以用作线程间通讯，不过没这个必要，线程间本来就已经共享了同一进程内的一块内存。

### 内存管理有哪几种方式

1. 块式管理：把主存分为一大块、一大块的，当所需的程序片断不在主存时就分配一块主存空间，把程序片断load入主存，就算所需的程序片度只有几个字节也只能把这一块分配给它。这样会造成很大的浪费，平均浪费了50％的内存空间，但是易于管理。
2. 页式管理：把主存分为一页一页的，每一页的空间要比一块一块的空间小很多，显然这种方法的空间利用率要比块式管理高很多。
3. 段式管理：把主存分为一段一段的，每一段的空间又要比一页一页的空间小很多，这种方法在空间利用率上又比页式管理高很多，但是也有另外一个缺点。一个程序片断可能会被分为几十段，这样很多时间就会被浪费在计算每一段的物理地址上。
4. 段页式管理：结合了段式管理和页式管理的优点。将程序分成若干段，每个段分成若干页。段页式管理每取一数据，要访问3次内存。

### 页面置换算法

1. 最佳置换算法OPT：只具有理论意义的算法，用来评价其他页面置换算法。置换策略是将当前页面中在未来最长时间内不会被访问的页置换出去。
2. 先进先出置换算法FIFO：简单粗暴的一种置换算法，没有考虑页面访问频率信息。每次淘汰最早调入的页面。
3. 最近最久未使用算法LRU：算法赋予每个页面一个访问字段，用来记录上次页面被访问到现在所经历的时间t，每次置换的时候把t值最大的页面置换出去(实现方面可以采用寄存器或者栈的方式实现)。
4. 时钟算法clock(也被称为是最近未使用算法NRU)：页面设置一个访问位，并将页面链接为一个环形队列，页面被访问的时候访问位设置为1。页面置换的时候，如果当前指针所指页面访问为为0，那么置换，否则将其置为0，循环直到遇到一个访问为位0的页面。
5. 改进型Clock算法：在Clock算法的基础上添加一个修改位，替换时根究访问位和修改位综合判断。优先替换访问位和修改位都是0的页面，其次是访问位为0修改位为1的页面。
6. LFU最少使用算法LFU：设置寄存器记录页面被访问次数，每次置换的时候置换当前访问次数最少的。

### 操作系统中进程调度策略有哪几种

1. 先来先服务调度算法FCFS：队列实现，非抢占，先请求CPU的进程先分配到CPU，可以作为作业调度算法也可以作为进程调度算法；按作业或者进程到达的先后顺序依次调度，对于长作业比较有利.
2. 最短作业优先调度算法SJF：作业调度算法，算法从就绪队列中选择估计时间最短的作业进行处理，直到得出结果或者无法继续执行，平均等待时间最短，但难以知道下一个CPU区间长度；缺点：不利于长作业；未考虑作业的重要性；运行时间是预估的，并不靠谱.
3. 优先级调度算法(可以是抢占的，也可以是非抢占的)：优先级越高越先分配到CPU，相同优先级先到先服务，存在的主要问题是：低优先级进程无穷等待CPU，会导致无穷阻塞或饥饿.
4. 时间片轮转调度算法(可抢占的)：按到达的先后对进程放入队列中，然后给队首进程分配CPU时间片，时间片用完之后计时器发出中断，暂停当前进程并将其放到队列尾部，循环 ;队列中没有进程被分配超过一个时间片的CPU时间，除非它是唯一可运行的进程。如果进程的CPU区间超过了一个时间片，那么该进程就被抢占并放回就绪队列。

### 死锁的4个必要条件
   
1. 互斥条件：一个资源每次只能被一个线程使用；
2. 请求与保持条件：一个线程因请求资源而阻塞时，对已获得的资源保持不放；
3. 不剥夺条件：进程已经获得的资源，在未使用完之前，不能强行剥夺；
4. 循环等待条件：若干线程之间形成一种头尾相接的循环等待资源关系。   

### 如何避免（预防）死锁

1. 破坏“请求和保持”条件：让进程在申请资源时，一次性申请所有需要用到的资源，不要一次一次来申请，当申请的资源有一些没空，那就让线程等待。不过这个方法比较浪费资源，进程可能经常处于饥饿状态。还有一种方法是，要求进程在申请资源前，要释放自己拥有的资源。
2. 破坏“不可抢占”条件：允许进程进行抢占，方法一：如果去抢资源，被拒绝，就释放自己的资源。方法二：操作系统允许抢，只要你优先级大，可以抢到。
3. 破坏“循环等待”条件：将系统中的所有资源统一编号，进程可在任何时刻提出资源申请，但所有申请必须按照资源的编号顺序提出（指定获取锁的顺序，顺序加锁）。

## 计算机网路

### Get和Post区别

1. Get是不安全的，因为在传输过程，数据被放在请求的URL中；Post的所有操作对用户来说都是不可见的。
2. Get传送的数据量较小，这主要是因为受URL长度限制；Post传送的数据量较大，一般被默认为不受限制。
3. Get限制Form表单的数据集的值必须为ASCII字符；而Post支持整个ISO10646字符集。
4. Get执行效率却比Post方法好。Get是form提交的默认方法。
5. GET产生一个TCP数据包；POST产生两个TCP数据包。（非必然，客户端可灵活决定）

### Http请求的完全过程

1. 浏览器根据域名解析IP地址（DNS）,并查DNS缓存
2. 浏览器与WEB服务器建立一个TCP连接
3. 浏览器给WEB服务器发送一个HTTP请求（GET/POST）：一个HTTP请求报文由请求行（request line）、请求头部（headers）、空行（blank line）和请求数据（request body）4个部分组成。
4. 服务端响应HTTP响应报文，报文由状态行（status line）、相应头部（headers）、空行（blank line）和响应数据（response body）4个部分组成。
5. 浏览器解析渲染

### 计算机网络的五层模型

1. 应用层：为操作系统或网络应用程序提供访问网络服务的接口 ，通过应用进程间的交互完成特定网络应用。应用层定义的是应用进程间通信和交互的规则。（HTTP，FTP，SMTP，RPC）
2. 传输层：负责向两个主机中进程之间的通信提供通用数据服务。（TCP,UDP）
3. 网络层：负责对数据包进行路由选择和存储转发。（IP，ICMP(ping命令)）
4. 数据链路层：两个相邻节点之间传送数据时，数据链路层将网络层交下来的IP数据报组装成帧，在两个相邻的链路上传送帧（frame)。每一帧包括数据和必要的控制信息。
5. 物理层：物理层所传数据单位是比特（bit)。物理层要考虑用多大的电压代表1 或 0 ，以及接受方如何识别发送方所发送的比特。

### tcp和udp区别

1. TCP面向连接，UDP是无连接的，即发送数据之前不需要建立连接。
2. TCP提供可靠的服务。也就是说，通过TCP连接传送的数据，无差错，不丢失，不重复，且按序到达;UDP尽最大努力交付，即不保证可靠交付。
3. TCP面向字节流，实际上是TCP把数据看成一连串无结构的字节流，UDP是面向报文的，UDP没有拥塞控制，因此网络出现拥塞不会使源主机的发送速率降低（对实时应用很有用，如IP电话，实时视频会议等）
4. 每一条TCP连接只能是点到点的，UDP支持一对一，一对多，多对一和多对多的交互通信。
5. TCP首部开销20字节，UDP的首部开销小，只有8个字节。
6. TCP的逻辑通信信道是全双工的可靠信道，UDP则是不可靠信道。

### tcp和udp的优点

* TCP的优点： 可靠，稳定 TCP的可靠体现在TCP在传递数据之前，会有三次握手来建立连接，而且在数据传递时，有确认、窗口、重传、拥塞控制机制，在数据传完后，还会断开连接用来节约系统资源。 TCP的缺点： 慢，效率低，占用系统资源高，易被攻击 TCP在传递数据之前，要先建连接，这会消耗时间，而且在数据传递时，确认机制、重传机制、拥塞控制机制等都会消耗大量的时间，而且要在每台设备上维护所有的传输连接，事实上，每个连接都会占用系统的CPU、内存等硬件资源。 而且，因为TCP有确认机制、三次握手机制，这些也导致TCP容易被人利用，实现DOS、DDOS、CC等攻击。
* UDP的优点： 快，比TCP稍安全 UDP没有TCP的握手、确认、窗口、重传、拥塞控制等机制，UDP是一个无状态的传输协议，所以它在传递数据时非常快。没有TCP的这些机制，UDP较TCP被攻击者利用的漏洞就要少一些。但UDP也是无法避免攻击的，比如：UDP Flood攻击…… UDP的缺点： 不可靠，不稳定 因为UDP没有TCP那些可靠的机制，在数据传递时，如果网络质量不好，就会很容易丢包。 基于上面的优缺点，那么： 什么时候应该使用TCP： 当对网络通讯质量有要求的时候，比如：整个数据要准确无误的传递给对方，这往往用于一些要求可靠的应用，比如HTTP、HTTPS、FTP等传输文件的协议，POP、SMTP等邮件传输的协议。 在日常生活中，常见使用TCP协议的应用如下： 浏览器，用的HTTP FlashFXP，用的FTP Outlook，用的POP、SMTP Putty，用的Telnet、SSH QQ文件传输。什么时候应该使用UDP： 当对网络通讯质量要求不高的时候，要求网络通讯速度能尽量的快，这时就可以使用UDP。 比如，日常生活中，常见使用UDP协议的应用如下： QQ语音 QQ视频 TFTP。

### 三次握手

* 第一次握手：建立连接时，客户端发送syn包（syn=x）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）。
* 第二次握手：服务器收到syn包，必须确认客户的SYN（ack=x+1），同时自己也发送一个SYN包（syn=y），即SYN+ACK包，此时服务器进入SYN_RECV状态；
* 第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=y+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手。

### 为什么不能两次握手

TCP是一个双向通信协议，通信双方都有能力发送信息，并接收响应。如果只是两次握手， 至多只有连接发起方的起始序列号能被确认， 另一方选择的序列号则得不到确认

### 四次挥手

1. 客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
2. 服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
3. 客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
4. 服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
5. 客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
6. 服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些

### 为什么连接的时候是三次握手，关闭的时候却是四次握手

因为当Server端收到Client端的SYN连接请求报文后，可以直接发送SYN+ACK报文。其中ACK报文是用来应答的，SYN报文是用来同步的。但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭SOCKET，所以只能先回复一个ACK报文，告诉Client端，"你发的FIN报文我收到了"。只有等到我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送。故需要四步握手。

## 数据结构与算法

### 排序算法

1. 冒泡排序
2. 选择排序：选择排序与冒泡排序有点像，只不过选择排序每次都是在确定了最小数的下标之后再进行交换，大大减少了交换的次数
3. 插入排序：将一个记录插入到已排序的有序表中，从而得到一个新的，记录数增1的有序表
4. 快速排序：通过一趟排序将序列分成左右两部分，其中左半部分的的值均比右半部分的值小，然后再分别对左右部分的记录进行排序，直到整个序列有序。
```
int partition(int a[], int low, int high){
    int key = a[low];
    while( low < high ){
        while(low < high && a[high] >= key) high--;
        a[low] = a[high];
        while(low < high && a[low] <= key) low++;
        a[high] = a[low];
    }
    a[low] = key;
    return low;
}
void quick_sort(int a[], int low, int high){
    if(low >= high) return;
    int keypos = partition(a, low, high);
    quick_sort(a, low, keypos-1);
    quick_sort(a, keypos+1, high);
}
```

5. 堆排序：将待排序序列构造成一个大顶堆，此时，整个序列的最大值就是堆顶的根节点。将其与末尾元素进行交换，此时末尾就为最大值。然后将剩余n-1个元素重新构造成一个堆，这样会得到n个元素的次小值。如此反复执行，便能得到一个有序序列了。
```
public class Test {

    public void sort(int[] arr) {
        for (int i = arr.length / 2 - 1; i >= 0; i--) {
            adjustHeap(arr, i, arr.length);
        }
        for (int j = arr.length - 1; j > 0; j--) {
            swap(arr, 0, j);
            adjustHeap(arr, 0, j);
        }

    }

    public void adjustHeap(int[] arr, int i, int length) {
        int temp = arr[i];
        for (int k = i * 2 + 1; k < length; k = k * 2 + 1) {
            if (k + 1 < length && arr[k] < arr[k + 1]) {
                k++;
            }
            if (arr[k] > temp) {
                arr[i] = arr[k];
                i = k;
            } else {
                break;
            }
        }
        arr[i] = temp;
    }

    public void swap(int[] arr, int a, int b) {
        int temp = arr[a];
        arr[a] = arr[b];
        arr[b] = temp;
    }

    public static void main(String[] args) {
        int[] arr = {9, 8, 7, 6, 5, 4, 3, 2, 1};
        new Test().sort(arr);
        System.out.println(Arrays.toString(arr));
    }
}
```
6. 希尔排序：先将整个待排记录序列分割成为若干子序列分别进行直接插入排序，待整个序列中的记录基本有序时再对全体记录进行一次直接插入排序。
7. 归并排序：把有序表划分成元素个数尽量相等的两半，把两半元素分别排序，两个有序表合并成一个

## 其他

### 高并发系统的设计与实现

在开发高并发系统时有三把利器用来保护系统：缓存、降级和限流。

* 缓存：缓存比较好理解，在大型高并发系统中，如果没有缓存数据库将分分钟被爆，系统也会瞬间瘫痪。使用缓存不单单能够提升系统访问速度、提高并发访问量，也是保护数据库、保护系统的有效方式。大型网站一般主要是“读”，缓存的使用很容易被想到。在大型“写”系统中，缓存也常常扮演者非常重要的角色。比如累积一些数据批量写入，内存里面的缓存队列（生产消费），以及HBase写数据的机制等等也都是通过缓存提升系统的吞吐量或者实现系统的保护措施。甚至消息中间件，你也可以认为是一种分布式的数据缓存。
* 降级：服务降级是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行。降级往往会指定不同的级别，面临不同的异常等级执行不同的处理。根据服务方式：可以拒接服务，可以延迟服务，也有时候可以随机服务。根据服务范围：可以砍掉某个功能，也可以砍掉某些模块。总之服务降级需要根据不同的业务需求采用不同的降级策略。主要的目的就是服务虽然有损但是总比没有好。
* 限流：限流可以认为服务降级的一种，限流就是限制系统的输入和输出流量已达到保护系统的目的。一般来说系统的吞吐量是可以被测算的，为了保证系统的稳定运行，一旦达到的需要限制的阈值，就需要限制流量并采取一些措施以完成限制流量的目的。比如：延迟处理，拒绝处理，或者部分拒绝处理等等。

### 负载均衡算法：

1. 轮询
2. 加权轮询
3. 随机算法
4. 一致性Hash

### 常见的限流算法：

常见的限流算法有计数器、漏桶和令牌桶算法。漏桶算法在分布式环境中消息中间件或者Redis都是可选的方案。发放令牌的频率增加可以提升整体数据处理的速度，而通过每次获取令牌的个数增加或者放慢令牌的发放速度和降低整体数据处理速度。而漏桶不行，因为它的流出速率是固定的，程序处理速度也是固定的。

### 秒杀并发情况下库存为负数问题

1. for update显示加锁
2. 把update语句写在前边，先把数量-1，之后select出库存如果>-1就commit,否则rollback。
```
update products set quantity = quantity-1 WHERE id=3;
select quantity from products WHERE id=3 for update;
```
3. update语句在更新的同时加上一个条件
```
quantity = select quantity from products WHERE id=3;
update products set quantity = ($quantity-1) WHERE id=3 and queantity = $quantity;
```
