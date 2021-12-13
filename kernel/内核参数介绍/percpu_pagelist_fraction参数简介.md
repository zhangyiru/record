一、参数介绍：

/proc/sys/vm/percpu_pagelist_fraction

设置每个zone域中percpu页面分配的最多页面的部分，最小值为8

如将每个zone域中的1/8分配给percpu缓存。



二、PCP分配器：

物理页直接从per_cpu缓存中分配，而不用经过伙伴系统，可以提高分配效率

![image-20210908162512981](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210908162512981.png)

1.count记录了per_cpu缓存中页帧的总数，
2.high记录了per_cpu缓存中页帧的上限，如果超过这个值就将释放 batch个页帧到伙伴系统中去，
3.如果per_cpu中没有可分配的页帧就从伙伴系统中分配batch个页帧到缓存中来。



三、参数取值：

默认值为0：

1、high：表示内核在启动时不使用此值为percpu缓存设置high值；

2、batch: (1024*1024/PAGE_SIZE)/4-1



设置大于8的值：

1、high：manged_pages/percpu_pagelist_fraction

此时percpu_pagelist_fraction设置为100

通过cat /proc/zoneinfo看到high值为managed/100

![image-20210909172657074](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210909172657074.png)



2、batch：min(high/8,PAGE_SHIFT*8)



参数值越大，表示分配给pcp的页面数目越小，high值也就越小

![image-20210909191006167](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210909191006167.png)



https://biscuitos.github.io/blog/HISTORY-PCP/

默认值：

None 0, zone    DMA

high：0

Node 0, zone    DMA32

high：186

Node 0, zone   Normal

high：42

batch=7 7*6=42

sysctl -n vm.percpu_pagelist_fraction改为8后再改为0后的情况：

None 0, zone    DMA

high：0

Node 0, zone    DMA32

high：186

Node 0, zone   Normal

high：186

batch=31 31*6=186