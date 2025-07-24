## 前言

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel
是面向分布式、多语言异构化服务架构的流量治理组件，主要以流量为切入点，从流量路由、流量控制、流量整形、熔断降级、系统自适应过载保护、热点流量防护等多个维度来帮助开发者保障微服务的稳定性。

今天这篇文章仅做Sentinel限流部分的核心源码解析，关于Sentinel的其他相关知识大家可以移步[官网](https://sentinelguard.io/zh-cn/index.html)
进行了解。

## 1.源码解析

```java
public class DemoApplication {
    public static void main(String[] args) {
        //限流规则，一秒两个请求
        FlowRule flowRule = new FlowRule();
        flowRule.setResource("abc");
        flowRule.setCount(2);
        flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        flowRule.setLimitApp("default");
        FlowRuleManager.loadRules(Collections.singletonList(flowRule));
        for (int i = 0; i < 3; i++) {
            try (Entry entry = SphU.entry("abc")) {
                // 被保护的逻辑
                System.out.println("tt");
            } catch (BlockException ex) {
                // 处理被流控的逻辑
                System.out.println("blocked!");
            }
        }
    }
}
```

```java
tt
tt
blocked!
```

上面是Sentinel进行限流最简单的演示demo,我们一步一步点进源码进行分析。

```java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    //获取上下文，通过ThreadLocal进行存储
    Context context = ContextUtil.getContext();
    if (context instanceof NullContext) {
        // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
        // so here init the entry only. No rule checking will be done.
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) {
        // Using default context.
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    // Global switch is close, no rule checking will do.
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }
    //这里说明一下，在Sentinel中使用了责任链设计模式，流量控制、日志、权限等都是责任链中的一个个slot
    //初始化责任链，在1.8版本中加了sip注解，可以更方便的添加自定义slot
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

    /*
     * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
     * so no rule checking will be done.
     */
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    Entry e = new CtEntry(resourceWrapper, chain, context, count, args);
    try {
        //开始执行责任链
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // This should not happen, unless there are errors existing in Sentinel internal.
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```

获取整个责任链

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
    ProcessorSlotChain chain = chainMap.get(resourceWrapper);
    if (chain == null) {
        synchronized (LOCK) {
            chain = chainMap.get(resourceWrapper);
            if (chain == null) {
                // Entry size limit.
                if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                    return null;
                }

                chain = SlotChainProvider.newSlotChain();
                Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                    chainMap.size() + 1);
                newMap.putAll(chainMap);
                newMap.put(resourceWrapper, chain);
                chainMap = newMap;
            }
        }
    }
    return chain;
}
public static ProcessorSlotChain newSlotChain() {
    if (slotChainBuilder != null) {
        return slotChainBuilder.build();
    }

    // Resolve the slot chain builder SPI.
    slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();

    if (slotChainBuilder == null) {
        // Should not go through here.
        RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
        slotChainBuilder = new DefaultSlotChainBuilder();
    } else {
        RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: {}",
            slotChainBuilder.getClass().getCanonicalName());
    }
    return slotChainBuilder.build();
}
```

<img alt="Sentinel限流源码解析(一)-1.png" height="550" src="../image/Sentinel%E9%99%90%E6%B5%81%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%28%E4%B8%80%29-1.png" width="1100"/>

默认情况下，会获取到上面这些slot

- `NodeSelectorSlot`负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
- `ClusterBuilderSlot`则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；
- `LogSlot`则用于记录日志信息；
- `StatisticSlot`则用于记录、统计不同纬度的 runtime 指标监控信息；
- `FlowSlot`则用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；
- `AuthoritySlot`则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
- `DegradeSlot`则通过统计信息以及预设的规则，来做熔断降级；
- `SystemSlot`则通过系统的状态，例如 load1 等，来控制总的入口流量；

<img alt="Sentinel限流源码解析(一)-2.png" height="600" src="../image/Sentinel%E9%99%90%E6%B5%81%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%28%E4%B8%80%29-2.png" width="1100"/>

```java
public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {
    private AbstractLinkedProcessorSlot<?> next = null;
    @Override
    public void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
        throws Throwable {
        if (next != null) {
            next.transformEntry(context, resourceWrapper, obj, count, prioritized, args);
        }
    }
    @SuppressWarnings("unchecked")
    void transformEntry(Context context, ResourceWrapper resourceWrapper, Object o, int count, boolean prioritized, Object... args)
        throws Throwable {
        T t = (T)o;
        entry(context, resourceWrapper, t, count, prioritized, args);
    }
    @Override
    public void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        if (next != null) {
            next.exit(context, resourceWrapper, count, args);
        }
    }
```

### FlowSlot

上面提到的Slot我们不一个一个的看了，直接定位到限流的FlowSlot。

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    checkFlow(resourceWrapper, context, node, count, prioritized);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
void checkFlow(ResourceWrapper resource, Context context, DefaultNode node, int count, boolean prioritized)
    throws BlockException {
    checker.checkFlow(ruleProvider, resource, context, node, count, prioritized);
}
```

