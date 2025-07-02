<img alt="jvm内存区域-1.png" height="500" src="../image/jvm%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F-1.png" width="700"/>

### ·程序计数器

<font style="color:#DF2A3F;">程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。</font>
字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完成。

线程私有：java多线程是通过线程轮流切换，分配处理器执行时间的方式实现的。为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，

⚠️ 注意：<font style="color:#DF2A3F;">程序计数器是唯一一个不会出现 OutOfMemoryError
的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。</font>

### ·Java 虚拟机栈

与程序计数器一样，Java 虚拟机栈（后文简称栈）也是线程私有的，它的生命周期和线程相同，随着线程的创建而创建，随着线程的死亡而死亡。

栈绝对算的上是 JVM 运行时数据区域的一个核心，除了一些 Native 方法调用是通过本地方法栈实现的(后面会提到)，其他所有的 Java
方法调用都是通过栈来实现的（也需要和其他运行时数据区域比如程序计数器配合）。

方法调用的数据需要通过栈进行传递，每一次方法调用都会有一个对应的栈帧被压入栈中，每一个方法调用结束后，都会有一个栈帧被弹出。

栈由一个个栈帧组成，而每个栈帧中都拥有：局部变量表、操作数栈、动态链接、方法返回地址。和数据结构上的栈类似，两者都是先进后出的数据结构，只支持出栈和入栈两种操作。

**<font style="color:#DF2A3F;">局部变量表</font>**
主要存放了编译期可知的各种数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference
类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。

**<font style="color:#DF2A3F;">操作数栈</font>** 主要作为方法调用的中转站使用，用于存放方法执行过程中产生的中间计算结果。另外，计算过程中产生的临时变量也会放在操作数栈中。

**<font style="color:#DF2A3F;">动态链接</font>**<font style="color:#DF2A3F;"> </font>主要服务一个方法需要调用其他方法的场景。Class
文件的常量池里保存有大量的符号引用比如方法引用的符号引用。当一个方法要调用其他方法，需要将常量池中指向方法的符号引用转化为其在内存地址中的直接引用。动态链接的作用就是为了将符号引用转换为调用方法的直接引用，这个过程也被称为
**动态连接** 。

<img alt="jvm内存区域-2.png" height="400" src="../image/jvm%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F-2.png" width="500"/>

栈空间虽然不是无限的，但一般正常调用的情况下是不会出现问题的。不过，如果函数调用陷入无限循环的话，就会导致栈中被压入太多栈帧而占用太多空间，导致栈空间过深。那么当线程请求栈的深度超过当前
Java 虚拟机栈的最大深度的时候，就抛出 StackOverFlowError 错误。

Java 方法有两种返回方式，一种是 return 语句正常返回，一种是抛出异常。不管哪种返回方式，都会导致栈帧被弹出。也就是说， *
*栈帧随着方法调用而创建，随着方法结束而销毁。无论方法正常完成还是异常完成都算作方法结束。**

除了 StackOverFlowError 错误之外，栈还可能会出现OutOfMemoryError错误，这是因为如果栈的内存大小可以动态扩展，
如果虚拟机在动态扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常。

简单总结一下程序运行中栈可能会出现两种错误：

+ **StackOverFlowError****：** 若栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前 Java 虚拟机栈的最大深度的时候，就抛出
  StackOverFlowError 错误。
+ **OutOfMemoryError：** 如果栈的内存大小可以动态扩展， 如果虚拟机在动态扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常。

### ·本地方法栈

和虚拟机栈所发挥的作用非常相似，区别是：**虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的
Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

本地方法被执行的时候，在本地方法栈也会创建一个栈帧，用于存放该本地方法的局部变量表、操作数栈、动态链接、出口信息。

方法执行完毕后相应的栈帧也会出栈并释放内存空间，也会出现 StackOverFlowError 和 OutOfMemoryError 两种错误。

### ·堆

Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。*
*此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**

Java 世界中“几乎”所有的对象都在堆中分配，但是，随着 JIT
编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化，所有的对象都分配到堆上也渐渐变得不那么“绝对”了。从
JDK 1.7 开始已经默认开启逃逸分析，如果某些方法中的对象引用没有被返回或者未被外面使用（也就是未逃逸出去），那么对象可以直接在栈上分配内存。

