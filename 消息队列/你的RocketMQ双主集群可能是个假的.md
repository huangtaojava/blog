# 问题现象

RocketMQ部署的双主双从集群，Topic_Test1只存在于其中一台主节点broker-a上，从而该Topic发送的消息只会发送到了broker-a节点，另一台主节点broker-b等于是“隔岸观火”没有发挥任何作用。
<img alt="你的RocketMQ双主集群可能是个假的-1.png" height="390" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-1.png" width="1512"/>

<img alt="你的RocketMQ双主集群可能是个假的-2.png" height="472" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-2.png" width="1512"/>

# 问题分析

我们在发送Topic_Test1主题消息时，MQ集群里并存在Topic_Test1这个主题，Broker参数autoCreateTopicEnable默认为true，会自动创建不存在的Topic。

这里补充一下RocketMQ路由获取规则的知识：

1. Broker 在启动时会向 Nameserver 注册其存储的路由信息，并持续每30秒发送心跳包，同时更新路由信息。
3. Nameserver 则每10秒扫描一次路由表。一旦发现 Broker 服务出现故障，对应的路由信息将被移除。
5. 消息生产者每30秒会从 Nameserver 重新获取 Topic 的路由信息，并更新其本地路由表。在发送消息之前，如果本地路由表中缺少特定主题的路由信息，生产者会主动从
   Nameserver 拉取该主题的消息。
   ![你的RocketMQ双主集群可能是个假的-3.png](../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-3.png)

   现在生产者在发送消息的时候，肯定是获取不到这个未创建Topic的路由信息的，那么它会如何处理呢？

# 源码

没有什么是比源码更能说明问题的，我们直接看源码吧。 RocketMQ版本：5.1.0

<img alt="你的RocketMQ双主集群可能是个假的-4.png" height="424" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-4.png" width="815"/>

既然是双主集群，那么两台主Broker应该都会创建这个默认主题的，我们继续看生产者是如何发送这个Broker上不存在的Topic信息发送出去的吧。

<img alt="你的RocketMQ双主集群可能是个假的-5.png" height="377" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-5.png" width="991"/>

这里只有一个默认的Topic信息，后面会自动创建Topic_Test1主题的路由信息

<img alt="你的RocketMQ双主集群可能是个假的-6.png" height="420" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-6.png" width="1030"/>

默认的主题在两台主节点都存在，这里选择其中一个队列

<img alt="你的RocketMQ双主集群可能是个假的-7.png" height="687" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-7.png" width="1143"/>

我们继续看服务端是如何处理这个消息的

<img alt="你的RocketMQ双主集群可能是个假的-8.png" height="876" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-8.png" width="1157"/>

服务端发现主题不存在后，会自动创建，等下次往Nameserver发送路由心跳的时候，就会带上这个自动创建的主题信息。而当前因为发送的消息量比较少，另一台主节点此时还是没有Topic_Test1这个主题的，后续发送方从Nameserver处获取到的Topic_Test1主题的路由信息只有其中一个主节点队列，导致无论再发多少条消息，都只会发往其中一台主节点。

到这里，问题已经很清晰了，我们再做个实验验证一下，一次发送10条消息（超过8条）。

<img alt="你的RocketMQ双主集群可能是个假的-9.png" height="563" src="../image/%E4%BD%A0%E7%9A%84RocketMQ%E5%8F%8C%E4%B8%BB%E9%9B%86%E7%BE%A4%E5%8F%AF%E8%83%BD%E6%98%AF%E4%B8%AA%E5%81%87%E7%9A%84-9.png" width="1512"/>

# 总结：

生产环境中，一定要关闭自动创建主题（autoCreateTopicEnable=false），当业务需要新建主题时，选择手动创建,如果不关闭，在第一次发送消息量比较少时，则可能出现某个主节点无该主题。

本人掘金文章链接 ：[我的定时任务怎么消失了？](https://juejin.cn/post/7324869701000331273)