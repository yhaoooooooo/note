---
typora-copy-images-to: img
typora-root-url: img
---

[TOC]

# JVM

## 1. JVM垃圾回收机制

- **判断对象的是否可以被回收**

  - 引用计数法：当对象被引用的时候，对象计数+1，引用消失-1，直到减为0， **但是会有一个问题，如果两个对象互相引用，则一直无法为0，无法被回收**

  - 可达性分析算法：从一个称为“ **GCROOT**“ 的对象，存在其引用链上的对象，无法被回收。 

    ![20180626084654607](/20180626084654607.png)

  - Java对引用的概念做了扩充，将引用分为强引用(Strong Reference)、软引用(Soft Reference)、弱引用(Weak Reference)和虚引用(Phantom Reference)四种，这四种引用的强度依次递减。**TODO  补充ThreadLocal 内存泄漏问题**

## 2. 垃圾回收算法

- **标记-清除法**

  根据可达性算法，可以把无用的对象进行标记，等待回收，但是有两个问题： 1. 效率问题： 标记和回收的效率都不高。 2. 空间里利用率问题：回收无用的对象，可能会造成大量的 内存碎片无法被利用。

![20180626093343689](/20180626093343689.jpg)

- **复制算法**

  将内存进行分块，每次使用的时候，只使用一半。 当进行垃圾回收的时候，将当前内存块的所有可用的对象复制到另外一块内存中。解决了清理的效率问题。**这样做的好处是每次都是对整个半区进行内存回收，内存分配时也就不需要考虑内存碎片等的复杂情况，只需要移动堆顶指针，按顺序分配即可。此算法实现简单，运行高效，，，，JVM中就是使用的这种算法，相当于将对象从Eden区复制到survivor区（大小 8：1）**

  **![20180626162321219](/20180626162321219.jpg)**

- **标记整理算法（老年代算法）**

  老年代中在执行GC时，将有用的对象向内存的中的一侧进行整理排序，避免出现碎片。

  ![20180626165125641](/20180626165125641.jpg)

- **分代收集算法**

  	当前JVM垃圾收集都采用的是"分代收集(Generational Collection)"算法，这个算法并没有新思想，只是根据对象存活周期的不同将内存划分为几块。
       一般是把Java堆分为新生代和老年代。在新生代中，每次垃圾回收都有大批对象死去，只有少量存活，因此我们采用复制算法；而老年代中对象存活率高、没有额外空间对它进行分配担保，就必须采用"标记-清理"或者"标记-整理"算法。

        **面试题**: 请问了解Minor GC和Full GC么，这两种GC有什么不一样吗？
        
         Minor GC又称为新生代GC : 指的是发生在新生代的垃圾收集。因为Java对象大多都具备朝生夕灭的特性，因此Minor GC(采用复制算法)非常频繁，一般回收速度也比较快。
          Full GC 又称为老年代GC或者Major GC : 指发生在老年代的垃圾收集。出现了Major GC，经常会伴随至少一次的Minor GC(并非绝对，在Parallel Scavenge收集器中就有直接进行Full GC的策略选择过程)。Major GC的速度一般会比Minor GC慢10倍以上。

## 3. 对象转移到老年代的几种情况

### 3.1 大对象直接从Eden区转移到老年代

		1.  对象 > survivor区大小，
  		2.  设置 - XX:PretenureSizeThreshold（**只在 Serial 和ParNew两个收集 器下有效**） 设置大对象大小

目的是，避免大对象来回复制。

### 3.2 长期存活的对象

-XX:MaxTenuringThreshold 可以设置对象 被MinorGC的次数，默认是15次。每GC一次，次数+1，当达到值以后，将对象转移到老年代。

### 3.3 对象动态年龄判断

**survivor区随机抽取一批对象，这批对象占内存大小，超过survivor区的50%，则把survivor区内年龄大于那批对象中的最大年龄的对象，直接转移到老年代。**目的是把那些大概率会存活很久的对象提前转移到老年区。

### 3.4 survivor盛不下存活的对象

