# 基本配置：

**<font style="color:rgb(24, 24, 24);">-Xmx</font>**

<font style="color:rgb(24, 24, 24);">设置最大堆大小，</font>**<font style="color:rgb(24, 24, 24);">-Xmx3550m</font>
**<font style="color:rgb(24, 24, 24);">，设置JVM最大可用内存为3550 MB。</font>

**<font style="color:rgb(24, 24, 24);">-Xms</font>**

<font style="color:rgb(24, 24, 24);">设置JVM初始内存，设置JVM初始内存。 -Xms3550m，设置JVM初始内存为3550
MB。此值建议与-Xmx相同，避免每次垃圾回收完成后JVM重新分配内存。</font>

**<font style="color:rgb(24, 24, 24);">-Xmn</font>**

<font style="color:rgb(24, 24, 24);">设置年轻代大小，</font>**<font style="color:rgb(24, 24, 24);">-Xmn2g</font>
**<font style="color:rgb(24, 24, 24);">，设置年轻代大小为2 GB。整个JVM内存大小=年轻代大小+年老代大小+持久代大小。持久代一般固定大小为64
MB，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。</font>

**<font style="color:rgb(24, 24, 24);">-Xss</font>**

<font style="color:rgb(24, 24, 24);">设置线程的栈大小，</font>**<font style="color:rgb(24, 24, 24);">-Xss128k</font>
**<font style="color:rgb(24, 24, 24);">，设置每个线程的栈大小为128 KB。</font>

**<font style="color:rgb(24, 24, 24);">-XX:NewRatio=n</font>**

<font style="color:rgb(24, 24, 24);">设置年轻代和年老代的比值，</font>**<font style="color:rgb(24, 24, 24);">-XX:
NewRatio=4</font>**<font style="color:rgb(24, 24, 24);">
，设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。如果设置为4，那么年轻代与年老代所占比值为1:
4，年轻代占整个堆栈的1/5。</font>

**<font style="color:rgb(24, 24, 24);">-XX:SurvivorRatio=n</font>**

<font style="color:rgb(24, 24, 24);">年轻代中Eden区与两个Survivor区的比值。</font>**<font style="color:rgb(24, 24, 24);">
-XX:SurvivorRatio=4</font>**<font style="color:rgb(24, 24, 24);">
，设置年轻代中Eden区与Survivor区的大小比值。如果设置为4，那么两个Survivor区与一个Eden区的比值为2:
4，一个Survivor区占整个年轻代的1/6。</font>

**<font style="color:rgb(24, 24, 24);">-XX:MaxPermSize=n</font>**

<font style="color:rgb(24, 24, 24);">设置持久代大小。设置持久代大小。 -XX:MaxPermSize=16m，设置持久代大小为16
MB。</font><font style="color:#DF2A3F;">jvav8已经移除了</font>

**<font style="color:rgb(24, 24, 24);">-XX:MaxTenuringThreshold=n</font>**

<font style="color:rgb(24, 24, 24);">设置垃圾最大年龄，设置垃圾最大年龄。 -XX:
MaxTenuringThreshold=0，设置垃圾最大年龄。如果设置为0，那么年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，提高了效率。如果将此值设置为较大值，那么年轻代对象会在Survivor区进行多次复制，增加了对象在年轻代的存活时间，增加在年轻代即被回收的概率。</font>

**<font style="color:rgb(24, 24, 24);">-XX:+PrintGC</font>**

<font style="color:rgb(24, 24, 24);">用于输出GC日志</font>

**<font style="color:rgb(24, 24, 24);">-XX:+PrintGCDetails</font>**

<font style="color:rgb(24, 24, 24);">用于输出GC日志，更详细</font>

**<font style="color:rgb(24, 24, 24);">-XX:+PrintGCTimeStamps</font>**

<font style="color:rgb(24, 24, 24);">用于输出GC时间戳（JVM启动到当前日期的总时长的时间戳形式）。示例如下：</font>

<font style="color:rgb(24, 24, 24);">
0.855: [GC (Allocation Failure) [PSYoungGen: 33280K->5118K(38400K)] 33280K->5663K(125952K), 0.0067629 secs] [Times: user=0.01 sys=0.01, real=0.00 secs]  
</font>**<font style="color:rgb(24, 24, 24);">-XX:+PrintGCDateStamps  
</font>**<font style="color:rgb(24, 24, 24);">-XX:+PrintGCDateStamps 用于输出GC时间戳（日期形式）。示例如下：</font>

