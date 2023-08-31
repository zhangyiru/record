## -f

跟踪子进程，因为它们是由当前跟踪的进程创建的，作为fork（2）、vfork（1）和clone（2）系统调用的结果。请注意，如果进程PID是多线程的，则-p PID-f将附加进程PID的所有线程，而不仅仅是thread_id＝PID的线程。



## 精确时间
strace -T -tt -o log xxx



## 查看指定系统调用的耗时

strace trace=read -o log xxx



## 内存泄漏检测

strace -emmap,brk,munmap xxx
看到只有mmap，没有munmap