### 3.5 老年代担保机制

​	minorGC之前，计算老年代可用空间是否大于年轻代的所有对象（包括垃圾对象），如果小于并且设置了担保机制“-XX:-HandlePromotionFailure”(**jdk1.8默认就设置了**)的参数是否设置了，设置了的话，计算年轻代每次转移到老年代的平均大小，如果小，执行fullGC，如果大，执行minor GC。如果没有设置担保参数，并且老年代的大小小于年轻代的对象大小，则执行full GC。

​	大概什么意思呢，就是说，害怕minorgc之后，转移到老年代的对象比较大，老年代盛不下，发生OOM。担保机制，就是计算下平均每次转移到老年代的对象，每次转移的对象小于剩余空间，就担保没问题，minorgc之后，老年代大概率是成的下的。

![image-20200411203852843](/image-20200411203852843.png)

## 4. 垃圾收集器

![image-20200408201035515](/image-20200408201035515.png)

​	**如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现**。 虽然我们对各个收集器进行比较，但并非为了挑选出一个最好的收集器。因为直到现在为止还没有 最好的垃圾收集器出现，更加没有万能的垃圾收集器，我们能做的就是根据具体应用场景选择适合 自己的垃圾收集器。试想一下：如果有一种四海之内、任何场景下都适用的完美收集器存在，那么 我们的Java虚拟机就不会实现那么多不同的垃圾收集器了。

### 4.1 Serial

参数：(-XX:+UseSerialGC -XX:+UseSerialOldGC)

Serial（串行）收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一 个单线程收集器了。它的 “单线程” 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃 圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ "Stop The World" ），直到它收集结束。

 新生代采用**复制算法**，老年代采用**标记-整理**算法。 

![image-20200411165259027](/image-20200411165259027.png)

**优点：**它简单而高效（与其他收集器的 单线程相比）。Serial收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。 Serial Old收集器是Serial收集器的老年代版本，它同样是一个单线程收集器。它主要有两大用 途：一种用途是在JDK1.5以及以前的版本中与Parallel Scavenge收集器搭配使用，另一种用途是 作为CMS收集器的后备方案。

### 4.2 ParNew收集器

参数：-XX:+UseParNewGC

ParNew收集器其实就是Serial收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为 （控制参数、收集算法、回收策略等等）和Serial收集器完全一样。默认的收集线程数跟cpu核数 相同，当然也可以用参数(-XX:ParallelGCThreads)指定收集线程数，但是一般不推荐修改。

 新生代采用**复制算法**。 

**它是许多运行在Server模式下的虚拟机的首要选择，除了Serial收集器外，只有它能与CMS收集器 （真正意义上的并发收集器，后面会介绍到）配合工作**

![image-20200411165648159](/image-20200411165648159.png)

### 4.3 Parallel Scavenge 收集器

参数：(-XX:+UseParallelGC(年轻代),- XX:+UseParallelOldGC(老年代))

**新生代采用复制算法，老年代采用标记-整理算法**。

和parnew类似（注重降低stw的时间），但是注重CPU的使用率，提高吞吐量。

![image-20200411171325593](/image-20200411171325593.png)

### 4.4 CMS（Concurrent Mark Sweep）收集器

参数：**-XX:+UseConcMarkSweepGC**

从名字中的**Mark Sweep**这两个词可以看出，**CMS收集器是一种 “标记-清除”算法实现的**，它 的运作过程相比于前面几种垃圾收集器来说更加复杂一些。整个过程分为四个步骤：

- 初始标记： 仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快，需要停顿（Stop-the-world）
- 并发标记： 进行 GC Roots Tracing 的过程，它在整个回收过程中耗时最长，不需要停顿
- 重新标记： 为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要停顿（Stop-the-world）
- 并发清除： 清理垃圾，不需要停顿
  ![image-20200411171857324](/image-20200411171857324.png)

**缺点：**

