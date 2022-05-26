# 垃圾回收器

## 年轻代

### Serial

单线程，简单高效，没有线程交互的开销。 但是用户线程必须等待。 新生代使用标记复制算法

### ParNew

其是相当于Serial的多线程版本，可以与CMS配合使用。 使用标记复制算法

### Parallel Scavenge

与吞吐量密切相关，也称为吞吐量优先收集器。 新生代采用标记复制算法，并行的多线程收集器。 目标：达到一个可控制的吞吐量。 GC自适应调节策略： -XX:+UseAdptiveSizePolicy 动态设置新生代的大小(-Xnn)，Eden和Survivor的比例（-XX:SurvivorRation）,晋升老年代的对象年龄 （-XX:PretenureSizeThreshold）

通过两个参数控制： XX:MaxGCPauseMillis控制最大的垃圾收集时间 XX:GCRatio 直接设置吞吐量大小

## 老年代

### Serial Old

单线程收集，老年代使用标记整理法

### Parallel Old

老年代使用标记整理法，多线程，高吞吐量和CPU资源敏感的场景

## 混合

### CMS

追求最短停顿时间为目标的收集器。 对年轻代使用标记复制算法，年老代使用标记清除算法 因为采用的清除算法，因此会导致内存碎片化，只能通过fullGC处理 初始标记阶段：只标记关联GCroot的对象，不用向下追溯。 并发标记：标记所有可达的对象。在这个过程中与用户线程并行，因此可能会产生很多变化 重新标记：为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在Stop The World问题。 并发清除：会有部分浮动垃圾

### G1

#### 目标

G1：被称为软实时的垃圾回收器。所谓软实时，是指在要求的时间内（需要合理），可以让90%以上的垃圾回收时间在这个范围内。

#### 结构

G1将新生代、老年代的物理空间划分取消了，但是保留了其逻辑概念。目前其结构为若干个Region，但是有逻辑上的Eden Region，Survivor Region，Old Region，Humongous Region(存放短期的巨型对象)。

#### 对象分配策略

4个阶段：

TLAB 线程本地分配缓冲区

Eden区分配

Humongous区分配

Old区分配

#### Remembered Set

使用point-in，标记哪些新生代引用了老年代。

#### 卡表（Card Table）

引用对象太多时，使用卡表解决。

如果一个线程修改了Region内部的引用，就必须要去通知RS，更改其中的记录。为了达到这种目的，G1回收器引入了一种新的结构，CT(Card Table)——卡表。每一个Region，又被分成了固定大小的若干张卡(Card)。每一张卡，都用一个Byte来记录是否修改过。卡表即这些byte的集合。实际上，如果把RS理解成一个概念模型，那么CT就可以说是RS的一种实现方式。

Young GC 阶段：

* 阶段1：根扫描 静态和本地对象被扫描
* 阶段2：更新RS 处理dirty card队列更新RS
* 阶段3：处理RS 检测从年轻代指向年老代的对象
* 阶段4：对象拷贝 拷贝存活的对象到survivor/old区域
* 阶段5：处理引用队列 软引用，弱引用，虚引用处理

Mix GC不仅进行正常的新生代垃圾收集，同时也回收部分后台扫描线程标记的老年代分区。

它的GC步骤分2步：

1. 全局并发标记（global concurrent marking）
2. 拷贝存活对象（evacuation）

在进行Mix GC之前，会先进行global concurrent marking（全局并发标记）。 global concurrent marking的执行过程是怎样的呢？

在G1 GC中，它主要是为Mixed GC提供标记服务的，并不是一次GC过程的一个必须环节。global concurrent marking的执行过程分为五个步骤：

* 初始标记（initial mark，STW） 在此阶段，G1 GC 对根进行标记。该阶段与常规的 (STW) 年轻代垃圾回收密切相关。
* 根区域扫描（root region scan） G1 GC 在初始标记的存活区扫描对老年代的引用，并标记被引用的对象。该阶段与应用程序（非 STW）同时运行，并且只有完成该阶段后，才能开始下一次 STW 年轻代垃圾回收。
* 并发标记（Concurrent Marking） G1 GC 在整个堆中查找可访问的（存活的）对象。该阶段与应用程序同时运行，可以被 STW 年轻代垃圾回收中断
* 最终标记（Remark，STW） 该阶段是 STW 回收，帮助完成标记周期。G1 GC 清空 SATB 缓冲区，跟踪未被访问的存活对象，并执行引用处理。
* 清除垃圾（Cleanup，STW） 在这个最后阶段，G1 GC 执行统计和 RSet 净化的 STW 操作。在统计期间，G1 GC 会识别完全空闲的区域和可供进行混合垃圾回收的区域。清理阶段在将空白区域重置并返回到空闲列表时为部分并发， 对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划 。

三色标记法
