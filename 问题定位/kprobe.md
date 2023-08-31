https://www.cnblogs.com/arnoldlu/p/9752061.html

## arm64寄存器

AArch64 体系结构支持 32 个整数寄存器：

| 寄存器  | 是否易失？ | 角色                                                         |
| :------ | :--------- | :----------------------------------------------------------- |
| x0      | 易失的     | 参数寄存器/临时寄存器 1、结果寄存器                          |
| x1-x7   | 易失的     | 参数寄存器/临时寄存器 2-8                                    |
| x8-x15  | 易失的     | 临时寄存器                                                   |
| x16-x17 | 易失的     | 过程内部调用临时寄存器                                       |
| x18     | 非易失性的 | 平台寄存器：在内核模式下，指向当前处理器的 KPCR；在用户模式下，指向 TEB |
| x19-x28 | 非易失性的 | 临时寄存器                                                   |
| x29/fp  | 非易失性的 | 帧指针                                                       |
| x30/lr  | 非易失性的 | 链接寄存器                                                   |



## x86寄存器

https://liuhangbin.netlify.app/post/kprobe/

![image-20230112195015362](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20230112195015362.png)



## 增加kprobe事件并使能

```
echo 'p:myprobe do_sys_open dfd=%ax filename=%dx flags=%cx mode=+4($stack)' > /sys/kernel/debug/tracing/kprobe_events
echo 'r:myretprobe do_sys_open ret=$retval' >> /sys/kernel/debug/tracing/kprobe_events
```

```c
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myretprobe/enable
```



## 卸载kprobe事件并disable

```
echo '-:myprobe' >> /sys/kernel/debug/tracing/kprobe_events
echo '-:myretprobe' >> /sys/kernel/debug/tracing/kprobe_events
```

```
echo 0 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable
echo 0 > /sys/kernel/debug/tracing/events/kprobes/myretprobe/enable
```



echo 'p:myprobe virtnet_receive num_free=%ax+0x5'

```
echo 'p:myprobe do_sys_open dfd=%ax filename=%dx flags=%cx mode=+4($stack)' > /sys/kernel/debug/tracing/kprobe_events
echo 'p:myprobe virtnet_receive rq=%ax budget=%dx xdp_xmit=%cx' > /sys/kernel/debug/tracing/kprobe_events


```



## 增加filter

echo dev==eth0  > /sys/kernel/debug/tracing/events/kprobes/p_ip_rcv_0/filter