Java 堆是垃圾收集器管理的主要区域，因此也被称作 **GC 堆（Garbage Collected Heap）**。从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以
Java 堆还可以细分为：新生代和老年代；再细致一点有：Eden、Survivor、Old 等空间。进一步划分的目的是更好地回收内存，或者更快地分配内存。

在 JDK 7 版本及 JDK 7 版本之前，堆内存被通常分为下面三部分：

1. 新生代内存(Young Generation)
2. 老生代(Old Generation)
3. 永久代(Permanent Generation)

**JDK 8 版本之后 PermGen(永久代) 已被 Metaspace(元空间) 取代，元空间使用的是本地内存。**

大部分情况，对象都会首先在 Eden 区域分配，在一次新生代垃圾回收后，如果对象还存活，则会进入 S0 或者 S1，并且对象的年龄还会加
1(Eden 区->Survivor 区后对象的初始年龄变为 1)，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数
-XX:MaxTenuringThreshold 来设置。

“Hotspot 遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了 survivor 区的一半时，取这个年龄和
MaxTenuringThreshold 中更小的一个值，作为新的晋升年龄阈值”。

**动态年龄计算的代码如下**

```cpp
uint ageTable::compute_tenuring_threshold(size_t survivor_capacity) {
    //survivor_capacity是survivor空间的大小
    size_t desired_survivor_size = (size_t)((((double) survivor_capacity)*TargetSurvivorRatio)/100);
    size_t total = 0;
    uint age = 1;
    while (age < table_size) {
        total += sizes[age];//sizes数组是每个年龄段对象大小
        if (total > desired_survivor_size) break;
        age++;
    }
    uint result = age < MaxTenuringThreshold ? age : MaxTenuringThreshold;
    ...
}

```

堆这里最容易出现的就是 OutOfMemoryError 错误，并且出现这种错误之后的表现形式还会有几种，比如：

