## 内存

### 一、虚拟地址与物理地址

#### 1、分段

分段的弊端：

（1）地址空间不隔离(安全性)：程序A可能会访问到B的地址；

（2）程序运行时地址不确定(动态链接)：访问地址写死的话，动态运行的话地址可能访问不到；

（3）内存使用率低下(内存共享)：会有内存碎片的问题，需要把大块内存换出到磁盘才可以；



虚拟内存中的标准内存布局

![img](https://images0.cnblogs.com/i/569008/201405/270929306664122.jpg)



#### 2、分页

分页的基本方法是将地址空间等分成某一个固定大小的页；

- 1.将进程的逻辑地址空间分成若干个大小相等的片，称为页面或页
- 2.内存空间分成与页大小相等的若干个存储块，称为物理块或页框
- 3.在为进程分配内存时，以块为单位，将进程中的若干页分别装入多个可以不相邻的块中



#### 硬件页表机制

对于最简单的分页机制，硬件上使用一级页表的方式是最简单的，访问效率也最高，页面的大小一般为 4KB。为了能够定位和访问每个页，需要有个页表，保存每个页的起始地址，再加上在页内的偏移量，组成线性地址，就能对于内存中的每个位置进行访问了，其访问流程图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200112231411581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI0ODkyMzY=,size_16,color_FFFFFF,t_70)

虚拟地址分为两部分，**页号**§和**页内偏移**(o)。页号(用高 20 位表示)作为页表的索引，页表包含物理页每页所在物理内存的基地址。这个基地址与页内偏移(低 12 位)的组合就形成了物理内存地址。一级页表这么简单，只要经过一次的地址转换就能找到对应的物理地址，访问效率应该是最好的。



缺陷：虚拟的地址空间为4GB，如果采用一级页表，采用4KB为一个页，那就需要1M个页表。每一个页表需要4个字节来存储，那么整个4GB的地址空间的映射就需要4MB的内存来存储映射表。如果每个进程都有自己的映射表，100个进程就需要400MB的内存，对于内核来说，确实有点大。



#### 多级页表

##### 提出

（1）想法1：页表里面只存放用到的页

比如一个程序用到了第1，3，5，6页，那么页表里面只需要存储这四页对应的页框号，但是这样的话页表里面的项就**不连续**了，这样找某一页对应的页框就**不能直接使用偏移量的**形式了。比较好的方法是折半查找（因为页号是有顺序的）。但是即便使用折半查找耗费的时间也会比使用偏移量大很多倍。比如如果一个表项有210个，时间复杂度log(210)=10,也就是需要10次，而如果使用偏移量就只需要一次就好了。所以**页表里面的页号必须是连续**的。

（2）想法2：

一个逻辑地址用10bits的页目录号+10bits的页号+12bits的偏移组成。

页目录表的每一项对应一个页表，然后再根据页表找到对应的页。这种思想就类似与书本，目录的地方有一个章目录（页目录表）和节目录（页表），如果要查找某一节的内容首先找到这一章的地方，然后再查具体的某一节。我如果要找第五章的第4节，那么前面四章都不用看，直接找第五章就行了，这样除了第五章之外的页号对应的页框号的就不用存了。能节省大量内存；并且保证了章目录和节目录都是连续的，这就意味着可以使用偏移量的形式查找对应的章节。如下图：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWFnZXMyMDE3LmNuYmxvZ3MuY29tL2Jsb2cvODI1OTc5LzIwMTgwMi84MjU5NzktMjAxODAyMDMxNzU0MzgzNTktNTkwMTQzMjcwLnBuZw?x-oss-process=image/format,png)

- 第一级表称为页目录，存放在一页 4K 大小的页面中，具有 2^10 个 4 字节长度的表项。 这些表象指向对应的二级表。 **线性地址的最高 10 位（31-22）**用作以及表中的索引。
- 第二级称为页表，长度也是 4K 大小的一个页面，最多有 1K 个 4 字节的表项。 每个 4 字节的表项含有相关页面的 20 位物理基地址。 二级页表使用线性地址的中间 10 位（21-12）作为表项索引值，以获取含有页面 20 物理地址基地址的表项。 该20位页面物理基地址和线性地址中的低12位（页内偏移）组合在一起就得到了分页转换过程的输出值，即对应的的最终物理地址。



