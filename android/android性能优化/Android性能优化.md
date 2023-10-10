### 1.内存缓存

#### （1）没有缓存的弊端

**流量开销**：对于客户端——服务器端应用，从远程获取图片算是经常要用的一个功能，而图片资源往往会消耗比较大的流量。
**加载速度**：如果应用中图片加载速度很慢的话，那么用户体验会非常糟糕。
那么如何处理好图片资源的获取和管理呢？**异步下载+本地缓存**

#### （2）缓存带来的好处

1. 服务器的压力大大减小；
2. 客户端的响应速度大大变快(用户体验好)；
3. 客户端的数据加载出错情况大大较少，大大提高了应有的稳定性(用户体验好)；
4. **一定程度上可以支持离线浏览**(或者说为离线浏览提供了技术支持)

#### （3）缓存管理的应用场景：

1. 提供网络服务的应用；
2. 数据更新不需要实时更新，**即便是允许3-5分钟的延迟也建议采用缓存机制；**

#### （4）大位图导致内存开销大的原因是什么？

1. 下载或加载的过程中容易导致阻塞；
2. 大位图Bitmap对象是png格式的图片的30至100倍；
3. 大位图在加载到ImageView控件前的解码过程；BitmapFactory.decodeFile()会有内存消耗。

#### （5）缓存设计的要点：

1. 命中率；
2. 合理分配占用的空间；
3. 合理的缓存层级。

#### （6）加载图片的正确流程

**“内存-文件-网络 三层cache策略”**

1. 先从内存缓存中获取，取到则返回，取不到则进行下一步；
2. 从文件缓存中获取，取到则返回并更新到内存缓存，取不到则进行下一步；
3. 从网络下载图片，并更新到内存缓存和文件缓存

#### （7）内存缓存分类

**从JDK1.2版本开始，把对象的引用分为四种级别，从而使程序能更加灵活的控制对象的生命周期。**
**这四种级别由高到低依次为：强引用、软引用、弱引用和虚引用。**

1、强引用：（在Android中**LruCache**就是强引用缓存）
平时我们编程的时候例如：Object object=new Object（）；那object就是一个强引用了。如果一个对象具有强引用，那就类似于必不可少的生活用品，垃圾回收器绝不会回收它。当内存空间不足，**Java虚拟机宁愿抛出OOM异常，使程序异常终止，也不会回收**具有强引用的对象来解决内存不足问题。

2、软引用（SoftReference）：
软引用类似于可有可无的生活用品。**如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存**。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。 软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。
使用软引用能防止内存泄露，增强程序的健壮性。

3、弱引用（WeakReference）：
弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。**在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存**。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

4、虚引用（PhantomReference）
"虚引用"顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，**在任何时候都可能被垃圾回收**。 虚引用主要用来跟踪对象被垃圾回收的活动。
虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

#### （8）缓存数据保存

在内存中保存的话，只能保存一定的量，而不能一直往里面放，需要设置数据的过期时间、LRU等算法。

这里有一个方法是把常用的数据放到一个缓存中（A），不常用的放到另外一个缓存中（B）。

当要获取数据时先从A中去获取，如果A中不存在那么再去B中获取。B中的数据主要是A中LRU出来的数据，这里的内存回收主要针对B内存，从而保持A中的数据可以有效的被命中。

#### （9）本地缓存

不可能每次获取让让应用从远程下载图片（浪费资源），又不能把所有图片资源放到内存中（占用大量空间导致oom）；

故将下载下来的图片保存到sdcard中，下次直接从sdcard中取



1. 在sdcard上开辟一定的空间，需要先判断sdcard上剩余空间是否足够，如果足够的话就可以开辟一些空间，比如10M
2. 当需要获取图片时，就先从sdcard上的目录中去找，如果找到的话，使用该图片，并**更新图片最后被使用的时间**。如果找不到，通过URL去download服务器端下载图片，如果下载成功了，放入到sdcard上，并使用，如果失败了，应该有**下载重试机制**。比如3次。
3. 下载成功后保存到sdcard上，需要先判断10M空间是否已经用完，如果没有用完就保存，**如果空间不足就根据LRU规则删除一些最近没有被用户的资源**
   



### 2.APK 缓存

https://source.android.google.cn/devices/tech/perf/apk-caching?hl=nl

本文档介绍了如何设计 APK 缓存解决方案，以在支持 A/B 分区的设备上快速安装预加载的应用。

