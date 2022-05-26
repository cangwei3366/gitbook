# JVM内存调优工具

#### 1.虚拟机进程状况工具

jps options pid

jps -q #只输出LVMID,省略主类名称

​ jps -m #输出虚拟机进程启动时传递给朱磊main()函数的参数

​ jps -l #输出LVMID以及类全名

​ jps -v #输出虚拟机进程启动时jvm的参数

#### 2.虚拟机统计信息监视工具

jstat -gcutils pid 250 20

表示每个250ms输出进程pid的垃圾收集情况，一共查询20次

options主要分三类：类加载、垃圾收集、运行期编译

\-class 打印类加载、卸载数量，类装载时间

![](../../.gitbook/assets/06\_05\_01.png)

\-gc 监控堆状况，包括s0、s1、老年代、永久代总容量(Capacity)和已使用(USE)、MinorGC、FULL次数以及时间(Time)，当前压缩类空间的容量和已使用容量。

\-gcutil 以百分比格式打印。

\-gccause LGCC 上一次或当前GC原因

远程监控

jstat -gcutil 256380@192.168.2.71

#### 3.配置信息工具

jinfo -falg {+-}name=\[value] pid 查看虚拟机配置信息

#### 4.内存映像工具

jmap用于生成堆转储快照。如果不使用jmap命令，还可以通过-XX：+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在内存溢出异常出现后自动生成堆转储快照。

\-dump生成java堆转储快照

\-finalizerinfo 显示在F-Queue中等待Finalizer线程执行finalize方法的对象

\-heap 显示堆详细信息，如使用那种回收器、参数配置，分代情况等。

\-histo 显示堆中对象统计信息，包括类、实例数量、合计容量。

**jmap -heap 12140 > C:\jvmtest\jmapheap**

jmap -dump:live,format=b,file=heap.hprof 6268

**jstack 12140 > C:\jvmtest\jstack**

jhat分析工具

jhat -J-Xmx1024m file

#### 5.堆转跟踪工具

jstack 生成虚拟机当前时刻的线程快照。生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间的死锁、死循环、请求外部资源导致的长时间挂起等

jstack -l pid

#### 6.远程监控工具

代码异常监控

jmx配置

```
./jstatd -J-Djava.security.policy=/home/intsmaze/jdk1.8.0_144/bin/jstatd-all.policy &
```