- 对CPU资源敏感（会和服务抢资源）； 
- 无法处理浮动垃圾(在并发清理阶段又产生垃圾，这种浮动垃圾只能等到下一次gc再清理 了)； 
- 它使用的回收算法-“**标记-清除**”算法会导致收集结束时会有大量空间碎片产生，当然 通过参数**-XX:+UseCMSCompactAtFullCollection** 可以让jvm在执行完标记清除后再做整 理 
- 执行过程中的不确定性，**会存在上一次垃圾回收还没执行完，然后垃圾回收又被触 发的情况**，特别是在**并发标记和并发清理阶**段会出现，一边回收，系统一边运行，也许没回 收完就再次触发full gc，也就是"concurrent mode failure"，**此时会进入stop the world，用serial old垃圾收集器来回收**

CMS的相关参数 

1. -XX:+UseConcMarkSweepGC：启用cms 
2. -XX:ConcGCThreads：并发的GC线程数
3.  -XX:+UseCMSCompactAtFullCollection：FullGC之后做压缩整理（减少碎片） 
4.  -XX:CMSFullGCsBeforeCompaction：多少次FullGC之后压缩一次，默认是0，代表每次 FullGC后都会压缩一次 
5.  -XX:CMSInitiatingOccupancyFraction: 当老年代使用达到该比例时会触发FullGC（默认 是92，这是百分比） 
6.  -XX:+UseCMSInitiatingOccupancyOnly：只使用设定的回收阈值(- XX:CMSInitiatingOccupancyFraction设定的值)，如果不指定，JVM仅在第一次使用设定 值，后续则会自动调整 
7. -XX:+CMSScavengeBeforeRemark：在CMS GC前启动一次minor gc，目的在于减少 老年代对年轻代的引用，降低CMS GC的标记阶段时的开销，一般CMS的GC耗时 80%都在 remark阶段

### 4.5 G1(Garbage-First)

参数:(-XX:+UseG1GC)

​	G1将Java堆划分为多个大小相等的独立区域（Region），JVM最多可以有2048个Region。

​	默认年轻代对堆内存的占比是5%，如果堆大小为4096M，那么年轻代占据200MB左右的内存， 对应大概是100个Region，可以通过“-XX:G1NewSizePercent”设置新生代初始占比，在系统 运行中，JVM会不停的给年轻代增加更多的Region，但是最多新生代的占比不会超过60%，可以 通过“-XX:G1MaxNewSizePercent”调整。年轻代中的Eden和Survivor对应的region也跟之前 一样，默认8:1:1，假设年轻代现在有1000个region，eden区对应800个，s0对应100个，s1对应 100个

​	一个Region可能之前是年轻代，如果Region进行了垃圾回收，之后可能又会变成老年代，也就是 说Region的区域功能可能会动态变化。

​	专门的Region **Humongous**区，用于存储大对象，大对象 > region50%，直接将对象存到Humongous区，如果很大，会存储多个连续的Humougous。降低老年代开销。

​	**G1 收集过程**

G1回收不管是老年代还是年轻代，都采用 **复制算法**，将存活对象直接复制到其他Region中。不存在空间碎片的问题。

- 初始标记（initial mark，STW）：暂停所有的其他线程，并记录下gc roots直接能引用 的对象，速度很快
- 并发标记（Concurrent Marking）：同CMS的并发标记
- 最终标记（Remark，STW）：同CMS的重新标记
- 筛选回收（Cleanup，STW）：筛选回收阶段首先对各个Region的回收价值和成本进行 排序，根据用户所期望的GC停顿时间(可以用JVM参数 -XX:MaxGCPauseMillis指定)来制 定回收计划（G1会计算回收多少个Region使用时间，尽可能的在指定时间内回收尽可能多的Region）。**G1收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字Garbage-First的由来)，比如一个Region花200ms能回收10M垃 圾，另外一个Region花50ms能回收20M垃圾，在回收时间有限情况下，G1当然会优先选择后面 这个Region回收。这种使用Region划分内存空间以及有优先级的区域回收方式，保证了G1收集 器在有限时间内可以尽可能高的收集效率。**

![image-20200411213243397](/image-20200411213243397.png)

