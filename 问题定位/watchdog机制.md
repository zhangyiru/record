## kernel watchdog

kernel watchdog是用来检测lockup

lockup，是指某段内核代码占着CPU不放。lockup严重的情况下会导致整个系统失去响应。



lockup有几个特点：

- **首先只有内核代码才能引起lockup**，因为用户代码是可以被抢占的，不可能形成lockup（只有一种情况例外，就是**SCHED_FIFO优先级为99**的实时进程即使在用户态也可能使[watchdog/x]内核线程抢不到CPU而形成soft lockup）
- 其次内核代码**必须处于禁止内核抢占的状态**(preemption disabled)，因为Linux是可抢占式的内核，只在某些特定的代码区才禁止抢占（例如spinlock），在这些代码区才有可能形成lockup。



Lockup分为两种：

soft lockup 和 hard lockup，它们的区别是 hard lockup 发生在CPU屏蔽中断的情况下。

而soft lockup则是单个CPU被一直占用的情况（中断仍然可以响应）。



## NMI，即非可屏蔽中断。

即使在内核代码中设置了屏蔽所有中断的时候，NMI也是不可以被屏蔽的。



中断分为可屏蔽中断和非可屏蔽中断。

其中，可屏蔽中断包含时钟中断，外设中断（比如键盘中断，I/O设备中断，等等），当我们处理中断处理程序的时候，在中断处理程序上半部，在不允许嵌套的情况下，需要关闭中断。

但NMI就不一样了，即便在关闭中断的情况下，他也能被响应。

**触发NMI的条件一般都是ECC error之类的硬件Error**。

但NMI也给我们提供了一种机制，在系统中断被误关闭的情况下，依然能通过中断处理程序来执行一些紧急操作，比如kernel panic。

这里涉及到了3个东西：kernel线程，时钟中断，NMI中断（不可屏蔽中断）。

这3个东西具有不一样的优先级，依次是**kernel线程 < 时钟中断 < NMI中断**。其中，kernel 线程是可以被调度的，同时也是可以被中断随时打断的。



## SoftLockup

Soft lockup是指CPU被内核代码占据，以至于无法执行其它进程。

检测soft lockup的原理是给每个CPU分配一个定时执行的内核线程[watchdog/x]，

如果该线程在设定的期限内没有得到执行的话就意味着发生了soft lockup，

[watchdog/x]是SCHED_FIFO实时进程，优先级为最高的99，拥有优先运行的特权。



系统会有一个高精度的计时器hrtimer（一般来源于APIC），该计时器能定期产生时钟中断，该中断对应的中断处理例程是kernel/watchdog.c: watchdog_timer_fn()，在该例程中： 
\- 要递增计数器hrtimer_interrupts，这个计数器同时为hard lockup detector用于判断CPU是否响应中断； 
\- 还要唤醒[watchdog/x]内核线程，该线程的任务是更新一个时间戳； 
\- soft lock detector检查时间戳，如果超过soft lockup threshold一直未更新，说明[watchdog/x]未得到运行机会，意味着CPU被霸占，也就是发生了soft lockup。

**注意，这里面的内核线程[watchdog/x]的目的是更新时间戳，该时间戳是被watch的对象。而真正的看门狗，则是由时钟中断触发的 watchdog_timer_fn()**，这里面 [watchdog/x]是被scheduler调用执行的，而watchdog_timer_fn()则是被中断触发的。



### 1、watchdog线程定义

```c
static struct smp_hotplug_thread watchdog_threads = {
	.store			= &softlockup_watchdog,
	.thread_should_run	= watchdog_should_run,
	.thread_fn		= watchdog,
	.thread_comm		= "watchdog/%u",
	.setup			= watchdog_enable,
	.cleanup		= watchdog_cleanup,
	.park			= watchdog_disable,
	.unpark			= watchdog_enable,
};
```

### 2、注册watchdog_threads

```c
lockup_detector_setup
	smpboot_register_percpu_thread_cpumask(&watchdog_threads,
						     &watchdog_allowed_mask);
```

### 3、watchdog线程定期调用watchdog

更新watchdog_touch_ts

```c
static void watchdog(unsigned int cpu)
{
	__this_cpu_write(soft_lockup_hrtimer_cnt,
			 __this_cpu_read(hrtimer_interrupts));
	__touch_watchdog();
}
static void __touch_watchdog(void)
{
	__this_cpu_write(watchdog_touch_ts, get_timestamp());
}
```

### 4、时钟中断

```c
watchdog_enable
	watchdog_timer_fn
        //1. 更新hrtimer_interrupts变量
        watchdog_interrupt_count
    	//2. 探测是否有soft lockup发生
		is_softlockup
```

