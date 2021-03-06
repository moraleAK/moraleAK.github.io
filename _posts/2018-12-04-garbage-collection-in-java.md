---
title: Java 垃圾收集
layout: posts
categories: java, garbage collection, gc
---

# Java 垃圾收集

-----

对*标记-清除*垃圾收集的介绍还仅仅停留在理论上，实际应用中，为了适应现实场景和需要，很多地方需要做调整。下面我们举个简单的例子，来看看 JVM 需要做些什么工作，才能让我们安全的给对象分配空间。

------

## 碎片和整理（Fragmenting and Compacting）

一旦清除工作开始，JVM 必须确保*不可达*对象占用的内存空间可以被重新使用。这会导致内存碎片，和硬盘碎片类似，又会导致下面 2 个问题：

* 写操作更加耗时，因为寻找下一块足够大的可用空间所需时间变长了。
* 由于创建新对象时，JVM 会分配连续的内存块，因此，如果碎片化严重到没有任何一块碎片能够容纳新创建的对象，内存分配就报错了。

为了避免这些问题，JVM 需要控制内存*碎片化*的程度。因此，在垃圾收集时，除了*标记-清除*对象外，一个*去碎片化*的*整理*任务也在进行，它为*可达*对象重新分配空间，使它们紧挨在一起，消除（或减轻）碎片化，效果如下图所示。

![碎片化的堆 vs 整理后的堆](/images/2018-12-04-fragmented-vs-compacted-heap.png)

------

## 分代假设（Generational Hypothesis）

我们以前提过，垃圾收集时必须完全暂停应用。很明显，应用中对象越多，垃圾收集的时间越长。但是，如果可以在更小的内存区域做垃圾收集会咋样呢？在研究这些可能性时，一组研究人员发现，应用中分配的大多数对象可落入下面 2 类：

* 大部分很快就不用了；
* 小部分对象通常会存活（很）长时间。

这些发现催生了*弱分代假设*。而基于这个假设，JVM 中的内存被划分成了*年轻代*和*老年代*。

有了这样各自独立的清理区域，以前为了改进垃圾收集性能的众多算法都能派上用场了。

并不是说这个方法就没有问题。比如，不同代中的对象可能会各自持有对方的引用。当 JVM 在某代执行垃圾收集时，这些对象也是事实上的*根对象*。
更重要的是，分代假设在某些应用中实际上就不成立。因为垃圾收集算法为`死的早`或`可能永生`的对象做了优化，所以，JVM 对介于这两者之间的对象的处理表现的**相当差**。

![object-age-based-on-GC-generation-generational-hypothesis](/images/2018-12-04-object-age-based-on-GC-generation-generational-hypothesis.png)

------

## 内存池（Memory Pools）

大家应该很熟悉下面的内存池分区图，只是不太清楚*垃圾收集*在不同的内存池上是如何工作的。注意,不同的回收算法实现细节可能不一样，但文中提到的概念依然是相同的。

![java-heap-eden-survivor-old](/images/2018-12-04-java-heap-eden-survivor-old.png)

### 伊甸区（Eden）

*伊甸区*是通常创建对象时的内存分配区域。一般情况下，JVM 中会有多个线程同时创建大量的对象，因此*伊甸区*中又进一步划分出一个或多个*线程本地分配缓存*（TLAB），这样各线程可以在自身的 TLAB 中直接分配对象，而不需要与其他线程进行昂贵的同步操作。

当 TLAB 中因空间不足而不能分配对象时，JVM 就会转到共享的*伊甸区*中分配对象。如果共享*伊甸区*也空间不足时，会触发*年轻代*的垃圾收集任务释放空间。如果垃圾收集之后，*伊甸区*空间依然不足，那么 JVM 会从*老年代*分配对象。