```java
public class FlowRuleChecker {
    public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource,
                          Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {
        if (ruleProvider == null || resource == null) {
            return;
        }
        Collection<FlowRule> rules = ruleProvider.apply(resource.getName());
        if (rules != null) {
            for (FlowRule rule : rules) {
                if (!canPassCheck(rule, context, node, count, prioritized)) {
                    throw new FlowException(rule.getLimitApp(), rule);
                }
            }
        }
    }
    public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node,
                                                    int acquireCount) {
        return canPassCheck(rule, context, node, acquireCount, false);
    }
    public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                    boolean prioritized) {
        String limitApp = rule.getLimitApp();
        if (limitApp == null) {
            return true;
        }
        if (rule.isClusterMode()) {
            //集群模式限流
            return passClusterCheck(rule, context, node, acquireCount, prioritized);
        }
        //单机模式限流
        return passLocalCheck(rule, context, node, acquireCount, prioritized);
    }
    private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                          boolean prioritized) {
        Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
        if (selectedNode == null) {
            return true;
        }
        //获取到对应的TrafficShapingController,执行canPass方法
        return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
    }
```

<img alt="Sentinel限流源码解析(一)-3.png" height="300" src="../image/Sentinel%E9%99%90%E6%B5%81%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%28%E4%B8%80%29-3.png" width="500"/>

TrafficShapingController有四个实现类

- DefaultController:默认的处理器，快速失败
- ThrottlingController:漏斗算法，匀速排队
- WarmUpController:预热冷启动
- WarmUpRateLimiterController:预热+匀速排队

### DefaultController

```java
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    // 计算当前的qps或者线程数
    int curCount = avgUsedTokens(node);
    // 如果当前请求通过超出限制
    if (curCount + acquireCount > count) {
        // 如果当前为高优先级业务，且指标为QPS
        if (prioritized && grade == RuleConstant.FLOW_GRADE_QPS) {
            long currentTime;
            long waitInMs;
            currentTime = TimeUtil.currentTimeMillis();
            // 尝试借用未来时间窗口，获取一个等待时间
            waitInMs = node.tryOccupyNext(currentTime, acquireCount, count);
            if (waitInMs < OccupyTimeoutProperty.getOccupyTimeout()) {
                node.addWaitingRequest(currentTime + waitInMs, acquireCount);
                node.addOccupiedPass(acquireCount);
                //睡眠
                sleep(waitInMs);

                // PriorityWaitException indicates that the request will pass after waiting for {@link @waitInMs}.
                //这里虽然抛异常，但是请求会通过
                throw new PriorityWaitException(waitInMs);
            }
        }
        //拒绝
        return false;
    }
    return true;
}
private int avgUsedTokens(Node node) {
    if (node == null) {
        return DEFAULT_AVG_USED_TOKENS;
    }
    return grade == RuleConstant.FLOW_GRADE_THREAD ? node.curThreadNum() : (int)(node.passQps());
}
```

### ThrottlingController

```java
private static final long MS_TO_NS_OFFSET = TimeUnit.MILLISECONDS.toNanos(1);
//排队最大等待时间500ms，默认值
private final int maxQueueingTimeMs;
private final int statDurationMs;
//qps限制
private final double count;
//是否使用纳秒，当qps超过1000时，会开启
private final boolean useNanoSeconds;
//上次请求通过时间
private final AtomicLong latestPassedTime = new AtomicLong(-1);

private boolean checkPassUsingCachedMs(int acquireCount, double maxCountPerStat) {
    long currentTime = TimeUtil.currentTimeMillis();
    // Calculate the interval between every two requests.
    //计算两次请求间隔
    long costTime = Math.round(1.0d * statDurationMs * acquireCount / maxCountPerStat);

    // Expected pass time of this request.
    long expectedTime = costTime + latestPassedTime.get();

    if (expectedTime <= currentTime) {
        // Contention may exist here, but it's okay.
        //可以通过
        latestPassedTime.set(currentTime);
        return true;
    } else {
        // Calculate the time to wait.
        //不可以通过，需要等待
        long waitTime = costTime + latestPassedTime.get() - TimeUtil.currentTimeMillis();
        //等待的时间超过最大值，返回false
        if (waitTime > maxQueueingTimeMs) {
            return false;
        }
        //上次请求通过时间+本次请求间隔时间
        long oldTime = latestPassedTime.addAndGet(costTime);
        //算出睡眠时间
        waitTime = oldTime - TimeUtil.currentTimeMillis();
        if (waitTime > maxQueueingTimeMs) {
            //睡眠的时间超过最大值，返回false
            latestPassedTime.addAndGet(-costTime);
            return false;
        }
        // in race condition waitTime may <= 0
        if (waitTime > 0) {
            //睡眠等待
            sleepMs(waitTime);
        }
        return true;
    }
}
```

