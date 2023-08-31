## top

%us：表示用户空间程序的cpu使用率（没有通过nice调度）

%sy：表示系统空间的cpu使用率，主要是内核程序。

%ni：表示用户空间且通过nice调度过的程序的cpu使用率。（用户进程空间内改变过优先级的进程占用CPU百分比）

%id：空闲cpu

%wa：cpu运行时在等待io的时间 

//**iowait时间就是CPU idle时间，但是这时候CPU上不是完全没TASK需要运行，而是休眠的task中有一个或者若干个是iowait的task**。

%hi：cpu处理硬中断的数量

%si：cpu处理软中断的数量

%st：被虚拟机偷走的cpu



### 监控每个逻辑CPU的状况

按 1 查看所有cpu核状态



### 增加监控参数

按 f 新增查看选项

​	右键选定新增选项，按上键移动

shift+w保存配置，下次运行top就不用重新配置



### 进程按指定列高亮显示

在top视图中，我们可以按b打开或关闭加亮效果

在打开加亮的效果之后，我们可以按x键实现列的加亮效果，同时可以按”shift+>”或者”shift+<”左右改变排序序列。

https://www.cnblogs.com/ajianbeyourself/p/8185973.html





-d 改变显示的更新速度，每s更新一次

-p 显示给定进程的信息

-n 指定更新次数后推出

-H 抓取线程

top -b -d 1 -n 2



eg：

iowait超过80%，说明一直在等待io，pidstat抓取数据看下是哪个pid进程导致了读写量比较大

```shell
while true;do
	iowait=$(top -H -b -d 1 -n 1 | grep %Cpu | grep wa | awk -F "," '{print $5}' | awk -F " " '{print $1}') | awk -F "." '{print $1}'
	if [[ $iowait -gt 80 ]];then
		echo "exceed iowait 80%"
		pidstat -d 1 100 > pidstat.txt &
	fi
done
```





top %mem

res是常驻内存

这个值就是该应用程序真的使用的内存

res除以物理内存总量



物理内存：

1、top

MiB Mem的total

2、free -g

Mem的total

3、/proc/meminfo

MemTotal