OEM 可以将预加载应用和热门应用放置在 APK 缓存中（对于采用 A/B 分区的新设备而言，这种缓存会存储在通常为空的 B 分区中），而且这样不会影响面向用户的任何数据空间。

APK缓存位于 `/data/preloads/file_cache` 中，布局如下：

```
/data/preloads/file_cache/
    app.package.name.1/
          file1
          fileN
    app.package.name.N/
```

只有特权应用才可以访问预加载缓存目录。如需获得访问权限，应用必须安装在 `/system/priv-app` 目录中



### 3.优化启动时间

##### 启动时间按时间段分布：

![img](https://upload-images.jianshu.io/upload_images/14421843-bf139a85fb9ea3b8.png?imageMogr2/auto-orient/strip|imageView2/2/w/688)



Android 8.0 支持一系列组件的多项改进，因而可以缩短启动时间。本文档提供了有关缩短特定 Android 设备的启动时间的合作伙伴指南。

| 组件         | 改进                                                         |
| :----------- | :----------------------------------------------------------- |
| 引导加载程序 | 通过**移除 UART 日志**节省了 1.6 秒<br />通过从 GZIP 更改为 LZ4 节省了 0.4 秒 |
| 设备内核     | 通过移除不使用的内核配置和减少驱动程序大小节省了 0.3 秒<br />通过 dm-verity 预提取优化节省了 0.3 秒<br />通过移除驱动程序中不必要的等待/测试，节省了 0.15 秒<br />通过移除 CONFIG_CC_OPTIMIZE_FOR_SIZE，节省了 0.12 秒 |
| I/O 调整     | 正常启动时间节省了 2 秒<br />首次启动时间节省了 25 秒        |
| init.*.rc    | 通过并行运行 init 命令节省了 1.5 秒<br />通过及早启动 zygote 节省了 0.25 秒<br />通过 cpuset 调整节省了 0.22 秒 |
| 启动动画     | 在未触发 fsck 的情况下，启动动画的开始时间提前了 2 秒，而触发 fsck 时启动动画则大得多<br />通过立即关闭启动动画在 Pixel XL 上节省了 5 秒 |
| SELinux 政策 | 通过 genfscon 节省了 0.2 秒                                  |



#### （1）优化io效率

对任何不必要内容的读取都应推迟到启动之后再进行



调整文件系统

```shell
on late-fs
  # boot time fs tune
    # boot time fs tune
    write /sys/block/sda/queue/iostats 0
    write /sys/block/sda/queue/scheduler cfq
    
    #当一个进程的队列被分配到时间片却没有 IO 请求时，调度器在轮询至下一个队列之前的等待时间；
    #为提升 IO 的局部性，可以将这个值设为 0。
    write /sys/block/sda/queue/iosched/slice_idle 0
    
    #在大的读I/O流中，增加提前读取缓冲的大小会提升性能。注意，因为大多数都是随机I/O的原因，所以在一般情况下，增加这个值不会增强性能。read_ahead_kb定义了预读多少数据
    write /sys/block/sda/queue/read_ahead_kb 2048
    
    #O调度队列大小
    write /sys/block/sda/queue/nr_requests 256
    
    
    write /sys/block/dm-0/queue/read_ahead_kb 2048
    write /sys/block/dm-1/queue/read_ahead_kb 2048

on property:sys.boot_completed=1
    # end boot time fs tune
    write /sys/block/sda/queue/read_ahead_kb 512
    ...
```



使用以下脚本来帮助分析启动性能。

- `system/extras/boottime_tools/bootanalyze/bootanalyze.py`：负责衡量启动时间，并详细分析启动过程中的重要步骤。
- `system/extras/boottime_tools/io_analysis/check_file_read.py boot_trace`：提供每个文件的访问信息。
- `system/extras/boottime_tools/io_analysis/check_io_trace_all.py boot_trace` 提供系统级细分信息。



#### （2）优化init.*.rc

##### a.并行运行任务**

移除 init.*.rc 中所有未使用的服务和命令。只要是早期阶段的 init 中没有使用的服务和命令，都应推迟到启动完成后再使用



##### b.**使用调度程序调整**

```shell
on init
    # boottime stune 性能优先
    write /dev/stune/schedtune.prefer_idle 1
    write /dev/stune/schedtune.boost 100
    on property:sys.boot_completed=1
    # reset stune 功耗优先
    write /dev/stune/schedtune.prefer_idle 0
    write /dev/stune/schedtune.boost 0

    # or just disable EAS during boot
    on init
    write /sys/kernel/debug/sched_features NO_ENERGY_AWARE
    on property:sys.boot_completed=1
    write /sys/kernel/debug/sched_features ENERGY_AWARE
```



##### **这里就提到sched-tune的概念**

schedtune提供了一套用户接口的工具，用于功耗-性能调节。schedtune是cgroup的一个子系统。所以在cgroup的mount节点下，stune分别为每个group，都提供了2个调节开关：

```shell
#/dev/stune/<stune cgroup mount point>/schedtune.prefer_idle
#/dev/stune/<stune cgroup mount point>/schedtune.boost
```

![image-20230921144300110](/Users/xayahion/Library/Application Support/typora-user-images/image-20230921144300110.png)

boost的值用int型表示，范围为[0, 100]。

boost默认值为0，代表CFS调度器会工作在能耗最低的状态。这也意味着schedutil使task跑在最低的OPP（工作频率点operation point）。

boost值100，则表示调度器为工作在性能最高的状态，同时OPP也处在最大。

0-100的范围可以根据其他场景来进行适当调节。比如，优化交互的响应、电池电量变化等。



prefer_idle是一个控制调度器节省功耗优先，还是性能优先的flag。

默认值0，会让CFS调度器根据energy-aware wakeup策略来分配在group中的task。（功耗优先）

当值设为1，会让CFS调度器分配task时，有最小的wakeup延迟。（性能优先）

android平台下使用这个flag用来表示正在和用户交互的应用。



##### 这里就提到EAS的概念

能量感知调度（Energy Aware Scheduling，简称EAS）是目前Android手机中Linux线程调度器的基础功能，它**使调度器能预测其决策对CPU能耗的影响**；依靠CPU的能量模型（Energy Model，简称EM），**EAS能为每个线程选择一个最能节约能量的CPU，并把对系统性能的影响降到最低**。



##### c.**及早启动zygote**

在Android系统中，应用程序进程都是由Zygote进程孵化出来的，而Zygote进程是由Init进程启动的。Zygote进程在启动时会创建一个Dalvik虚拟机实例，**每当它孵化一个新的应用程序进程时，都会将这个Dalvik虚拟机实例复制到新的应用程序进程里面去**，从而使得每一个应用程序进程都有一个独立的Dalvik虚拟机实例

部分服务在启动过程中可能需要进行优先级提升

![image-20230921154235340](/Users/xayahion/Library/Application Support/typora-user-images/image-20230921154235340.png)



##### d.停用节电设置

在设备启动期间，可以停用 UFS 和/或 CPU 调节器等组件的节电设置

```shell
on init
    # Disable UFS powersaving
    write /sys/devices/soc/${ro.boot.bootdevice}/clkscale_enable 0
    write /sys/devices/soc/${ro.boot.bootdevice}/clkgate_enable 0
    write /sys/devices/soc/${ro.boot.bootdevice}/hibern8_on_idle_enable 0
    write /sys/module/lpm_levels/parameters/sleep_disabled Y
on property:sys.boot_completed=1
    # Enable UFS powersaving
    write /sys/devices/soc/${ro.boot.bootdevice}/clkscale_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/clkgate_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/hibern8_on_idle_enable 1
    write /sys/module/lpm_levels/parameters/sleep_disabled N
on charger
    # Enable UFS powersaving
    write /sys/devices/soc/${ro.boot.bootdevice}/clkscale_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/clkgate_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/hibern8_on_idle_enable 1
    write /sys/class/typec/port0/port_type sink
    write /sys/module/lpm_levels/parameters/sleep_disabled N
```



##### e.推迟非关键初始化

非关键初始化（如 ZRAM）可以推迟到 boot_complete。

```shell
on property:sys.boot_completed=1
   # Enable ZRAM on boot_complete
   swapon_all /vendor/etc/fstab.${ro.hardware}
```



**这里就提到ZRAM的概念**

zram 风暴是说，以安卓 11 为例，手机中的内存分为两块，一块是正常的内存，另一块是“虚拟内存，但是仍然保存在内存里”。
这在 Windows 被叫做“虚拟内存（页面文件）”，在 Linux 被叫做 swap。
通常这些 swap 被放在硬盘上，压缩后的叫做 zswap，这是因为机械硬盘的随机读写速度很慢，而手机或固态硬盘的存储空间寿命有限，
我不想频繁读写我的固态，所以把**不活跃的内存压缩一下然后还是放在内存里**，这种做法叫做 zram。



#### （3）优化启动动画

配置为及早启动

Android 8.0 支持在装载 userdata 分区之前及早启动动画。然而，即使 Android 8.0 中使用了新的 ext4 工具链，系统也会出于安全原因定期触发 fsck，导致启动 bootanimation 服务时出现延迟。

为了使 bootanimation 及早启动，请将 fstab 装载分为以下两个阶段：

- 在早期阶段，仅装载不需要运行检查的分区（例如 `system/` 和 `vendor/`），然后启动启动动画服务及其依赖项（例如 servicemanager 和 surfaceflinger）。
- 在第二个阶段，装载需要运行检查的分区（例如 `data/`）。

启动动画将会更快速地启动（且启动时间恒定），不受 fsck 影响。



#### （4）工具和方法【待补充使用方法】

bootchart

systrace



### 3.运行状况

Android 9 引入了从 health@1.0 HAL 升级的主要版本 `android.hardware.health` HAL 2.0。这一新版 HAL 使框架代码和供应商代码之间的区别更加清楚，供应商对运行状况信息报告进行自定义的自由度更高，并提供了更多设备运行状况信息（不仅包括电池信息）。



### 4.lowmemorykiller

Android 低内存终止守护程序 (`lmkd`) 进程可监控运行中的 Android 系统的内存状态，并通过终止最不必要的进程来应对内存压力大的问题，使系统以可接受的性能水平运行。



### 5.配置文件引导的优化

Profile-guided optimization (PGO)，配置文件引导的优化，基于插桩或采样从程序运行时生成配置文件，**使编译器对内联和代码布局做优化，可以获得免费的性能提升**。

https://www.jianshu.com/p/98b37707f41d



### 6.任务快照

任务快照是在 Android O（8.0） 中引入的基础架构，可将窗口管理器中的最近任务缩略图和已保存 Surface 这两者的屏幕截图进行合并。最近任务缩略图会在“最近”视图中呈现任务的最后状态。



### 7.预写日志

Android 9 引入了 SQLiteDatabase 的一种特殊模式，称为“兼容性 WAL（预写日志记录）”，它允许数据库使用 `journal_mode=WAL`，同时保留每个数据库最多创建一个连接的行为。



### 8.应用休眠

应用休眠可以让用户连续几个月不使用的应用休眠，与权限自动撤消类似。这会强行停止应用，并将其置于我们针对存储空间而非性能进行优化的状态。

强行停止的应用不会在后台运行作业或发出提醒，也无法发送推送通知。

当用户重新使用该应用时，它会退出休眠状态，并照常运行作业、发出提醒和发送通知。

您需要重新调度在应用进入休眠状态之前调度的所有作业、提醒和通知。



### 9.游戏加载时性能提升

为了缩短游戏的加载时间，Android 13 在 Android 动态性能框架 (ADPF) 中引入了一个名为 [`GAME_LOADING`](https://cs.android.com/android/platform/superproject/+/master:hardware/interfaces//power/aidl/android/hardware/power/Mode.aidl?hl=zh-cn) 的新电源模式。



### 10.内存统计信息

#### （1）启用 mm_events

要启用 mm_events，请从供应商`init.rc`设置 sysprop `persist.mm_events.enabled=true`



#### （2）定制mm_events

`mm_events`使用`perfetto`跟踪配置文件来指定在跟踪会话期间要捕获的统计信息。

您可以在`/vendor/etc/mm_events.cfg`跟踪配置。



#### （3）分析 mm_events 数据

如果启用`mm_events` ，则在设备开始遇到高内存压力后不久捕获的事件的错误报告会以`FS/data/misc/perfetto-traces/bugreport/systrace.pftrace.`中压缩报告的形式提供历史`mm_events`统计信息`FS/data/misc/perfetto-traces/bugreport/systrace.pftrace.`

可以使用[Perfetto UI](https://ui.perfetto.dev/)查看`vmstat`数据和`ftrace`事件以进行分析。



### 11.参考链接

1.Android中的缓存概述

https://blog.csdn.net/lwzhang1101/article/details/78751516

2.Android官网性能优化文档

https://source.android.com/docs/core/perf?hl=zh-cn&bookmark=true

3.schedtune学习笔记

https://www.cnblogs.com/hellokitty2/p/13875073.html

4.Linux EAS介绍

https://zhuanlan.zhihu.com/p/626902632

5.android开机速度优化初探

https://www.jianshu.com/p/ca4b42cb266e

6.后台神话：安卓手机卡顿、杀后台、zram风暴

https://zhuanlan.zhihu.com/p/629640718