## 调优思路

![image-20200411195535030](/image-20200411195535030.png)

![img](/640.webp)

1. 常用8G内存，分4G给JVM，堆3G，年轻代1.5G，线程栈1M，非堆内存256M（-XX:PermSize设置非堆内存初始值，默认是物理内存的1/64），surviorRate 伊甸园区/survior = 8

   `‐Xms3072M ‐Xmx3072M ‐Xmn1536M ‐Xss1M ‐XX:PermSize=256M ‐XX:MaxPermSize=256M ‐XX:SurvivorRat io=8`

2. 计算程序中执行会产生多大的对象，然后乘以每秒访问量，计算出每秒会生成多少对象在jvm中。

3. Eden区 大小/每秒对象 ，计算出多久可以装满Eden区。1.5G/60M ≈ 20S， 接口一般很多就会执行完毕，则20秒内，18秒左右的数据都会是垃圾数据，在minorGC时就会删掉，剩余1 2秒的对象，则会被转移到survivor区

   ![image-20200411204313507](/image-20200411204313507.png)

4. 考虑1~2秒的对象（约等于100M）什么时候进入到老年代。

   - 100M 》 150M的二分之，根据**动态年龄计算**，这批对象会被转移到老年代。**所以调整年轻代的大小，或者调整Eden和Survivor的比例**

5. 调整年轻代大小 改位2G

   ```shell
   ‐Xms3072M ‐Xmx3072M ‐Xmn2048M ‐Xss1M ‐XX:PermSize=256M ‐XX:MaxPermSize=256M ‐XX:SurvivorRatio=8
   ```

   ![image-20200411204715044](/image-20200411204715044.png)

6. 计算得出每次minorgc的时间20s，并且很快都会编程垃圾对象，没必要把那些存活较久的对象被回收15次才转移到老年代，可以设置参数`-XX:MaxTenuringThreshold`为5，对象被minorgc 5次以后大概5*20s约等于一分多，被转移到老年代。
7. 结合你自己系统 看下有没有什么大对象生成，预估下大对象的大小，一般来说设置为` - XX:PretenureSizeThreshold`(在Serial和ParNew收集器中起作用)1M就差不多了，很少有超过 1M的大对象，这些对象一般就是你系统初始化分配的缓存对象，比如大的缓存List，Map之类的 对象

```shell
#堆3G，年轻代2G，栈1M，非堆256M，年轻代比例8，对象在年轻代的最大年龄为5，大对象为1M（大于1M直接到老年代），年轻代使用ParNew，老年代使用CMS
‐Xms3072M ‐Xmx3072M ‐Xmn2048M ‐Xss1M ‐XX:PermSize=256M ‐XX:MaxPermSize=256M ‐XX:SurvivorRa
tio=8 ‐XX:MaxTenuringThreshold=5 ‐XX:PretenureSizeThreshold=1M ‐XX:+UseParNewGC ‐XX:+UseConcMarkSweepGC
```

8. 考虑CMS的配置
   - 碎片清理，如果年轻代配置合理，很久才会执行一次fullgc，所以，考虑每次fullgc之后整理空间`XX:+UseCMSCompactAtFullCollection ‐XX:CMSFullGCsBeforeCompaction=0`
   - CMS默认空间占用92之后，FULLGC`‐XX:CMSInitiatingOccupancyFaction=92`

```shell
‐XX:CMSInitiatingOccupancyFaction=92 ‐XX:+UseCMSCompactAtFullCollection ‐XX:CMSFullGCsBeforeCompaction=0
```

## 调优实战

### 常用命令

jmap

- -histo pid 查看堆内存信息，可以看到实例个数记忆占用内存大小
- -dump:format=b,file=xxx pid 可以将信息导出dump文件，可由jvisualvm加载查看。软件启动的时候可以加参数，当oom的时候导出dump`‐XX:+HeapDumpOnOutOfMemoryError ‐XX:HeapDumpPath=D:\jvm.dump`
- 

jstack

jinfo

导出dump文件，

**jstat**