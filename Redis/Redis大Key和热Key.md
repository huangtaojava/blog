## 前言

在互联网高并发场景中使用Redis，我们难免会遇到大Key和热Key这两个问题。今天这篇文章我们就来聊聊这两个问题及解决办法。

## 1.什么是大Key

我们知道Redis是key-value类型的内存数据存储结构，大key其实不是说的key大，而是key对应的value很大。
例如：

- key本身的数据量过大：一个String类型的value，它的值超过5MB(数据过大)；。
- key中的成员数过多：一个List类型的Key，它的列表数量超过20000个,一个ZSET类型的key，它的成员数量超过10,000个(列表数量过多)。
- key中成员的数据量过大：一个Hash类型的key，它的成员数量虽然只有1,000个但这些成员的值总大小超过100 MB(成员体积过大)。

## 2.什么是热Key

通常以其接收到的Key被请求频率来判定，例如：

- QPS集中在特定的Key：Redis实例的总QPS（每秒查询率）为10,000，而其中一个Key的每秒访问量达到了7,000。
- 带宽使用率集中在特定的Key：对一个拥有上千个成员且总大小为1 MB的HASH Key每秒发送大量的HGETALL操作请求。
- CPU使用时间占比集中在特定的Key：对一个拥有数万个成员的Key（ZSET类型）每秒发送大量的ZRANGE操作请求。

## 3.大Key会造成哪些问题

- 客户端执行命令的时长变慢。
- Redis内存达到maxmemory参数定义的上限引发操作阻塞或重要的Key被逐出，甚至引发内存溢出（Out Of Memory）。
- 集群架构下，某个数据分片的内存使用率远超其他数据分片，无法使数据分片的内存资源达到均衡。
- 对大Key执行读请求，会使Redis实例的带宽使用率被占满，导致自身服务变慢，同时易波及相关的服务。
- 对大Key执行删除操作，易造成主库较长时间的阻塞，进而可能引发同步中断或主从切换。

## 4.热Key会造成哪些问题

- 占用大量的CPU资源，影响其他请求并导致整体性能降低。
- 集群架构下，产生访问倾斜，即某个数据分片被大量访问，而其他数据分片处于空闲状态，可能引起该数据分片的连接数被耗尽，新的连接建立请求被拒绝等问题。
- 在抢购或秒杀场景下，可能因商品对应库存Key的请求量过大，超出Redis处理能力造成超卖。
- 热Key的请求压力数量超出Redis的承受能力易造成缓存击穿，即大量请求将被直接指向后端的存储层，导致存储访问量激增甚至宕机，从而影响其他业务。

## 5.什么原因会造成大Key、热Key

未正确使用Redis、业务规划不足、无效数据的堆积、访问量突增等都会产生大Key与热Key，如：

- 大key

    - 在不适用的场景下使用Redis，易造成Key的value过大，如使用String类型的Key存放大体积二进制文件型数据；
    - 业务上线前规划设计不足，没有对Key中的成员进行合理的拆分，造成个别Key中的成员数量过多；
    - 未定期清理无效数据，造成如HASH类型Key中的成员持续不断地增加；
    - 使用LIST类型Key的业务消费侧发生代码故障，造成对应Key的成员只增不减。

- 热key

    - 预期外的访问量陡增，如突然出现的爆款商品、访问量暴涨的热点新闻、直播间某主播搞活动带来的大量刷屏点赞、游戏中某区域发生多个工会之间的战斗涉及大量玩家等。

## 6.如何找出Redis中的大Key、热Key

#### redis-cli命令

使用Redis官方命令如下图：

```shell
[root@localhost redis]# redis-cli  --bigkeys -a redis123
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far '"lock"' with 1111 bytes

-------- summary -------

Sampled 1 keys in the keyspace!
Total key length in bytes is 4 (avg len 4.00)

Biggest string found '"lock"' has 1111 bytes

1 strings with 1111 bytes (100.00% of keys, avg size 1111.00)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

```shell
[root@localhost redis]# redis-cli  --hotkeys -a redis123
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.

# Scanning the entire keyspace to find hot keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

Error: ERR An LFU maxmemory policy is not selected, access frequency not tracked. Please note that when switching between policies at runtime LRU and LFU data will take some time to adjust.
```

很不幸，热key的查找需要将maxmemory-policy参数设置为LFU。

#### 使用redis-rdb-tools工具

[Redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools)是通过Python编写，支持定制化分析Redis
RDB快照文件的开源工具。您可以根据您的精细化需求，全面地分析Redis实例中所有Key的内存占用情况，同时也支持灵活地分析查询。

#### 通过业务层定位热Key

指向Redis的每一次访问都来自业务层，因此我们可以通过在业务层增加相应的代码对Redis的访问进行记录并异步汇总分析。该方案的优势为能够准确并及时的分析出热Key的存在，缺点为业务代码复杂度的增加，同时可能会降低一些性能。

#### 使用云厂商自带的Key分析功能

像国内知名的阿里云数据库智能服务系统CloudDBA就支持Redis大Key与热Key的实时分析、发现功能。

## 7.如何解决大Key和热Key问题

### 大Key的常见处理办法

#### 拆分

不是Key很大吗，我们可以将一个Key拆分成多个，每个单一的Key自然就小了。

#### 清理

将不适合Redis能力的数据存放至其它存储并将过期数据进行清理，删除时使用UNLINK命令，该命令能够以非阻塞的方式缓慢逐步的清理传入的Key，通过UNLINK，你可以安全的删除大Key甚至特大Key。

#### 监控

突然出现的大Key问题会让我们措手不及，因此，在大Key产生问题前发现它并进行处理是保持服务稳定的重要手段。我们可以通过监控系统并设置合理的Redis内存报警阈值来提醒我们此时可能有大Key正在产生，如：Redis内存使用率超过70%，Redis内存1小时内增长率超过20%等。  
通过此类监控手段我们可以在问题发生前解决问题，如：List的消费程序故障造成对应Key的列表数量持续增长，将告警转变为预警从而避免故障的发生。

### 热Key的常见处理办法

#### 复制

所谓的热Key不就是某个Key请求频率较高吗，我们可以复制多个一样的Key，均摊请求，当然这里又得保证多个Key值数据的一致性，如果业务对一致性要求较高，这种解决方式并不太好。

#### 读写分离

如果热Key的产生来自于读请求，那么读写分离是一个很好的解决方案。在使用读写分离架构时可以通过不断的增加从节点来降低每个Redis实例中的读请求压力。

#### 本地缓存

使用本地缓存，降低Redis缓存的请求频率，不过这会牵扯到多级缓存的一致性问题，需要业务上能够容忍不一致的情况。

本人掘金文章链接 ：[Redis大Key和热Key](https://juejin.cn/post/7325261939432243236)