对于给定的线性地址，CR3 寄存器指定页目录表的基地址。

线性地址的高10位用于索引这个页目录表，以获得指向相关第二级页表的指针。

线性地址空间中间10位用于索引二级页表，以获得物理地址的高20位。

线性地址的低12位直接作为物理地址的低12位，从而组成一个完整的32位物理地址。



缺陷：多级页表虽然解决了内存浪费的问题，但是页表存放在主存中，因此程序每次访存至少须要两次：一次访问获取物理地址，第二次访问才获得数据。

内存访问的速度就减半；对于这种情况，硬件又基于页表的访问局限性设计了TLB来解决这个问题



### 二、基于分页的虚拟内存

#### 1、TLB（快表 cpu封装在芯片里）

多级页表虽然节约了我们的存储空间，但是却存在问题：

- 原本我们对于只需要进行一次地址转换，只需要访问一次内存就能找到对应的物理页号了，算出物理地址
- 现在我们需要多次访问内存，才能找到对应的物理页号。

最终带来了时间上的开销，变成了一个“以时间换空间”的策略



TLB就是页表的Cache，属于MMU（**硬件电路，速度很快**）的一部分，其中存储了当前最可能被访问到的页表项。

可以直接在 TLB 里面查询结果，而不需要多次访问内存来完成一次转换。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200130174425874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI0ODkyMzY=,size_16,color_FFFFFF,t_70)

#### TLB与指令/数据cache的区别?

cache为了更快的访问main memory中的数据和指令，而TLB是为了更快的进行**地址翻译**而将部分的页表内容缓存到了Translation lookasid buffer中，**避免了从main memory访问页表**的过程。



#### TLB组成部分

- 标识区:存放的是虚地址的一部

- 数据区:存放物理页号、存储保护信息以及其他一些辅助信息

  辅助信息包括以下内容：

  （1）有效位(Valid)：对于操作系统，所有的数据都不会加载进内存，当数据不在内存的时候，就需要到硬盘查找并加载到内存。当为1时，表示在内存上，为0时，该页不在内存，就需要到硬盘查找。
  （2）引用位(reference)：对于TLB中的项数是一定的，所以当有新的TLB项需要进来但是又满了的话，如果根据LRU算法，就将最近最少使用的项替换成新的项。故需要引用位。同时要注意的是，页表中也有引用位。
  （3）脏位(dirty):当内存上的某个块需要被新的块替换时，它需要根据脏位判断这个块之前有没有被修改过，如果被修改过，先把这个块更新到硬盘再替换，否则就直接替换。



引用位、脏位何时更新?

1. 如果是TLB命中，那么引用位就会被置1，当TLB或页表满时，就会根据该引用位选择适合的替换位置
2. 如果**TLB命中且这个访存操作是个写操作**，那么脏位就会被置1，表明该页被修改过，当该页要从内存中移除时会先执行将该页写会外存的操作，保证数据被正确修改。



#### TLB的访问流程：

1、当CPU收到应用程序发来的虚拟地址后，首先去TLB中根据标志Tag寻找页表数据，假如TLB中正好存放所需的页表并且有效位是1，说明TLB命中了，那么直接就可以从TLB中获取该虚拟页号对应的物理页号。
2、假如有效位是0，说明该页不在内存中，这时候就发生**缺页异常**，**CPU需要先去外存中将该页调入内存并将页表和TLB更新**
3、假如在TLB中没有找到，通过分页机制来实现虚拟地址到物理地址的查找。
4、如果TLB已经满了，那么还要设计替换算法来决定让哪一个TLB entry失效，从而加载新的页表项。



#### cold TLB

##### 缘由：

对于多核CPU，每个processor core都有自己的TLB。

假如不做任何的处理，那么在进程A切换到进程B的时候，TLB和Cache中同时存在了A和B进程的数据。

