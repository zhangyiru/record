# 一、优缺点



## tcmalloc

优点：

1、当线程少于16时进行malloc/free时所使用的时钟周期小于其他malloc算法；

2、适合小内存的分配

3、垃圾回收机制

4、最大限度优化了锁的争用

5、很多系统都可以用源来安装 TCMalloc ，支持的 gcc 编译库比较新



缺点：

1、当线程大于16时进行malloc/free时所使用的时钟周期大于其他malloc算法；

2、线程数大于1后，tcmalloc比jemalloc，ptmalloc内存开销大



## jemalloc

优点：

1、适用于多线程环境

2、当线程数量固定，不会频繁创建退出时，优先选择jemalloc

3、目前是 Maridab 、Tengine、Redis 中默认推荐的内存优化工具



缺点：

1、某个线程在这个 arena使用了很多内存，之后这个 arena并没有其他线程使用，导致这个 arena的内存无法被 gc，占用过多

2、两个位于不同 arena的线程频繁进行内存申请，导致两个 arena的内存出现大量交叉，但是连续的内存由于在不同 arena而无法进行合并

3、对使用最新的gcc编译不友好。



## ptmalloc2

缺点：

1、ptmalloc 不适合用于管理长生命周期的内存，特别是持续不定期分配和释放长生命周期的内存，这将导致 ptmalloc 内存暴增。

2、不适合多线程程序

3、容易内存泄漏



# 二、16 核 AMD 5950x (Zen3) 的基准测试结果

![image-20210914222740711](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210914222740711.png)

横轴表示各种基准测试，纵轴表示消耗时间。

tcmalloc(紫色)，jemalloc(黄色)，ptmalloc(深绿)

1、redis：tcmalloc>jemalloc>ptmalloc

2、larsonN服务器基准测试在线程之间分配和释放：tcmalloc>jemalloc>ptmalloc

3、mstressN工作负载执行许多分配和重新分配：tcmalloc>jemalloc>ptmalloc

4、rptestN尝试模拟真实的多线程分配模式：jemalloc>tcmalloc>ptmalloc

![image-20210914222948167](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210914222948167.png)



32个逻辑核上并行跑N次

alloc-test是一个非常密集的分配基准测试，在不同大小的类别中执行数百万个分配。

sh6bench和sh8bench基准作为SmartHeap（包括一个类似malloc的API来有效地分配block(小到4字节)的内存，以及一个类似free的API来释放单个block）的一部分

xmalloc-testN基准测试模拟了非对称工作负载，其中一些线程仅分配，而其他线程仅空闲——他们在较大的服务器应用程序中观察到这种模式

1、alloc-testN：tcmalloc>jemalloc>ptmalloc

2、sh6benchN：tcmalloc>jemalloc>ptmalloc

3、sh8benchN：jemalloc>tcmalloc>ptmalloc

4、xmalloc-testN：ptmalloc>jemalloc>tcmalloc



# 三、在 36 核 Intel Xeon 上

![image-20210915000316507](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210915000316507.png)

1、redis：tcmalloc>jemalloc>ptmalloc

2、larsonN服务器基准测试在线程之间分配和释放：tcmalloc>jemalloc>ptmalloc

3、mstressN工作负载执行许多分配和重新分配：tcmalloc>jemalloc>ptmalloc

4、rptestN尝试模拟真实的多线程分配模式：jemalloc>tcmalloc>ptmalloc

和16核表现类似



1、alloc-testN：jemalloc>tcmalloc>ptmalloc 与16核不同

2、sh6benchN：tcmalloc>jemalloc>ptmalloc

3、sh8benchN：jemalloc>tcmalloc>ptmalloc

4、xmalloc-testN：ptmalloc>jemalloc>tcmalloc



# 三、业务场景：



## MySQL

场景：处理这些查询将涉及大量的malloc()/free()操作

分配器策略：

（1）当小于8个core时，glibc malloc和其他的分配器没有区别；

（2）当超过8个core时，推荐选择jemalloc或者tcmalloc来作为内存分配器，将显著地提高MySQL服务器高吞吐量

![image-20210914230815752](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210914230815752.png)



## ceph

场景：每秒输入输出次数（IOPS）

1、相比于tcmalloc，使用 jemalloc使得IOPS性能提高了4.7倍。

2、在小 IO 工作负载下，对于 4KB 随机写入，jemalloc 大约比 tcmalloc 2.1 快 4.21 倍；虽然 jemalloc 在所有测试的配置中产生了最好的 4KB 随机写入性能，但它也消耗了最多的内存。



参考链接：

1、MySQL performance: Impact of memory allocators

https://www.percona.com/blog/2013/03/08/mysql-performance-impact-of-memory-allocators-part-2/

2、jemalloc

https://uncp.github.io/JeMalloc/

3、深入研究glibc内存管理器原理及优缺点

https://blog.csdn.net/whbing1471/article/details/111769751

4、内存管理库TCMalloc和Jemalloc比较

http://blog.onecodeall.com/index/blog/detail/id/1410269142350456/tid/0

5、The Ceph and TCMalloc performance story

https://ceph.com/geen-categorie/the-ceph-and-tcmalloc-performance-story/

6、mimalloc github

https://github.com/microsoft/mimalloc