当*伊甸区*进行垃圾收集时，垃圾收集器从*根对象*遍历所有*可达*对象并标记它们为存活状态。上文我们说过，对象可能会在不同的代之间交叉引用，那么一个很直接的方法就是检查所有其他代中的对象到*伊甸区*的引用，但这样做很不幸地偏离了我们*内存分代*的初衷。JVM 耍了技巧：*卡片标记*。本质上来说，JVM 只是标记那些从*老年代*可能有指向它们的脏对象在*伊甸区*的大概位置，详情参见[这里](http://psy-lob-saw.blogspot.com/2014/10/the-jvm-write-barrier-card-marking.html)。

![TLAB-in-Eden-memory](/images/2018-12-04-TLAB-in-Eden-memory.png)

标记阶段完成后，所有的存活对象会被复制（不是移动）到其中一个*存活区*，之后整个*伊甸区*完全空闲，可以再次进行对象分配。这种方法就是*标记-复制*：标记存活对象，然后拷贝到*存活区*。

### 存活区（Survivor Spaces）

紧靠着*伊甸区*的就是 2 个*存活区*了，角色分别为`from`和`to`。必须要注意的是，`to`总是空闲的。下一次*年轻代*垃圾收集时，*年轻代*中所有的存活对象（包括*伊甸区*和非空`from`存活区）将被复制到存活区`to`中。垃圾收集结束后，`from`空闲，`to`中含有存活对象，此时，它们的角色将被互换，`from`成为`to`，`to`变成`from`。

![how-java-garbage-collection-works](/images/2018-12-04-how-java-garbage-collection-works.png)

2 个*存活区*之间复制存活对象的过程会被重复好几次，直到有些对象被认为已经成熟并且足够年长。记住，基于*分代假设*，已存活一段时间的对象被期望继续存活很长的时间。这样年长的对象就可以被提升到*老年代*中去。年长对象会从*存活区 from*移动到*老年代*，而不再复制到*存活区 to*中。

为了确定一个对象是否足够年长因而可以被提升到*老年代*，垃圾收集器会跟踪一个对象经历的垃圾回收次数，也就是对象年龄。当年龄达到一个阀值，这个对象就会被提升到*老年代*。

JVM 可以动态调整实际的年龄阀值，参数`-XX:+MaxTenuringThreshold`设置阈值上限。

比如，设置`-XX:+MaxTenuringThreshold=0`的结果就是对象直接从*存活区*提升到*老年代*，而不会在 2 个*存活区*之间来回复制。默认情况下，现代 JVM 的年龄阀值为 15，这也是 HotSpot 虚拟机的最大值。

如果*存活区*没有足够的空间容纳*年轻代*中所有的存活对象，那么*年轻*对象也有可能提前被提升到*老年代*。

### 老年代（Old Generation）

*老年代*中的情况更加复杂。由于*老年代*内存空间通常更大，里面的对象不太可能需要被垃圾回收，因此，垃圾收集的频率要低于*年轻代*。并且，由于*老年代*中的大部分对象被期望一直存活着，这里不会发生*标记-复制*对象，取而代之的是，移动它们来最小化*碎片程度*。清理*老年代*的垃圾收集算法通常建立在不同的基础之上，但总体步骤类似：

1. 标记从*根对象*开始的*可达*对象；
2. 删除所有*不可达*对象；
3. 通过复制存活对象的方式整理*老年代*内存空间。

通过上面的描述，你可以看出，*老年代*必须显式的做*内存整理*来减轻*碎片化*的程度。

### 永久代（PermGen）

Java 8 之前的 JVM 中还有一个叫*永久代*的内存区。这里存放像`Class`这样的元数据，还有内部池化的字符串等等。它实际上给 Java 开发者带了了很多麻烦，因为很难预测它到底需要占用多少内存空间，而如果预测失败，就会出现`java.lang.OutOfMemoryError: Permgen space`这样的异常。除非是由于内存泄露导致的`OOME`，否则一般消除这个异常的方法就像下面的示例一样，只要简单的增加*永久代*最大内存空间阀值。

{% highlight bash linenos %}
java -XX:MaxPermSize=256m com.mycompany.MyApplication
{% endhighlight %}

### 元数据区（Metaspace）

由于预测元数据所需空间大小的复杂性和不便性，Java 8中移除了*永久代*，代之以*元数据区*，从此以后，大部分繁杂的东西都转移到了常规的 Java 堆中。

*类定义*现在被加载到*元数据区*。它位于操作系统原生内存区上，并不和 Java 堆上的对象打交道。*元数据区*的默认大小只受限于操作系统给 Java 进程分配的内存大小，这样就不会出现开发者仅仅是想在应用中增加一个类而导致`OOME:Permgen space`异常了。但是要注意，*元数据区*大小不受限于 Java 堆并不是没有代价的，无限制的增长会导致操作系统频繁的页交换和（或）代之以系统内存分配失败。

假如你想在这种场合下保护自己，你可以主动设置*元数据区*的最大空间阈值，比如 256M：

{% highlight bash linenos %}
java -XX:MaxMetaspaceSize=256m com.mycompany.MyApplication
{% endhighlight %}

------

## Minor GC vs Major GC vs Full GC

清理不同堆内区域的垃圾收集事件常常被叫做`Minor GC`，`Major GC`和`Full GC`。下面我们来看看它们的区别，在这个过程中，我们希望能看到这个区别实际上对我们来说并不是太有意义。一般情况下，有意义的是你的应用有没有达到它的服务等级协议（SLA）、应用的延迟和吞吐量。此时垃圾收集才会和结果相关：它们有没有暂停应用和需要暂停多久。

但是，由于`Minor GC`，`Major GC`和`Full GC`这些术语被广泛使用，并且没有一个适当的定义，让我们更详细的深入这个主题来看看。

### Minor GC

*年轻代*中的垃圾收集叫`Minor GC`。这个定义很清楚且达成了共识。但是，当你在涉及`Minor GC`时，下面这些小知识点必须要清楚。

1. `Minor GC`总是在 JVM 不能给新对象分配空间时触发，比如，*伊甸区*满了。因此，对象分配越频繁，`Minor GC`频率越高。
2. `Minor GC`时，*老年代*实际上是被忽略的。在标记阶段，从*老年代*到*年轻代*的引用被认为是*根对象*，而*年轻代*到*老年代*的引用被简单的无视了。
3. 与常识相悖的是，`Minor GC`会暂停应用，挂起应用线程。对大多数应用来说，如果*伊甸区*中的大部分对象都是垃圾且永远不用复制到*存活区*或*老年区*，那么暂停的时间是很轻微的，可以忽略。反之，如果大部分新生对象不能被垃圾收集，那么`Minor GC`会暂停相当的一段时间。

好了，定义`Minor GC`很简单：`Minor GC`清理*年轻代*。

### Major GC vs Full GC

不管是在 JVM 规范还是在垃圾收集的研究论文中，这两个术语都没有被正式的定义。乍一看，基于我们对`Minor GC`清理*年轻代*的理解，可以简单定义它们：

* `Major GC`清理*老年代*。
* `Full GC`清理整个 Java 堆，包括*年轻代*和*老年代*。

不幸的是，这更复杂和令人困惑。首先，很多`Major GC` 是被`Minor GC`触发的，因此割离它们两个在很多情况下是不可能的。另一方面，现代垃圾收集算法，比如 G1，会执行部分清理，因此，说*清理*只说对了一部分。

这让我们与其担心垃圾收集叫`Major GC`或`Full GC`，我们更该关心的是垃圾收集时，是挂起了全部的应用线程还是可以和应用线程并发执行。

> 原文地址：[Garbage Collection in Java](https://plumbr.io/handbook/garbage-collection-in-java)。

------

## 相关文章

* [Java 垃圾收集](/garbage-collection-in-java/)
* [Java 垃圾收集算法：基础篇](/garbage-collection-algorithms-basics/)
* [Java 垃圾收集算法：Serial GC](/garbage-collection-algorithms-serial-gc/)
* [Java 垃圾收集算法：Parallel GC](/garbage-collection-algorithms-parallel-gc/)
* [Java 垃圾收集算法：Concurrent Mark and Sweep](/garbage-collection-algorithms-concurrent-mark-and-sweep/)
* [Java 垃圾收集算法：G1](/garbage-collection-algorithms-garbage-first/)