- kernel space在所有的进程都是共享的
- 对于A和B进程，它们各种有自己的独立的用户地址空间，也就是说，同样的一个虚拟地址X，在A的地址空间中可以被翻译成Pa，而在B地址空间中会被翻译成Pb，如果在地址翻译过程中，TLB中同时存在A和B进程的数据，那么旧的A地址空间的缓存项会影响B进程地址空间的翻译

##### 解决：

当系统发生进程切换，从进程A切换到进程B，从而导致地址空间也从A切换到B，这时候，我们可以认为在A进程执行过程中，所有TLB和Cache的数据都是for A进程的，一旦切换到B，整个地址空间都不一样了，因此**需要全部flush掉**



##### 后果：

B进程开始执行的时候，TLB和Cache都是cold的。

因此刚开始执行的时候，TLB miss和Cache miss都非常严重，从而导致了性能的下降。我们管这种空TLB叫做**cold TLB**，它需要随着进程的运行warm up起来才能慢慢发挥起来效果，而在这个时候有可能又会有新的进程被调度了，而造成TLB的颠簸效应。



##### 改进：

a、无论进程如何切换，内核地址空间转换到物理地址的关系是永远不变的，其实在进程A切换到B的时候，不需要flush掉，因为B进程也可以继续使用这部分的TLB内容；

b、对于用户地址空间，各个进程都有自己独立的地址空间，在进程A切换到B的时候，TLB中的和A进程相关的entry对于B是完全没有任何意义的，需要flush掉。

在页表描述符中往往有一个bit来标识该地址翻译是global（各个进程共享）还是local（进程特有）的



#### 2、缺页 页面置换

#### 缺页基本原理：

 缺页中断就是要访问的页不在主存，需要操作系统**将其调入主存后再进行访问**

【当软件试图访问已映射在虚拟地址空间中，但是并未被加载在物理内存中的一个分页时，由中央处理器的内存管理单元所发出的中断】



只有程序运行时用到了才去内存中寻找虚拟地址对应的页帧，找不到才可能进行分配，这就是内存的**惰性(延时)分配机制**。



#### 缺页错误的分类：

- 硬件缺页(Hard Page Fault): 此时物理内存中没有对应的页帧，需要CPU打开磁盘设备读取到物理内存中，再让MMU建立VA和PA的映射
- 软缺页(Soft Page Fault): 此时物理内存中存在对应的页帧，只不过可能是其他进程调入，发生缺页异常的进程不知道，此时**MMU只需要重新建立映射**即可，无**需从磁盘写入内存**，一般出现在多进程共享内存区域
- 无效缺页(Invalid Page Falut): 比如进程访问的内存地址**越界访问**，**空指针引用**就会报段错误等



#### 常见的场景：

1、地址空间映射关系未建立：

- 内核提供了很多申请内存的接口函数malloc/mmap，申请的虚拟地址空间，但是**并未分配实际的物理页面，当首次访问的时候将会触发缺页异常**
- 用户态的经常要进行地址访问，在进程刚创建运行时，页会伴随着大量的缺页异常，例如**文件页(代码段/数据段)映射到进程地址空间**，首次访问会产生缺页异常

2、地址空间映射已建立:

- 当访问的页面已经被swapping到磁盘，访问时触发缺页异常
- fork子进程时，子进程共享父进程的地址空间，写时触发缺页异常(COW技术)
- 要访问的页面被KSM合并，写时触发缺页异常(COW技术)

3、访问的地址空间不合法：

- 用户空间访问内核空间地址，触发缺页异常
- 内核空间访问用户空间地址，触发缺页异常



#### 缺页异常软件处理流程：

当进行存储访问时发生异常，处理器会跳转到异常向量表Data abort向量中。

对于ARM处理器而言，当发生异常的时候，处理器会暂停当前指令的执行，保存现场，转而去执行对应的异常向量处的指令，当处理完该异常的时候，恢复现场，回到原来的那点去继续执行程序

- 如果进入data abort之前处于usr模式，那么跳转到dabt_usr

- 如果处于svc模式，那么跳转到dabt_svc；否则跳转到__dabt_invalid。


