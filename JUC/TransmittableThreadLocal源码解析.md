**ThreadLocal**相信大家并不陌生，它是 Java 中用于实现线程封闭（Thread
confinement）的一种机制，它允许每个线程拥有自己独立的变量副本，从而在多线程环境下实现数据的隔离性。然而，尽管 ThreadLocal
提供了便利，但也存在一些缺点。其中最显著的是在涉及线程池或者线程复用的情况下，ThreadLocal 变量的传递会遇到问题。

当线程池中的线程复用时，ThreadLocal 的值并不会被自动传递给新的线程。这导致在跨线程调用时，原始线程中设置的 ThreadLocal
变量无法被新线程访问，进而可能导致错误的行为或意外的结果。这种情况下，ThreadLocal 的作用变得相对受限，限制了其在特定多线程场景下的适用性。

针对这些 ThreadLocal 的局限性，**TransmittableThreadLocal** 应运而生。它是阿里巴巴开源的一个库，旨在解决 ThreadLocal
在线程池或者线程复用情况下的传递问题，能够有效地克服 ThreadLocal 的限制，使得在多线程环境下数据的传递更加可靠和灵活。接下来，我们将从源码角度深入探讨
TransmittableThreadLocal 的实现原理和解决方案，以及它是如何优雅地解决了 ThreadLocal 的不足之处。

# 一、ThreadLocal的局限性

我们运行下列两组代码，对比一下ThreadLocal和TransmittableThreadLocal的执行结果

````
public class TestThreadLocal {
    public static void main(String[] args) {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        for (int i = 1; i <= 4; i++) {
            threadLocal.set("Java Boy"+i);
            System.out.println("主线程:"+threadLocal.get());
            executorService.execute(()->{
                System.out.println("---子线程---:"+threadLocal.get());
            });
        }
    }
}
````

![TransmittableThreadLocal源码解析-1.png](../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-1.png)

````
public class TestTransmittableThreadLocal {
    public static void main(String[] args) {
        ThreadLocal<String> threadLocal = new TransmittableThreadLocal<>();
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        //这一步很重要
        executorService = TtlExecutors.getTtlExecutorService(executorService);
        for (int i = 1; i <= 4; i++) {
            threadLocal.set("Java Boy"+i);
            System.out.println("主线程:"+threadLocal.get());
            executorService.execute(()->{
                System.out.println("子线程:"+threadLocal.get());
            });
        }
    }
}
````

![TransmittableThreadLocal源码解析-2.png](../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-2.png)

当我们使用 ThreadLocal 时，子线程无法直接获取到父线程所设置的 ThreadLocal 变量。这种现象是合理的，因为 ThreadLocal
是与线程相关联的线程本地变量，所以子线程无法直接访问父线程的 ThreadLocal
变量。然而，在实际业务中，我们常常需要在多线程异步操作中使子线程能够获取到全局变量的情况。这种情况下，ThreadLocal
就无法满足我们的需求。在这种情况下，TransmittableThreadLocal 的出现能够很好地解决这个问题，允许子线程有效地获取到在父线程中设置的变量，从而实现了在多线程环境中全局变量的传递。

# 二、TransmittableThreadLocal源码解析

TransmittableThreadLocal [github地址](https://github.com/alibaba/transmittable-thread-local)

首先我们看下TransmittableThreadLocal类

![TransmittableThreadLocal源码解析-3.png](../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-3.png)
再看下set和get方法

<img alt="TransmittableThreadLocal源码解析-4.png" height="300" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-4.png" width="600"/>

<img alt="TransmittableThreadLocal源码解析-5.png" height="300" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-5.png" width="600"/>

可以看到，TransmittableThreadLocal继承了InheritableThreadLocal和ThreadLocal，虽然重写了set及get方法，但是最终这两个方法都是调用了父类中的方法，这样肯定是无法实现线程池变量传递的。

我们接着往下看，TransmittableThreadLocal是的使用姿势中对我们的线程池TtlExecutors.getTtlExecutorService，我们点进这个方法看一下

<img alt="TransmittableThreadLocal源码解析-6.png" height="300" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-6.png" width="600"/>

看到Wapper，很明显，这里是使用了装饰器模式，对我们的线程池进行了包装操作，直接看代码可能还不够清晰，我们进行debug断点调试吧。

<img alt="TransmittableThreadLocal源码解析-7.png" height="300" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-7.png" width="600"/>
<br>
<img alt="TransmittableThreadLocal源码解析-8.png" height="100" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-8.png" width="800"/>
<br>
<img alt="TransmittableThreadLocal源码解析-9.png" height="300" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-9.png" width="600"/>
<br>
<img alt="TransmittableThreadLocal源码解析-10.png" height="150" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-10.png" width="600"/>
<br>
<img alt="TransmittableThreadLocal源码解析-11.png" height="400" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-11.png" width="800"/>
 <br>
<img alt="TransmittableThreadLocal源码解析-12.png" height="600" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-12.png" width="800"/>
<br>

代码执行到这里做了这几件事：TransmittableThreadLocal对线程池进行了包装，将扔到线程池的Runnable先组装成TtlRunnable，将父线程的TransmittableThreadLocal变量记录到TtlRunnable的capturedRef属性上，TtlRunnable重写了run方法，在执行Runnable.run()
之前，将父线程中的变量值取了出来。

我们再接着往下看

<img alt="TransmittableThreadLocal源码解析-13.png" height="500" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-13.png" width="800"/>
<br>

<img alt="TransmittableThreadLocal源码解析-14.png" height="400" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-14.png" width="800"/>
<br>

<img alt="TransmittableThreadLocal源码解析-15.png" height="400" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-15.png" width="800"/>

<br>

<img alt="TransmittableThreadLocal源码解析-16.png" height="200" src="../image/TransmittableThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-16.png" width="800"/>

这里将取出的父线程中的变量值captured赋给了子线程，从而成功实现了线程本地变量在父子线程中的传递。

# 三、总结

在本文中，我们对 TransmittableThreadLocal 核心源码进行了探讨，特别是其在实现线程传递过程中的核心机制。然而，TransmittableThreadLocal
的实现远不止所介绍的内容那么简单。在其背后涉及了更多深入的细节和技术原理，包括线程池中线程的管理、序列化和反序列化机制、内存模型等方面。

在深入学习 TransmittableThreadLocal 的源码过程中，你会发现其实现中涉及了对 Java
并发编程的精妙设计和高效实现。阅读源码不仅能够加深对其工作原理的理解，还能够培养我们解决复杂多线程问题的能力。

如果你对 Java 并发编程和线程传递机制感兴趣，强烈建议花时间深入研究 TransmittableThreadLocal
的源码和相关技术细节。通过阅读源码、调试和实验，你将更深入地了解其内部工作原理，并从中获得宝贵的经验和知识。

*喜欢的可以点个关注，我将持续更新技术文章，谢谢。*

本人掘金文章链接 ：[TransmittableThreadLocal源码解析](https://juejin.cn/post/7308620787329564723)