<font style="color:rgb(24, 24, 24);">2022-01-27T16:22:20.885+0800:
0.299: [GC pause (G1 Evacuation Pause) (young), 0.0036685 secs]</font>

**<font style="color:rgb(24, 24, 24);">-XX:+PrintHeapAtGC</font>**

<font style="color:rgb(24, 24, 24);">在进行GC前后打印出堆的信息。</font>

**-XX:+TraceClassLoading**  
打印出类加载信息，在排查是否是频繁类加载导致元空间不足从而导致full gc时需要用到的打印参数

**<font style="color:rgb(24, 24, 24);">-Xloggc:../logs/gc.log</font>**

<font style="color:rgb(24, 24, 24);">日志文件的输出路径。</font>

# CMS：

<font style="color:#DF2A3F;">堆内存在2-4g，且对停顿时间比较敏感的业务建议使用cms</font>

**<font style="color:rgb(25, 27, 31);">-XX:UseConcMarkSweepGC</font>**

<font style="color:rgb(25, 27, 31);">打开CMS GC收集器。JVM在1.8之前默认使用的是Parallel GC，9以后使用G1 GC。</font>

**<font style="color:rgb(25, 27, 31);">-XX:UseParNewGC</font>**

<font style="color:rgb(25, 27, 31);">
当使用CMS收集器时，默认年轻代使用多线程并行执行垃圾回收（UseConcMarkSweepGC开启后则默认开启）。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+CMSParallelRemarkEnabled</font>**

<font style="color:rgb(25, 27, 31);">采用并行标记方式降低停顿（默认开启）。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+CMSConcurrentMTEnabled</font>**

<font style="color:rgb(25, 27, 31);">
被启用时，并发的CMS阶段将以多线程执行（因此，多个GC线程会与所有的应用程序线程并行工作）。（默认开启）</font>

**<font style="color:rgb(25, 27, 31);">-XX:ConcGCThreads</font>**

<font style="color:rgb(25, 27, 31);">定义并发CMS过程运行时的线程数。</font>

**<font style="color:rgb(25, 27, 31);">-XX:ParallelGCThreads</font>**

<font style="color:rgb(25, 27, 31);">定义CMS过程并行收集的线程数。</font>

**<font style="color:rgb(25, 27, 31);">-XX:CMSInitiatingOccupancyFraction</font>**

<font style="color:rgb(25, 27, 31);">
该值代表老年代堆空间的使用率，默认值为92。当老年代使用率达到此值之后，并行收集器便开始进行垃圾收集，该参数需要配合UseCMSInitiatingOccupancyOnly一起使用，单独设置无效。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+UseCMSInitiatingOccupancyOnly</font>**

<font style="color:rgb(25, 27, 31);">该参数启用后，参数CMSInitiatingOccupancyFraction才会生效。默认关闭。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+CMSClassUnloadingEnabled</font>**

<font style="color:rgb(25, 27, 31);">
相对于并行收集器，CMS收集器默认不会对永久代进行垃圾回收。如果希望对永久代进行垃圾回收，可用设置-XX:
+CMSClassUnloadingEnabled。默认关闭。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+CMSIncrementalMode</font>**

<font style="color:rgb(25, 27, 31);">
开启CMS收集器的增量模式。增量模式使得回收过程更长，但是暂停时间往往更短。默认关闭。</font>

**<font style="color:rgb(25, 27, 31);">-XX:CMSFullGCsBeforeCompaction</font>**

<font style="color:rgb(25, 27, 31);">设置在执行多少次Full GC后对内存空间进行压缩整理，默认值0。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+CMSScavengeBeforeRemark</font>**

<font style="color:rgb(25, 27, 31);">在cms gc remark之前做一次ygc，减少gc
roots扫描的对象数，从而提高remark的效率，默认关闭。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+ExplicitGCInvokesConcurrent</font>**

<font style="color:rgb(25, 27, 31);">该参数启用后JVM无论什么时候调用系统GC，都执行CMS GC，而不是Full GC。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses</font>**

<font style="color:rgb(25, 27, 31);">该参数保证当有系统GC调用时，永久代也被包括进CMS垃圾回收的范围内。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+DisableExplicitGC</font>**

<font style="color:rgb(25, 27, 31);">该参数将使JVM完全忽略系统的GC调用（不管使用的收集器是什么类型）。</font>

**<font style="color:rgb(25, 27, 31);">-XX:+UseCompressedOops</font>**

<font style="color:rgb(25, 27, 31);">这个参数用于对类对象数据进行压缩处理，提高内存利用率。（默认开启）</font>

**<font style="color:rgb(25, 27, 31);">-XX:MaxGCPauseMillis=200</font>**