实际上，进入异常向量前Linux只能处于usr或者svc（supervisor mode）两种模式之一。这时因为irq等异常在跳转表中都要经过vector_stub宏，而不管之前是哪种状态，这个宏都会将CPU状态改为svc模式。



#### 缺页处理具体流程：

```c
do_page_fault(regs,error_code)
    //CR2 寄存器中包含有最新的页错误发生时的虚拟地址
    address = read_cr2()
	__do_page_fault(regs, error_code, address)
    	//处理内核态的缺页中断
    	do_kern_addr_fault(regs, error_code, address);
    	//处理用户态的缺页中断
        do_user_addr_fault(regs, error_code, address);
```



```c
do_user_addr_fault(regs,error_code,address)
	//访问地址合法 -> goto good_area
	handle_mm_fault(vma,address,flags)
    	__handle_mm_fault(vma,address,flags)
    		//根据vmf决定如何分配一个新的页面
    		handle_pte_fault(&vmf)
    			//1.vmf->pte为空，即尚未分配为缺失的页分配页表
    			do_anonymous_page(vmf) or do_fault(vmf)
    			//2.页表已经建立，但不存在于物理内存之中
    			do_swap_page(vmf)
    			//3.页表已经建立，且也贮存在物理内存中，因为写操作触发了缺页中断
    			do_wp_page
```



#### 总结如下：

- #### 访问的页表 Page Table 尚未分配:

  当页表从未被访问时，有两种方法装入所缺失的页，这取决于这个页是否被映射到磁盘文件:

  **`vma->vm_ops`不为NULL**

  即`vma`对应磁盘上某一个文件，调用`vma->vm_ops->fault(vmf)`.

  **`vma->vm_ops`为NULL**

  即`vma`没有对应磁盘的文件为匿名映射，调用`do_anonymous_page(vmf)`分配页面.

  - 处理只读的缺页: `do_read_fault()`

    > 即根据文件系统设置的`vma`的缺页处理函数，在`EXT4`文件系统中，对应的是`ext4_filemap_fault()`，其逻辑就是读文件: 先从 Page Cache 中查找，假如不存在，从文件上读取至 Page Cache.

  - 处理写时复制的缺页: `do_cow_fault()`

  - 处理共享页的缺页: `do_shared_fault()`

- #### 访问的页表已经分配，但保存在 swap 交换区:

  **`do_swap_page()`**

- #### 访问的页表已经分配，且存在于物理内存中，即触发写时复制(COW)的缺页中断:

  **`do_wp_page()`**

  写时复制的概念不多做介绍，仅仅来看 Linux 内核是如何处理写时复制:

  - 为`vma`申请一个 Page.
  - 调用`vma->vm_ops->fault(vmf)`读取数据.
  - 将函数把旧页面的内容复制到新分配的页面.



```c
do_fault
	//vma->vm_ops->fault(vmf)对应的文件系统的缺页处理函数
    .fault		= ext4_filemap_fault,
		ext4_filemap_fault
            //调用了内存管理模块的filemap_fault 
            filemap_fault
```



Page Cache 的插入主要流程（filemap_fault）如下:

