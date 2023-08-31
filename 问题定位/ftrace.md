# ftrace

## 函数跟踪程序

```sh
#!/bin/sh

dir=/sys/kernel/debug/tracing

sysctl kernel.ftrace_enabled=1
echo function > ${dir}/current_tracer
echo 1 > ${dir}/tracing_on
sleep 1
echo 0 > ${dir}/tracing_on
less ${dir}/trace
```

![image-20220629093057329](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20220629093057329.png)

- 进程标识符（TASK-PID）
- 运行这个进程的 CPU（CPU#）
- 进程开始时间（TIMESTAMP）
- 被跟踪函数的名字以及调用它的父级函数（FUNCTION）；例如，在我们输出的第一行，`rb_simple_write` 调用了 `mutex-unlock` 函数。





## function_graph 跟踪程序

和function跟踪程序类似，它会显示了每个函数的进入和退出点


echo function_graph > /sys/kernel/debug/tracing/current_tracer

![image-20220629093311469](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20220629093311469.png)



DURATION 展示了花费在每个运行的函数上的时间。

```
  + means that the function exceeded 10 usecs.
  ! means that the function exceeded 100 usecs.
  # means that the function exceeded 1000 usecs.
  * means that the function exceeded 10 msecs.
  @ means that the function exceeded 100 msecs.
  $ means that the function exceeded 1 sec
```

在 FUNCTION_CALLS 下面，我们可以看到每个函数调用的信息



## 函数过滤器

使用过滤器去简化我们的搜索，实现过滤，我们只需要在 set_ftrace_filter文件中写入我们需要过滤的函数的名字



禁用过滤器，只需要将此文件设为空

echo > set_ftrace_filter



## 跟踪事件

当相关代码片断运行时，动态跟踪点调用一个跟踪函数。跟踪数据是写入到 Ring 缓冲区

跟踪点可以设置在代码的任何位置；事实上，它们确实可以在许多的内核函数中找到，如kmem_cache_alloc函数



```c
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
	void *ret = slab_alloc(cachep, flags, _RET_IP_);

	kasan_slab_alloc(cachep, ret, flags);
	trace_kmem_cache_alloc(_RET_IP_, ret,
			       cachep->object_size, cachep->size, flags);
	
	return ret;

}
```



trace_kmem_cache_alloc 它本身就是一个跟踪点

在 Linux 内核中为了从用户空间使用跟踪点，它有一个专门的 API。在 /sys/kernel/debug/tracing 目录中有事件目录，为了保存系统事件。

在跟踪事件前，要开启Ring缓冲区写入（echo 1 > /sys/kernel/debug/tracing/tracing_on）

![image-20220629095940054](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20220629095940054.png)

通过availble_events查看事件列表

![image-20220629095739878](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20220629095739878.png)



### 事件使用

进入/sys/kernel/debug/tracing目录

1、echo 1 > tracing_on

2、echo 1 > events/syscalls/sys_enter_chroot/enable

3、chroot操作

4、echo 0 > tracing_on

5、查看trace



## 参考链接

1、ftrace文档

https://www.kernel.org/doc/Documentation/trace/ftrace.txt

2、使用ftrace跟踪内核

https://zhuanlan.zhihu.com/p/39788032
