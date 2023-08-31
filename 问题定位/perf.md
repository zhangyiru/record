##  perf-sched - Tool to trace/measure scheduler properties

```
perf sched record -- sleep 1
perf sched timehist
```

默认情况下，它会显示单独的调度事件，包括任务的等待时间(在任务的 sched-out 和下一个 sched-in 事件之间的时间)、任务调度延迟(在唤醒和实际运行之间的时间)和运行时间:

![image-20220701095540616](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20220701095540616.png)

https://man7.org/linux/man-pages/man1/perf-sched.1.html



## 抓指定cpu

```shell
perf record -g -C xxx -- sleep 10
perf report --no-ch
```



## 火焰图

git clone https://github.com/brendangregg/FlameGraph.git


1、用perf script工具对perf.data进行解析

perf script -i perf.data &> perf.unfold



2、将perf.unfold中的符号进行折叠：

/stackcollapse-perf.pl perf.unfold &> perf.folded


3、最后生成svg图：

./flamegraph.pl perf.folded > perf.svg
