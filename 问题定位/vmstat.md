https://www.cnblogs.com/zsql/p/11643750.html#_label1_3

### 1、procs

r  #表示运行队列(就是说多少个进程真的分配到CPU)，当这个值超过了CPU数目，就会出现CPU瓶颈了。这个也和top的负载有关系，一般负载超过了3就比较高，超过了5就高，超过了10就不正常了，服务器的状态很危险。top的负载类似每秒的运行队列。如果运行队列过大，表示你的CPU很繁忙，一般会造成CPU使用率很高。

b  #表示阻塞的进程,在等待资源的进程，这个不多说，进程阻塞，大家懂的。



### 2、memory

swpd #虚拟内存已使用的大小，如果大于0，表示你的机器物理内存不足了，如果不是程序内存泄露的原因，那么你该升级内存了或者把耗内存的任务迁移到其他机器。

free  # 空闲的物理内存的大小

buff  #Linux/Unix系统是用来存储，目录里面有什么内容，权限等的缓存

cache #cache直接用来记忆我们打开的文件,给文件做缓冲，把空闲的物理内存的一部分拿来做文件和目录的缓存，是为了提高 程序执行的性能，当程序使用内存时，buffer/cached会很快地被使用。



### 3、swap

si  #每秒从磁盘读入虚拟内存的大小，如果这个值大于0，表示物理内存不够用或者内存泄露了，要查找耗内存进程解决掉。我的机器内存充裕，一切正常。

so #每秒虚拟内存写入磁盘的大小，如果这个值大于0，同上。



### 4、io

bi  #块设备每秒接收的块数量，这里的块设备是指系统上所有的磁盘和其他块设备，默认块大小是1024byte

bo #块设备每秒发送的块数量，例如我们读取文件，bo就要大于0。bi和bo一般都要接近0，不然就是IO过于频繁，需要调整。



### 5、system

in  #每秒CPU的中断次数，包括时间中断

cs  #每秒上下文切换次数，例如我们调用系统函数，就要进行上下文切换，线程的切换，也要进程上下文切换，这个值要越小越好，太大了，要考虑调低线程或者进程的数目



### 6、cpu

us  #用户CPU时间，我曾经在一个做加密解密很频繁的服务器上，可以看到us接近100,r运行队列达到80(机器在做压力测试，性能表现不佳)。

sy  #系统CPU时间，如果太高，表示系统调用时间长，例如是IO操作频繁。

id  #空闲 CPU时间，一般来说，id + us + sy = 100,一般我认为id是空闲CPU使用率，us是用户CPU使用率，sy是系统CPU使用率。

wt  #等待IO CPU时间。
