GC:
https://community.aijishu.com/a/1060000000083459
#
https://segmentfault.com/a/1190000004233812
#
https://tech.meituan.com/2016/09/23/g1.html
#
https://tech.meituan.com/2020/11/12/java-9-cms-gc.html
#
https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html

##CMS痛点
##1. 内存碎片与“Full GC”风险（最致命缺陷）痛点原理：CMS 采用 Mark-Sweep（标记-清除） 算法，回收后不移动存活对象，会导致老年代产生大量不连续的内存碎片。当大对象（如风控大报文、长视频切片元数据）进入老年代时，虽然总剩余空间足够，但没有一块连续空间能装下，从而触发 Serial Old 收集器介入进行单线程的压缩清理，引发长达数秒甚至数分钟的灾难性 Full GC。

#实干调优参数：
-XX:+UseCMSCompactAtFullCollection：默认开启。在完成垃圾收集后，或者触发 Full GC 时，强制进行一次内存碎片整理（压缩），此过程需要 STW。
-XX:CMSFullGCsBeforeCompaction=N：设置执行多少次不压缩的 CMS 垃圾回收后，跟着来一次带碎片压缩的回收（通常设为 0，代表每次进入 Full GC 都压缩）。

##2. 浮动垃圾（Floating Garbage）与“Concurrent Mode Failure”痛点原理：在 4. 并发清除 阶段，用户线程还在继续运行，期间产生的新垃圾（称为“浮动垃圾”）在当前回收周期内无法被标记和清理，只能留到下一次 GC。这就要求老年代不能等满了才回收，必须预留一部分空间给并发运行的用户线程。如果预留空间不够，用户线程新产生的对象放不下，就会触发 Concurrent Mode Failure，同样会退化为 Serial Old 引发漫长的 STW。

#实干调优参数：-XX:CMSInitiatingOccupancyFraction=N：设置老年代内存达到百分之多少时触发 CMS 回收（默认 68% 或 92% 左右）。调优思路：如果系统大对象多、浮动垃圾多，应该调低该值（如 65%-70%），让 CMS 提早开始工作，预留更多空间给用户线程，防止并发失败。

##3. 抢占 CPU 资源（对 CPU 核心数敏感）

痛点原理：在并发标记和并发清除阶段，CMS 虽然没有让用户线程停顿，但它会占用一部分 CPU 线程去干垃圾回收的工作，导致高并发业务的吞吐量下降。CMS 默认启动的回收线程数是 (CPU核心数 + 3) / 4。影响：当 CPU 核心数小于 4 个时（如早期的双核虚拟机），CMS 会分走 25% 以上的算力，对业务性能影响极大。
