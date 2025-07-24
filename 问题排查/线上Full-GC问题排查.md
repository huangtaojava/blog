## 前言

近期，项目在集成aviator规则引擎后，线上环境出现了频繁的FullGC现象，这严重影响了系统的稳定性与性能，对业务的正常运行造成了极大困扰，因此我们迅速展开了问题排查工作。

## 1.查看GC日志分析

为了找出FullGC频繁发生的原因，我们首先查看了GC日志，以下是部分关键的GC日志信息:

```java
-XX:+PrintHeapAtGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:/Users/huangtao/htgithub/ht-demos/spring-boot-demos/spring-boot-aviator/logs/gc.log
```

```java
{Heap before
GC invocations = 45 (full 11):
PSYoungGen total 80384K,used 28248K [0x00000007bab00000,0x00000007c0000000,0x00000007c0000000)
eden space 73728K,33%used [0x00000007bab00000,0x00000007bc3361c8,0x00000007bf300000)
from space 6656K,51%used [0x00000007bf300000,0x00000007bf660000,0x00000007bf980000)
to space 6656K,0%used [0x00000007bf980000,0x00000007bf980000,0x00000007c0000000)
ParOldGen total 175104K,used 10421K [0x00000007b0000000,0x00000007bab00000,0x00000007bab00000)
object space 175104K,5%used [0x00000007b0000000,0x00000007b0a2d7b8,0x00000007bab00000)
Metaspace used 49224K,capacity 80334K,committed 80384K,reserved 1118208K

class space    used 9817K,capacity 12416K,committed 12464K,reserved 1048576K
18.195:[

GC(Metadata GC Threshold) [PSYoungGen:28248K->3936

K(80384K)]38670K->14357

K(255488K), 0.0031892secs][Times:user=0.01sys=0.00,real=0.00secs]
Heap after
GC invocations = 45 (full 11):
PSYoungGen total 80384K,used 3936K [0x00000007bab00000,0x00000007c0000000,0x00000007c0000000)
eden space 73728K,0%used [0x00000007bab00000,0x00000007bab00000,0x00000007bf300000)
from space 6656K,59%used [0x00000007bf980000,0x00000007bfd58000,0x00000007c0000000)
to space 6656K,0%used [0x00000007bf300000,0x00000007bf300000,0x00000007bf980000)
ParOldGen total 175104K,used 10421K [0x00000007b0000000,0x00000007bab00000,0x00000007bab00000)
object space 175104K,5%used [0x00000007b0000000,0x00000007b0a2d7b8,0x00000007bab00000)
Metaspace used 49224K,capacity 80334K,committed 80384K,reserved 1118208K

class space    used 9817K,capacity 12416K,committed 12464K,reserved 1048576K
}
```

从日志中可以清晰地看到，FullGC的触发原因是“Metadata GC Threshold”，这表明元空间的使用出现了问题。
<br>
元空间用于存储类的元数据信息，当元空间不足时，就会触发FullGC来进行回收。从日志中的数据可知，元空间的使用量已接近其容量，这正是导致频繁FullGC的直接原因。
<br>
为了弄清楚为何元空间会快速被消耗，我们添加了-XX:+TraceClassLoading参数，用于跟踪类的加载情况，通过该参数的输出，我们发现了大量类似以下的日志：

```java
-XX:+TraceClassLoading
```

```java
[Loaded Script_1753322577883_69983/1590072645
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69984/442315763
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69985/2001062540
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69986/2084822785
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69987/382013809
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69988/1710521073
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69989/1019010409
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69990/1592108584
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69991/1620408860
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69992/653305613
from com.googlecode.aviator.Expression]
        [
Loaded Script_1753322577883_69993/2096980271
from com.googlecode.aviator.Expression]
```

这表明系统在频繁地加载由aviator规则引擎生成的Script类，大量的类加载必然会消耗大量的元空间，从而导致元空间不足，引发频繁的FullGC。

## 2.问题原因分析

为了找到aviator规则引擎频繁生成类的原因，我们查阅了其相关源码，发现aviator规则引擎的compile方法默认没有使用缓存模式，其代码如下：

```java
public Expression compile(String expression) {
    return compile(expression, false);
}

public Expression compile(final String expression, final boolean cached) {
    if (expression == null || expression.trim().length() == 0) {
        throw new CompileExpressionErrorException("Blank expression");
    }

    if (cached) {
        FutureTask<Expression> task = cacheExpressions.get(expression);
        if (task != null) {
            return getCompiledExpression(expression, task);
        }
        task = new FutureTask<Expression>(new Callable<Expression>() {
            @Override
            public Expression call() throws Exception {
                return innerCompile(expression, cached);
            }

        });
        FutureTask<Expression> existedTask = cacheExpressions.putIfAbsent(expression, task);
        if (existedTask == null) {
            existedTask = task;
            existedTask.run();
        }
        return getCompiledExpression(expression, existedTask);

    } else {
        return innerCompile(expression, cached);
    }

}
```

其中，第二个参数为false，即不启用缓存。这意味着每次调用compile方法编译表达式时，都会生成新的类，而不是复用已有的类。
在项目中如果频繁调用compile方法，就会导致大量的新类被不断加载，进而消耗大量元空间，最终引发频繁的FullGC。

## 3.解决方案

找到问题根源后，我们的解决办法是改用aviator规则引擎的缓存模式，将compile方法的第二个参数改为true，启用缓存，修改后的调用如下：

```java
Object execute = aviatorEvaluator.compile(expression, true).execute(params);
```

启用缓存模式后，aviator规则引擎会对编译过的表达式进行缓存，当再次编译相同的表达式时，会复用已缓存的结果，而不会生成新的类。
<br>
经过这样的修改后，系统不再频繁加载新的Script类，元空间的使用趋于稳定，频繁FullGC现象也随之消失，系统的性能和稳定性得到了显著提升。