- 判断查找的 Page 是否存在于 Page Cache，存在即直接返回
- 否则通过[ Linux 内核物理内存分配](https://www.leviathan.vip/2019/06/01/Linux内核源码分析-Page-Cache原理分析/[http://leviathan.vip/2019/04/13/Linux内核源码分析-物理内存的分配/#核心算法](http://leviathan.vip/2019/04/13/Linux内核源码分析-物理内存的分配/#核心算法)介绍的伙伴系统分配一个空闲的 Page.
- 将 Page 插入 Page Cache，即插入`address_space`的`i_pages`.
- 调用`address_space`的`readpage()`来读取指定 offset 的 Page.



#### 缺页异常调入新页面而内存已满

置换算法选择被置换的物理页面，其主要的目的是以尽可能减少页面的调入调出次数，把未来不再访问或短期内不访问的页面调出，以提高系统的性能

1、局部页面置换

局部页面置换的选择范围仅限于当前进程占用的物理页面内，其主要由最优算法、先进先出算法、最近最久未使用算法

2、全局页面置换

在局部算法里面并没有考虑各个进程之间的访存差异，全局置换算法为进程分配可变数目的物理页面。常驻集是指进程在运行时，当前时刻实际驻留在内存当中的页面集合。而工作集是进行再运行过程所固有的特征。置换算法的工作就是在进程的工作集的前提下，确定常驻集的大小以及相应页面。



#### 参考链接：

1、Linux 内核源码分析-内存请页机制

https://www.leviathan.vip/2019/03/03/Linux%E5%86%85%E6%A0%B8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%86%85%E5%AD%98%E8%AF%B7%E9%A1%B5%E6%9C%BA%E5%88%B6/




### 三、虚拟内存功能

#### 1、共享内存

#### 2、写时拷贝

#### 3、内存去重

#### 4、内存压缩

#### 5、大页

（1）标准大页

（2）透明大页

（3）cgroup大页

（4）tmpfs大页

（5）exec大页

（6）glibc动态库大页

#### 6、交换技术（swap）

目的：更好利用内存；让正在运行的程序或需要运行的程序有更多的内存资源

思路：

- 可将暂时不能运行的程序送入外存中，从而获得空闲内存空间
- 换出（swap out）：把一个进程的整个地址空间保存到外存
- 换入（swap in）：将外存中某进程的地址空间读入到内存
- 换入换出的基本单位：整个进程的地址空间

交换时机：只有当内存空间不够或者空间有不够使用的时候

交换区大小：必须足够大以存放所有用户进程的所有内存映像的拷贝，必须能对这些内存映像进行直接存储

#### 7、局部性原理



### 四、物理内存分配与管理

1、伙伴系统

2、slub算法

3、memblock



### 五、内存申请及映射

#### 1、vmalloc mmap

##### mmap：

mmap用于内存映射，就是将一段区域映射到自己的进程地址空间上

##### **（1）分类：**

https://www.cnblogs.com/LoyenWang/p/12037658.html

- 文件映射： 将文件区域映射到进程空间，文件存放在**存储设备**上；
- 匿名映射：没有文件对应的区域映射，内容存放在**物理内存**上；

同时，针对其他进程是否可见，又分为两种：

- 私有映射：将数据源拷贝副本，不影响其他进程；
- 共享映射：共享的进程都能看到；

根据排列组合，就存在以下几种情况了：

1. 私有匿名映射： 通常分配大块内存时使用，堆，栈，bss段等；
2. 共享匿名映射：常用于**父子进程间通信**，在内存文件系统中创建`/dev/zero`设备；
3. 私有文件映射：常用的比如动态库加载，代码段，数据段等；
4. 共享文件映射：常用于进程间通信，文件读写等；



##### **（2）mmap介绍**

https://blog.csdn.net/weixin_42096901/article/details/103227930

mmap是操作这些设备的一种方法，所谓操作设备，比如IO端口（点亮一个LED）、LCD控制器、磁盘控制器，实际上就是**往设备的物理地址读写数据**

但是，由于应用程序不能直接操作设备硬件地址，所以操作系统提供了这样的一种机制——内存映射，把**设备地址映射到进程虚拟地址**，mmap就是实现内存映射的接口。

操作设备还有很多方法，如ioctl、ioremap

mmap的好处是，mmap把设备内存映射到虚拟内存，则用户操作虚拟内存相当于直接操作设备了，省去了用户空间到内核空间的复制过程，相对IO操作来说，增加了数据的吞吐量。

##### （3）虚拟地址空间：

![img](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwMTE0MjAxMzAzMzkx?x-oss-process=image/format,png)

虚拟空间装的大概是上面那些数据了，内存映射大概就是把设备地址映射到上图的红色段了，暂且称其为“内存映射段”，至于映射到哪个地址，是由操作系统分配的，操作系统会把进程空间划分为三个部分：

（1）未分配的，即进程还未使用的地址

（2）缓存的，缓存在ram中的页

（3）未缓存的，没有缓存在ram中



https://blog.csdn.net/weixin_42096901/article/details/103227930

/dev/mmap_driver设备地址通过mmap映射到了进程地址空间

```shell
[root@localhost ~]# cat /proc/15709/maps
#可执行文件 代码段
00400000-00401000 r-xp 00000000 fd:00 654096                             /root/a.out
#只读段
0041f000-00420000 r--p 0000f000 fd:00 654096                             /root/a.out
#读写段
00420000-00421000 rw-p 00010000 fd:00 654096                             /root/a.out
04096000-040b7000 rw-p 00000000 00:00 0                                  [heap]
ffffaed7f000-ffffaeef0000 r-xp 00000000 fd:00 790146                     /usr/lib64/libc-2.28.so
ffffaeef0000-ffffaef0b000 ---p 00171000 fd:00 790146                     /usr/lib64/libc-2.28.so
ffffaef0b000-ffffaef0f000 r--p 0017c000 fd:00 790146                     /usr/lib64/libc-2.28.so
ffffaef0f000-ffffaef11000 rw-p 00180000 fd:00 790146                     /usr/lib64/libc-2.28.so
ffffaef11000-ffffaef15000 rw-p 00000000 00:00 0 
ffffaef15000-ffffaef36000 r-xp 00000000 fd:00 790139                     /usr/lib64/ld-2.28.so
ffffaef44000-ffffaef46000 rw-p 00000000 00:00 0 
ffffaef51000-ffffaef52000 rw-s 00000000 00:06 374                        /dev/mmap_driver
ffffaef52000-ffffaef53000 r--p 00000000 00:00 0                          [vvar]
ffffaef53000-ffffaef54000 r-xp 00000000 00:00 0                          [vdso]
ffffaef54000-ffffaef55000 r--p 0002f000 fd:00 790139                     /usr/lib64/ld-2.28.so
ffffaef55000-ffffaef56000 rw-p 00030000 fd:00 790139                     /usr/lib64/ld-2.28.so
ffffaef56000-ffffaef57000 rw-p 00000000 00:00 0 
ffffd6ac5000-ffffd6ae6000 rw-p 00000000 00:00 0                          [stack]
```



##### vmalloc

（1）介绍

内核提供了一种申请一片连续的虚拟地址空间，但不保证物理空间连续

- vmalloc的工作方式类似于kmalloc，只不过前者分配的内存虚拟地址连续，而物理地址则无需连续，因此不能用于dma缓冲区
- 通过vmalloc获得的页必须一个一个地进行映射，效率不高，因此不得已时才使用，同时vmalloc分配的一般是大块内存
- vmalloc分配的一般是高端内存，只有当内存不够的时候，才会分配低端内存

（2）数据结构

struct vm_struct(vmalloc描述符)和struct vmap_area(记录在**vmap_area_root**中的vmalloc分配情况和**vmap_area_list**列表中)。

内核在管理虚拟内存中的vmalloc区域时，必须跟踪哪些区域被使用，哪些是空闲的，为此定义了一个数据结构，将所有的部分保存在一个链表中



![image-20230824091133385](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20230824091133385.png)



#### 2、页表映射



#### 3、内存模型

#### 4、匿名映射 文件映射 反向映射



### 六、内存回收

### 七、内存规整

### 八、内存碎片

### 九、pagecache

#### 1、介绍

`Page Cache`是内核与存储介质的重要缓存结构，当我们使用`write()`或者`read()`读写文件时，假如不使用`O_DIRECT`标志位打开文件，我们均需要经过`Page Cache`来帮助我们提高文件读写速度

在 Linux 内核内存的基本单元是`Page`，而`Page Cache`也驻存于物理内存，所以`Page Cache`的缓存基本单位也是`Page`；

而`Page Cache`缓存的内容属于文件系统，所以`Page Cache`属于文件系统与物理内存管理的枢纽



![image-20230717143254559](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20230717143254559.png)

#### 2、page cache相关数据结构

```c
//inode在文件系统代表一个文件的元信息结构
struct inode {
    ...
    //i_mapping代表inode所拥有的address_space
	struct address_space    i_mapping;
  	...
 }
```



```c
struct address_space {
	struct inode		*host;		/* owner: inode, block_device */
	struct radix_tree_root	i_pages;	/* cached pages */
	atomic_t		i_mmap_writable;/* count VM_SHARED mappings */
	struct rb_root_cached	i_mmap;		/* tree of private and shared mappings */
	struct rw_semaphore	i_mmap_rwsem;	/* protect tree, count, list */
	/* Protected by the i_pages lock */
	unsigned long		nrpages;	/* number of total pages */
	/* number of shadow or DAX exceptional entries */
	unsigned long		nrexceptional;
	pgoff_t			writeback_index;/* writeback starts here */
	const struct address_space_operations *a_ops;	/* methods */
	unsigned long		flags;		/* error bits */
	spinlock_t		private_lock;	/* for use by the address_space */
	gfp_t			gfp_mask;	/* implicit gfp mask for allocations */
	struct list_head	private_list;	/* for use by the address_space */
	void			*private_data;	/* ditto */
	errseq_t		wb_err;

	KABI_RESERVE(1)
	KABI_RESERVE(2)
	KABI_RESERVE(3)
	KABI_RESERVE(4)
} __attribute__((aligned(sizeof(long)))) __randomize_layout;
```



```c
struct address_space_operations {
	int (*writepage)(struct page *page, struct writeback_control *wbc);
	int (*readpage)(struct file *, struct page *);

	/* Write back some dirty pages from this mapping. */
	int (*writepages)(struct address_space *, struct writeback_control *);

	/* Set a page dirty.  Return true if this dirtied it */
	int (*set_page_dirty)(struct page *page);

	/*
	 * Reads in the requested pages. Unlike ->readpage(), this is
	 * PURELY used for read-ahead!.
	 */
	int (*readpages)(struct file *filp, struct address_space *mapping,
			struct list_head *pages, unsigned nr_pages);
	...
}
```

- `writepage`：将`Page`写回磁盘。
- `readpage`: 从磁盘读取`Page`。
- `writepages`: 写多个`Page`至磁盘。
- `set_page_dirty`：设置某个`Page`为脏页。
- `readpages`: 读取多个`Page`， 一般用来预读。



#### 3、pagecache回写

- 手动调用`fsync()`或者`sync`强制落盘
- 脏页占用比率过高，超过了设定的阈值，导致内存空间不足，触发刷盘(**强制回写**).
- 脏页驻留时间过长，触发刷盘(**周期回写**).





## 存储

### 一、block

### 1、数据的组织BIO

### 2、mq基本原理

### 3、IO下发概述

### 4、io下发之bio的切分与合并

### 5、io下发之bio bounce过程

### 6、io下发之SGL聚散列表

### 7、io下发之request的分配和获取

### 8、io下发之plug/unplug机制

### 9、io下发之同步与异步下发

### 10、io完成

### 11、io拆分



### 二、scsi

### 1、scsi设备管理

### 2、scsi层io下发和完成

### 3、错误处理

### 4、超时处理



## 网络

### 1、内核如何接收网络包？

### 2、内核如何与用户进程协作？

### 3、内核如何发送网络包？

### 4、本机网络io

### 5、tcp建连过程

### 6、一条tcp连接消耗多大内存

### 7、一台机器最多支持多少tcp连接

### 8、网络性能优化建议

### 9、容器网络虚拟化



## 调度

### 1、irqbalance

### 2、进程调度

### 3、内核调度BFS CFS



## cpu

real > user + sys 表示offcpu

### oncpu/offcpu

- **On-CPU**: where threads are spending time running on-CPU.
- **Off-CPU**: where time is spent waiting while blocked on I/O, locks, timers, paging/swapping, etc.

https://www.brendangregg.com/offcpuanalysis.html

![img](file:///C:/Users/z00585918/AppData/Roaming/eSpace_Desktop/UserData/z00585918/imagefiles/originalImgfiles/DA162A74-0D4C-4605-9D4D-9790C6B2C261.png)

