Linux kernel中执行的代码大体分normal和interrupt context两种。

1、tasklet/softirq可以归为normal因为他们可以进入等待；

2、nested interrupt是interrupt context的一种特殊情况

3、Normal级别可以被interrupt抢断，interrupt会被另一个interrupt抢断，但不会被normal中断

4、各个 interrupt之间没有优先级关系，只要有可能，每个interrupt都会被其他interrupt中断。







## spinlock前需要关中断，免得在中断上下文去获取spinlock导致死锁；

```c
extern spinlock_t lock;
// ...
cli; // disable interrupt on current CPU
spin_lock(&lock);
// do something
spin_unlock(&lock);
sti; // enable interrupt on current CPU
```

https://blog.csdn.net/wesleyluo/article/details/8807919



CLI禁止中断发生
STI允许中断发生

https://blog.csdn.net/zang141588761/article/details/52325106



## spin_lock_irq也可能导致死锁

```c
Code 1                       Code 2

extern spinlock_t lock1;

                             extern spinlock_t lock2;

// ...
spin_lock_irq(&lock1);
// do something 
                             spin_unlock_irq(&lock2);
                             // do something
spin_unlock_irq(&lock1); 
```

问题：在第二个spin_unlock_irq后这个CPU的中断已经被打开，会导致死锁

解决方法：**在每次关闭中断前纪录当前中断的状态，然后恢复它而不是直接把中断打开**

```c
Code 1                       Code 2

extern spinlock_t lock1;
unsigned long flags;
local_irq_save(flags);
                             extern spinlock_t lock2;
							 unsigned long flags;
							 local_irq_save(flags);

spin_lock_irq(&lock1);
// do something 
                             spin_unlock_irq(&lock2);
							 local_irq_restore(flags);
                             // do something
spin_unlock_irq(&lock1); 
local_irq_restore(flags);
```

Linux提供了便捷的方式：

```c
Code 1                       			Code 2

extern spinlock_t lock1;
unsigned long flags;
                             			extern spinlock_t lock2;
							 			unsigned long flags;
							 			spin_lock_irqsave(&lock2, flags);

spin_lock_irqsave(&lock1, flags);
// do something 
                             			spin_unlock_irqrestore(&lock2, flags);
                             			// do something

spin_unlock_irqrestore(&lock1, flags);
```



## spinlock禁止睡眠

```c
spin_lock_A
...
 do something
 lock_mutex_B
   ...
   do something
 unlock_mutex_B
...
spin_unlock_A 
```

如果自旋锁锁住以后进入睡眠，而此时又不能进行处理器抢占(锁住会disable prempt)，其他进程无法获得cpu，这样也不能唤醒睡眠的自旋锁，因此不响应任何操作

```c
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}

#define raw_spin_lock(lock)	_raw_spin_lock(lock)

void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}

static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
	preempt_disable(); //关闭抢占
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```



1. 禁止内核抢占，关闭中断，保存中断状态寄存器的标志位

2. 为何需要关闭内核抢占：

   假如进程A获得spin_lock->进程B抢占进程A->进程B尝试获取spin_lock->由于进程B优先级比进程A高，先于A运行，而进程B又需要A unlock才得以运行，这样死锁。所以这里需要关闭抢占。



## 用户态使用spinlock

https://stackoverflow.com/questions/14723924/using-spinlocks-in-user-space-application


关抢占和关中断在用户态无法做到



## 参考链接

1、那些情况该使用它们spin_lock到spin_lock_irqsave

https://blog.csdn.net/wesleyluo/article/details/8807919
