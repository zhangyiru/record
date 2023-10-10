在我们开始使用 Perfetto 之前，有个配置要注意下，Perfetto 是基于 Android 的系统追踪服务， 这个配置在 Android11(R) 之后是默认打开的，但是如果你是 Android 9 (P) 或者 10 (Q) ，那么就需要手动设置一下相应的 prop 属性



![image-20230922164832488](/Users/xayahion/Library/Application Support/typora-user-images/image-20230922164832488.png)



### 抓取trace

1、通过UI网页获取config命令，贴到终端使用

![image-20230922174853625](/Users/xayahion/Library/Application Support/typora-user-images/image-20230922174853625.png)

```shell
#将配置保存为文件传入到设备中
adb push my_test.config /data/local/tmp/my_test.config

#抓取trace，文件保存到trace.perfetto-trace
adb shell "cat /data/local/tmp/my_test.config | perfetto --txt -c - -o /data/misc/perfetto-traces/trace.perfetto-trace"

#trace文件传输到本地，使用UI网页进行解析
adb pull /data/misc/perfetto-traces/trace.perfetto-trace  ./
```



### 选择trace文件进行解析

![image-20230922175144612](/Users/xayahion/Library/Application Support/typora-user-images/image-20230922175144612.png)





### 分析native物理内存泄漏

https://www.jianshu.com/p/f25539c80768

1.点击Record new trace -》Memory -》Native Heap Profiling，并填入你想跟踪的进程名或进程id

2.点击 Recording Settings -》Long trace -》Max duration 调整合适的抓取时间，这里设置为1分钟。

3.点击recording command，复制生成的命令，在终端执行

4.重复复现泄露路径

5.导出trace

adb pull /data/misc/perfetto-traces/trace

6.点击Open trace file 分析trace



参考链接：

1、Android 系统使用 Perfetto 抓取 trace 教程

https://zhuanlan.zhihu.com/p/508526020

2、perfetto使用

https://blog.csdn.net/mingxinjianx/article/details/131422978