### WarmUpController

<img alt="Sentinel限流源码解析(一)-4.png" height="600" src="../image/Sentinel%E9%99%90%E6%B5%81%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%28%E4%B8%80%29-4.png" width="800"/>

```java
//qps限制
protected double count;
private int coldFactor;
//转折点令牌数
protected int warningToken = 0;
//最大令牌数
private int maxToken;
//斜率
protected double slope;
//累计的令牌数量
protected AtomicLong storedTokens = new AtomicLong(0);
//令牌的最后更新时间
protected AtomicLong lastFilledTime = new AtomicLong(0);
@Override
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    //当前的qps
    long passQps = (long) node.passQps();
    //上一个时间窗口的qps
    long previousQps = (long) node.previousPassQps();
    //同步令牌
    syncToken(previousQps);
    // 开始计算它的斜率
    // 如果进入了警戒线，开始调整他的qps
    long restToken = storedTokens.get();
    if (restToken >= warningToken) {//令牌超过转折点
        long aboveToken = restToken - warningToken;
        // 消耗的速度要比warning快，但是要比慢
        // current interval = restToken*slope+1/count
        //计算当前情况下能达到的最大qps
        double warningQps = Math.nextUp(1.0 / (aboveToken * slope + 1.0 / count));
        if (passQps + acquireCount <= warningQps) {
            return true;
        }
    } else {
        if (passQps + acquireCount <= count) {
            //当前qps+请求量不大于qps限制，允许通过
            return true;
        }
    }
    return false;
}
protected void syncToken(long passQps) {
    long currentTime = TimeUtil.currentTimeMillis();
    currentTime = currentTime - currentTime % 1000;
    long oldLastFillTime = lastFilledTime.get();
    if (currentTime <= oldLastFillTime) {
        //如果当前的时间窗口小于等于上次请求通过的时间窗口，不需要同步
        return;
    }
    long oldValue = storedTokens.get();
    long newValue = coolDownTokens(currentTime, passQps);
    //CAS操作，累计的令牌数量设置为新的令牌数量
    if (storedTokens.compareAndSet(oldValue, newValue)) {
        //令牌数量-上次的qps
        long currentValue = storedTokens.addAndGet(0 - passQps);
        if (currentValue < 0) {
            storedTokens.set(0L);
        }
        lastFilledTime.set(currentTime);
    }

}
private long coolDownTokens(long currentTime, long passQps) {
    long oldValue = storedTokens.get();
    long newValue = oldValue;

    // 添加令牌的判断前提条件:
    // 当令牌的消耗程度远远低于警戒线的时候
    if (oldValue < warningToken) {
        newValue = (long)(oldValue + (currentTime - lastFilledTime.get()) * count / 1000);
    } else if (oldValue > warningToken) {
        if (passQps < (int)count / coldFactor) {
            newValue = (long)(oldValue + (currentTime - lastFilledTime.get()) * count / 1000);
        }
    }
    return Math.min(newValue, maxToken);
}
```

### WarmUpRateLimiterController

```java
@Override
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    long previousQps = (long) node.previousPassQps();
    syncToken(previousQps);

    long currentTime = TimeUtil.currentTimeMillis();

    long restToken = storedTokens.get();
    long costTime = 0;
    long expectedTime = 0;
    //这里计算出了预热时间，其它部分代码和上面的ThrottlingController一样
    if (restToken >= warningToken) {
        long aboveToken = restToken - warningToken;

        // current interval = restToken*slope+1/count
        double warmingQps = Math.nextUp(1.0 / (aboveToken * slope + 1.0 / count));
        costTime = Math.round(1.0 * (acquireCount) / warmingQps * 1000);
    } else {
        costTime = Math.round(1.0 * (acquireCount) / count * 1000);
    }
    expectedTime = costTime + latestPassedTime.get();

    if (expectedTime <= currentTime) {
        latestPassedTime.set(currentTime);
        return true;
    } else {
        long waitTime = costTime + latestPassedTime.get() - currentTime;
        if (waitTime > timeoutInMs) {
            return false;
        } else {
            long oldTime = latestPassedTime.addAndGet(costTime);
            try {
                waitTime = oldTime - TimeUtil.currentTimeMillis();
                if (waitTime > timeoutInMs) {
                    latestPassedTime.addAndGet(-costTime);
                    return false;
                }
                if (waitTime > 0) {
                    Thread.sleep(waitTime);
                }
                return true;
            } catch (InterruptedException e) {
            }
        }
    }
    return false;
}
```

## 2.总结

以上就是Sentinel限流源码核心部分的解析，关于Sentinel通过滑动窗口统计分、秒两个维度相关qps和并发部分的源码解析将会在下篇文章进行解析，感兴趣的可以关注我，谢谢大家。

本人掘金文章链接 ：[Sentinel限流源码解析(一)](https://juejin.cn/post/7337518755918741539)