<font style="color:rgb(25, 27, 31);">这个参数用于设置GC暂停等待时间，单位为毫秒，不要设置过低。</font>

**<font style="color:rgb(25, 27, 31);">CMS的线程数计算公式</font>**

<font style="color:rgb(25, 27, 31);">区分young区的parnew gc线程数和old区的cms线程数，分别为以下两参数：</font>

+ <font style="color:rgb(25, 27, 31);">-XX:ParallelGCThreads=m // STW暂停时使用的GC线程数，一般用满CPU</font>
+ <font style="color:rgb(25, 27, 31);">-XX:ConcGCThreads=n // GC线程和业务线程并发执行时使用的GC线程数，一般较小</font>

**<font style="color:rgb(25, 27, 31);">ParallelGCThreads</font>**

<font style="color:rgb(25, 27, 31);">其中ParallelGCThreads 参数的默认值是：</font>

+ <font style="color:rgb(25, 27, 31);">CPU核心数 <= 8，则为 ParallelGCThreads=CPU核心数，比如我的那个旧电脑是4</font>
+ <font style="color:rgb(25, 27, 31);">CPU核心数 > 8，则为 ParallelGCThreads = CPU核心数 * 5/8 + 3 向下取整</font>
+ <font style="color:rgb(25, 27, 31);">16核的情况下，ParallelGCThreads = 13</font>
+ <font style="color:rgb(25, 27, 31);">32核的情况下，ParallelGCThreads = 23</font>
+ <font style="color:rgb(25, 27, 31);">64核的情况下，ParallelGCThreads = 43</font>
+ <font style="color:rgb(25, 27, 31);">72核的情况下，ParallelGCThreads = 48</font>

**<font style="color:rgb(25, 27, 31);">ConcGCThreads</font>**

<font style="color:rgb(25, 27, 31);">ConcGCThreads的默认值则为：</font>

<font style="color:rgb(25, 27, 31);">ConcGCThreads = (ParallelGCThreads + 3)/4 向下去整。</font>

+ <font style="color:rgb(25, 27, 31);">ParallelGCThreads = 1~4时，ConcGCThreads = 1</font>
+ <font style="color:rgb(25, 27, 31);">ParallelGCThreads = 5~8时，ConcGCThreads = 2</font>
+ <font style="color:rgb(25, 27, 31);">ParallelGCThreads = 13~16时，ConcGCThreads = 4</font>

# G1：

<font style="color:#DF2A3F;">堆内存大于4g，且对停顿时间比较敏感的业务建议使用G1</font>

**-XX:+UseG1GC**

使用 G1 收集器

**-XX:MaxGCPauseMillis=200**

指定目标停顿时间，默认值 200 毫秒。在设置 -XX:MaxGCPauseMillis 值的时候，不要指定为平均时间，而应该指定为满足 90%
的停顿在这个时间之内。记住，停顿时间目标是我们的目标，不是每次都一定能满足的。

**-XX:InitiatingHeapOccupancyPercent=45**

整堆使用达到这个比例后，触发并发 GC 周期，默认 45%。

如果要降低晋升失败的话，通常可以调整这个数值，使得并发周期提前进行。

**-XX:NewRatio=n**

老年代/年轻代，默认值 2，即 1/3 的年轻代，2/3 的老年代。不要设置年轻代为固定大小，否则：G1 不再需要满足我们的停顿时间目标，不能再按需扩容或缩容年轻代大小。

**-XX:SurvivorRatio=n**

Eden/Survivor，默认值 8，这个和其他分代收集器是一样的

**-XX:MaxTenuringThreshold =n**

从年轻代晋升到老年代的年龄阈值，也是和其他分代收集器一样的

**-XX:ParallelGCThreads=n**

并行收集时候的垃圾收集线程数

**-XX:ConcGCThreads=n**

并发标记阶段的垃圾收集线程数，增加这个值可以让并发标记更快完成，如果没有指定这个值，

JVM 会通过以下公式计算得到：

ConcGCThreads=(ParallelGCThreads + 2) / 4^3

**-XX:G1ReservePercent=n**

堆内存的预留空间百分比，默认 10，用于降低晋升失败的风险，即默认地会将 10% 的堆内存预留下来。

**-XX:G1HeapRegionSize=n**

每一个 region 的大小，默认值为根据堆大小计算出来，取值 1MB~32MB，这个我们通常指定整堆大小就好了。

<font style="color:rgb(24, 24, 24);"></font>

<font style="color:rgb(24, 24, 24);"></font>

