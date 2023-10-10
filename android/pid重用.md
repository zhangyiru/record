### 痛点：PID会被重复回收利用

当某个进程退出的时候，它释放的PID可能会在一段时间后会重新分配给其他新进程使用。

这就会造成一种竞争问题(称为race-free process signaling)

例如：

进程server本来想通过发送信号kill进程A，但是发送信号前进程A退出了，且进程A的PID号很快就被重复分配给了另外一个进程B，此时server就会错把信号发送给了B，严重的话会导致B进程被迫退出。



针对这个问题Linux内核在Linux-5.1引入了一个新系统调用pidfd_send_signal(2)以通过操作/proc/<pid>文件描述符的方式来解决该问题



### pidfd基本原理

pidfd本质上是文件描述符，但是它在形式上没有一个对应的实际可见的文件路径(dentry)，而是采用**anon_inode(匿名节点)**方式来实现。

与PID(process ID)的实现不同，pidfd是通过专有的pidfd open函数"打开"目标进程dst task，然后返回dst task对应的文件描述符pidfd

在当前进程未主动调用close()函数关闭pidfd前不论dst task是否退出，pidfd对于当前任务都是有效的，这就避免了PID"race-free process signaling"的问题。



### 参考链接：

1、pidfd -- 一种基于进程ID的文件描述符

https://zhuanlan.zhihu.com/p/381302990