1. **java.lang.OutOfMemoryError: GC Overhead Limit Exceeded**：当 JVM 花太多时间执行垃圾回收并且只能回收很少的堆空间时，就会发生此错误。
2. **java.lang.OutOfMemoryError: Java heap space** :假如在创建新的对象时, 堆内存中的空间不足以存放新创建的对象,
   就会引发此错误。(和配置的最大堆内存有关，且受制于物理内存大小。最大堆内存可通过-Xmx参数配置，若没有特别配置，将会使用默认值

### ·方法区

方法区属于是 JVM 运行时数据区域的一块逻辑区域，是各个线程共享的内存区域。

《Java 虚拟机规范》只是规定了有方法区这么个概念和它的作用，方法区到底要如何实现那就是虚拟机自己要考虑的事情了。也就是说，在不同的虚拟机实现上，方法区的实现是不同的。

当虚拟机要使用一个类时，它需要读取并解析 Class 文件获取相关信息，再将信息存入到方法区。方法区会存储已被虚拟机加载的 *
*类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。

**方法区和永久代以及元空间是什么关系呢？** 方法区和永久代以及元空间的关系很像 Java
中接口和类的关系，类实现了接口，这里的类就可以看作是永久代和元空间，接口可以看作是方法区，也就是说永久代以及元空间是 HotSpot
虚拟机对虚拟机规范中方法区的两种实现方式。并且，永久代是 JDK 1.8 之前的方法区实现，JDK 1.8 及以后方法区的实现变成了元空间。

**为什么要将永久代 (PermGen) 替换为元空间 (MetaSpace) 呢?**

下图来自《深入理解 Java 虚拟机》第 3 版 2.2.5

![jvm内存区域-3.png](../image/jvm%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F-3.png)

1、整个永久代有一个 JVM 本身设置的固定大小上限，无法进行调整，而元空间使用的是本地内存，受本机可用内存的限制，虽然元空间仍旧可能溢出，但是比原来出现的几率会更小。

当元空间溢出时会得到如下错误：java.lang.OutOfMemoryError: MetaSpace

你可以使用 -XX：MaxMetaspaceSize 标志设置最大元空间大小，默认值为 unlimited，这意味着它只受系统内存的限制。-XX：MetaspaceSize
调整标志定义元空间的初始大小如果未指定此标志，则 Metaspace 将根据运行时的应用程序需求动态地重新调整大小。

2、元空间里面存放的是类的元数据，这样加载多少类的元数据就不由 MaxPermSize 控制了, 而由系统的实际可用空间来控制，这样能加载的类就更多了。

3、在 JDK8，合并 HotSpot 和 JRockit 的代码时, JRockit 从来没有一个叫永久代的东西, 合并之后就没有必要额外的设置这么一个永久代的地方了。

### ·运行时常量池

Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有用于存放编译期生成的各种字面量（Literal）和符号引用（Symbolic
Reference）的 **常量池表(Constant Pool Table)** 。

### ·字符串常量池

**字符串常量池** 是 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。

<font style="color:#DF2A3F;background-color:rgb(238, 240, 244);">
常量与常量的拼接结果在常量池。且常量池中不会存在相同内容的常量。</font><font style="color:#DF2A3F;">  
</font><font style="color:#DF2A3F;background-color:rgb(238, 240, 244);">
只要其中有一个是变量，结果就在堆中</font><font style="color:#DF2A3F;">  
</font><font style="color:#DF2A3F;background-color:rgb(238, 240, 244);"> 如果拼接的结果调用intern()
方法，返回值就在常量池中 </font>

```java
// 在堆中创建字符串对象”ab“
// 将字符串对象”ab“的引用保存在字符串常量池中
String aa = "ab";
// 直接返回字符串常量池中字符串对象”ab“的引用
String bb = "ab";
System.out.println(aa==bb);// true
```

**<font style="color:#DF2A3F;">1和5在字符串常量池，234不在</font>**

<img alt="jvm内存区域-4.png" height="350" src="../image/jvm%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F-4.png" width="300"/>

JDK1.7 之前，字符串常量池存放在永久代。JDK1.7 字符串常量池和静态变量从永久代移动了 Java 堆中。

**JDK 1.7 为什么要将字符串常量池移动到堆中？**

主要是因为永久代（方法区实现）的 GC 回收效率太低，只有在整堆收集 (Full GC)的时候才会被执行 GC。Java
程序中通常会有大量的被创建的字符串等待回收，将字符串常量池放到堆中，能够更高效及时地回收字符串内存。

<font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">java7 之前字符串常量池被放到了 Perm 区，所有被
intern 的 String 都会被存在这里，由于 String.intern
是不受控的，所以 </font><font style="color:rgb(51, 51, 51);background-color:rgb(228, 228, 228);">-XX:
MaxPermSize</font><font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">
的值也不太好设置，经常会出现 </font><font style="color:rgb(51, 51, 51);background-color:rgb(228, 228, 228);">
java.lang.OutOfMemoryError: PermGen
space</font><font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);"> 异常，所以在 Java7
之后常量池等字面量（Literal）、类静态变量（Class Static）、符号引用（Symbols Reference）等几项被移到 Heap 中。而 Java8 之后
PermGen 也被移除，取而代之的是 MetaSpace。</font>

**运行时常量池、方法区、字符串常量池这些都是不随虚拟机实现而改变的逻辑概念，是公共且抽象的，Metaspace、Heap
是与具体某种虚拟机实现相关的物理概念，是私有且具体的。**

**根据虚拟机规范，字符串常量，需要放在运行时常量池中。所以，我认为字符串池就是运行时常量池的一个逻辑子区域。即字符串池是运行时常量池的分池！
**

### ·直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，不在 Java
堆或方法区中分配的，而是通过 <font style="color:#DF2A3F;">JNI</font> 的方式在本地内存上分配的。

JDK1.4 中新加入的 NIO（Non-Blocking I/O，也被称为 New I/O），引入了一种基于通道（Channel）与缓存区（Buffer）的 I/O 方式，它可以直接使用
Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为避免了在
Java 堆和 Native 堆之间来回复制数据。

参考：

+ <font style="color:rgb(60, 60, 67);">《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》-周志明老师</font>
+ [https://javaguide.cn/java/jvm/memory-area.html](https://javaguide.cn/java/jvm/memory-area.html)

