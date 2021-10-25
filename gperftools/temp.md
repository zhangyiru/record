## 1.1    业务流程

gperftools软件包包括 tcmalloc和一些调测工具。产品主要是使用tcmalloc来替代libc的malloc算法来获取多线程上的性能提升。故核心模块是tcmalloc内存分配算法，以下对tcmalloc进行介绍。

### 1.1.1   tcmalloc概述

tcmalloc是 Google 开发的内存分配器，在不少项目中都有使用，例如在 Golang 中就使用了类似的算法进行内存分配。它具有现代化内存分配器的基本特征：对抗内存碎片、在多核处理器能够扩展

tcmalloc为每个线程分配一个thread-local cache，小对象的分配直接从thread-local cache中分配。根据需要将对象从PageHeap中移动到thread-local cache，同时定期的用垃圾回收器把内存从thread-local cache回收到Central free list中。

### 1.1.2    tcmalloc架构图

​                               

上图展示了tcmalloc的整体结构。

tcmalloc主要由三个组件组成：ThreadCache、CentralFreeList及PageHeap。 其中：

1、ThreadCache: 线程缓存，它是一个TLS(线程本地存储)对象，尺寸小于256K的小内存申请均由ThreadCache进行分配；通过ThreadCache分配过程中不需要任何锁，可以极大的提高分配速度

2、PageHeap: 中央堆分配器，被所有线程共享(分配时需要全局锁定)，负责与操作系统的直接交互(申请及释放内存)，并且大尺寸的内存申请直接通过PageHeap进行分配

3、CentralFreeList：作为PageHeap与ThreadCache的中间人，负责将PageHeap中的内存切分为小块，在恰当时机分配给ThreadCache。获取从ThreadCache中回收的内存并在恰当的时机将部分内存归还给PageHeap

### 1.1.3    tcmalloc核心思想

tcmalloc的动态内存分配核心思想为离散式空闲列表算法

1、freelist

分配定长内存：假设一个page是4KB，要以n字节为单位（object）进行分配

使用freelist来进行分配，将4KB的内存划分为n字节的单元，每个单元的前8个字节作为节点指针，指向下一个单元。

初始化的时候把所有指针指向下一个单元，如上图Freelist展示的，指针ptr指向下一个单元的指针ptr;

分配时，从链表头分配一个对象出去，如上图Allocate展示的，左边绿框表示分配出去的对象，剩下的单元保持不变；释放时，插入到链表。

 

2、核心算法思想

l 线程私有性（ThreadCache）：一般与负责小内存分配，每个线程都拥有一份ThreadCache，理想情况下，每个线程的内存申请都可以在自己的ThreadCache内完成，线程之间无竞争，所以Tcmalloc非常高效

l 内存分配粒度：

1、span：用于内部管理。span是由连续的page内存组成的一个大块内存，负责分配超过256KB的大内存

2、object：用于面向对象分配。object是由span切割成的小块，其尺寸被预设了一些规格（class），如16B，32B（88种），不会大于256KB（交给了span）。同一个span切出来的都是相同的object。

ThreadCache和CentralCache是管理object的，PageHeap管理的是span

 

3、ThreadCache

每个thread独立维护了各自的离散式空闲列表，它的核心结构如下：

```
class FreeList {
private:
   void*    list_;       // Linked list of nodes
   uint32_t length_;      // Current length.
   uint32_t lowater_;     // Low water mark for list length.
   uint32_t max_length_;  // Dynamic max list length based on usage.
};

class ThreadCache {
private:
    FreeList      list_[kClassSizesMax];     // Array indexed by size-class
};
```

每个线程都一个线程局部的 ThreadCache，按照不同的规格，维护了对象的链表；如果ThreadCache 的对象不够了，就从 CentralFreeList 进行批量分配；如果 CentralFreeList 依然没有，就从PageHeap申请Span；如果 PageHeap没有合适的 Page，就只能从操作系统申请了。

在释放内存的时候，ThreadCache依然遵循批量释放的策略，对象积累到一定程度就释放给 CentralFreeList；CentralFreeList发现一个 Span的内存完全释放了，就可以把这个 Span 归还给 PageHeap；PageHeap发现一批连续的Page都释放了，就可以归还给操作系统。

 

4、CentralFreeList

tcmalloc为每个size class设置设置了一个CentralFreeList(中央自由列表)，ThreadCache之间共享这些CentralFreeList。有个问题，在多线程的场景下，所有线程都从centralFreeList分配的话，竞争会很激烈。

```
static CentralFreeListPadded central_cache_[kNumClasses];
 class CentralFreeList {
 private:
     SpinLock lock_;
     size_t size_class_;
     Span empty_;       
     Span nonempty_;
};
```

作用：

（1）维护span的链表，每个span下面再挂一个由这个span切分出来的object的链。便于在span内的object都已经free的情况下，将span整体回收给PageHeap；每个回收回来的object都需要寻找自己所属的span后才挂进freelist，比较耗时。

（2）empty的span链和nonempty的span链：CentralFreeList中的span链表有nonempty*和**empty*两个，根据span的object链是否有空闲，放入对应链表。如果span的内存已经用完则把这个span移到empty链表中。

（3）通过页找到对应span：被CentralFreeList使用的span，都会把这个span上的所有页都注册到radixtree中，这样对于这个span上的任意页都可以通过页ID找到这个span。

（4）如果span的内存已经完全被释放（span->refcount==0），则把这个span归还到PageHead中。

 

每当ThreadCache从CentralFreeList获取object时：

1、从nonempty_链表中获取第一个Span，并从此Span中的objects链表中获取可用object返回，每分配一个object，Span的refcount + 1。

2、 当Span无可用object时，将此Span从nonempty链表摘除，挂接到empty链表(object重新归还给此Span时会从新将其挂载到nonempty_链表)

 

当ThreadCache归还object给CentralFreeList时：

3、找到此object对应的Span，挂接到objects链表表头，如果Span在empty链表，则重新挂接到nonempty链表

4、Span的refcount – 1。如果refcount变成了0，表示此Span所有的object都已经归还，将此Span从CentralFreeList的链表中摘掉，并将其退还给PageHeap。(pageheap->Delete(Span))

 

5、Pageheap

作用：

（1）page到span的映射关系通过radix tree来实现，逻辑上理解为一个大数组，以page的值作为偏移，就能访问到page对应的span节点。

（2）为减少查询radix tree的开销，PageHeap还维护了一个最近最常使用的若干个page到object的对应关系cache。为了保持cache的效率，cache只提供64个固定坑位。

（3）空闲span的伙伴系统为上层提供span的分配与回收。当需要的span没有空闲时，可以把更大尺寸的span拆小；当span回收时，又需要判断相邻的span是否空闲，以便组合他们

（4）normal和returned：多余的内存放到returned中，与normal隔离。normal的内存总是优先被使用，kernel倾向于一直保留他们；而returned的内存不常被使用，kernel内存不足时优先swap他们。

 

## 1.2    tcmalloc接口实现原理

### 1.2.1    内存管理数据结构

（1）PageHeap

内存管理单位：span（连续的page的内存）

（2）CentralCache

内存管理单位：object（由span切成的小块，同一个span切出来的object都是相同的规格）

（3）ThreadCache

线程私有的缓存，理想情况下，每个线程的内存需求都在自己的ThreadCache中完成，线程之间不需要竞争，非常高效。

内存管理单位：class（由span切成的小块）

### 1.2.2    函数入口

1、tcmalloc.cc文件中定义了函数入口。

```
extern "C" {
 void* tc_malloc(size_t size) PERFTOOLS_NOTHROW
     ATTRIBUTE_SECTION(google_malloc);
 void tc_free(void* ptr) PERFTOOLS_NOTHROW
     ATTRIBUTE_SECTION(google_malloc);
}
```

2、tc_malloc和tc_free函数实现

```
extern "C" PERFTOOLS_DLL_DECL void* tc_malloc(size_t size) __THROW {
 void* result = do_malloc_or_cpp_alloc(size);
 MallocHook::InvokeNewHook(result, size);
 return result;
}

extern "C" PERFTOOLS_DLL_DECL void tc_free(void* ptr) __THROW {
 MallocHook::InvokeDeleteHook(ptr);
 do_free(ptr);
}
```

do_malloc_or_cpp_alloc里面可以看到，因为tc_new_mode等于0，所以实际调用的就是do_malloc这个函数

```
inline void* do_malloc_or_cpp_alloc(size_t size) {
 return tc_new_mode ? cpp_alloc(size, true) : do_malloc(size);
}
```

3、tc_malloc和tc_free函数替换为malloc和free

libc_override_glibc.h

```
#define ALIAS(tc_fn)   __attribute__ ((alias (#tc_fn)))
extern "C" {
 void* __libc_malloc(size_t size)                ALIAS(tc_malloc);
 void __libc_free(void* ptr)                     ALIAS(tc_free);
}
```

libc_override_gcc_and_weak.h

```
extern "C" {
 void* malloc(size_t size) __THROW               ALIAS(tc_malloc);
 void free(void* ptr) __THROW                    ALIAS(tc_free);
}
```

 

### 1.2.3   malloc

1、流程

 

（1）小块内存（<256KB）。

ThreadCache：先尝试在list_[class]的FreeList分配。

CentralCache：找到对应class的tc_slots链表，从链表中分配 -> 从nonempty_链表分配（尽量分配batch_size个object）

HeadPage：伙伴系统对应的npages的span链表 （normal->returned）-> 更大的npages的span链表，拆小

kernel：申请若干个page的内存（可能大于npages）

（2）大块内存（>256KB）。

HeadPage：伙伴系统对应的npages的span链表 （normal->returned）-> 更大的npages的span链表，拆小

kernel：申请若干个page的内存（可能大于npages）

 

2、分配逻辑

分配入口是do_malloc函数

```
inline void* do_malloc(size_t size) {
 void* ret = NULL;

 // The following call forces module initialization
 ThreadCache* heap = ThreadCache::GetCache();
 if (size <= kMaxSize) { //kMaxSize=256k
   size_t cl = Static::sizemap()->SizeClass(size);
   size = Static::sizemap()->class_to_size(cl);
//采样分配
   if ((FLAGS_tcmalloc_sample_parameter > 0) && heap->SampleAllocation(size)) {
     ret = DoSampledAllocation(size);
  } else {
     // The common case, and also the simplest. This just pops the
     // size-appropriate freelist, after replenishing it if it's empty.
     ret = CheckedMallocResult(heap->Allocate(size, cl));
  }
} else {
   ret = do_malloc_pages(heap, size); //分配对象大于256k的情况
}
 if (ret == NULL) errno = ENOMEM;
 return ret;
}
```

**（****1****）小对象分配逻辑**

通过ThreadCache分配小内存

1、 通过SizeMap查找要分配的内存对应的size class及object size大小。

2、 查看当前ThreadCache的free list是否为空，如果free list不为空，直接从列表中移除第一个object并返回，由于这个过程中需要获取任何锁，所以速度极快。

3、 如果free list为空，从CentralFreeList中获取若干个object(具体object个数由慢启动算法决定，防止空间浪费)到ThreadCache对应的size class列表中，并取出其中一个object返回。

4、 如果CentralFreeList中object也不够用，则CentralFreeList会向PageHeap申请一连串页面(由Span表示，每次申请class_to_pages个)，并将申请的页面切割成一系列的object，之后再将部分object转移给ThreadCache。

**（****2****）大对象分配逻辑**

入口是do_malloc_pages函数

```
inline void* do_malloc_pages(ThreadCache* heap, size_t size) {
 void* result;
 bool report_large;

 Length num_pages = tcmalloc::pages(size); 
 size = num_pages << kPageShift;
 //采样分配
 if ((FLAGS_tcmalloc_sample_parameter > 0) && heap->SampleAllocation(size)) {
   result = DoSampledAllocation(size);
    
   SpinLockHolder h(Static::pageheap_lock());
   report_large = should_report_large(num_pages);
} else {
   SpinLockHolder h(Static::pageheap_lock());
   Span* span = Static::pageheap()->New(num_pages);
   result = (span == NULL ? NULL : SpanToMallocResult(span)); //检查span是否可以，已经将span的slab[0]缓存
   report_large = should_report_large(num_pages); //判断对象分配地是否过大
}

 if (report_large) {
   ReportLargeAlloc(num_pages, result); //如果分配过大选择进行report
}
 return result;
}
```

tcmalloc使用基于页的分配方式，即每次至少像系统申请1页空间。tcmalloc中定义的页大小为8K个字节

虽然PageHeap是按页申请内存，但是它管理内存的基本单位为Span(跨度)，Span对象代表了表示连续的页面

```
struct Span {
 PageID        start;          // Span描述的内存的起始地址
 Length        length;         // Span页面数量
 Span*         next;           // Span由双向链表组成，PageHeap和CentralFreeList中都用的到
 Span*         prev;           //
 void*         objects;        // Span会在CentralFreeList中拆分成由object组成的free list
 unsigned int  refcount : 16;  // Span的object被引用次数，当refcount=0时，表示此Span没有被使用
 unsigned int  sizeclass : 8;  // Span属于的size class
 unsigned int  location : 2;   // Span在的位置
 unsigned int  sample : 1;     // Sampled object?
 // What freelist the span is on: IN_USE if on none, or normal or returned
 enum { IN_USE, ON_NORMAL_FREELIST, ON_RETURNED_FREELIST };
};
```

**PageHeap****管理****Span**

```
PageMap pagemap_; // page id 到 Span的映射

struct SpanList {
  Span        normal;
  Span        returned;
};

SpanList large_;

SpanList free_[kMaxPages]; // kMaxPages = 128
```

从PageHeap的主要结构中看到：

1、   PageHeap通过free_数组保存了每个页大小对应的空闲Span双向链表。

2、   大于kMaxPages页面，统一保存在large_中，不再按照页面数目区分。

3、   Span列表又分为了normal和returned两个部分，其中:

l normal部分包含的Span，是页面明确映射到进程地址空间的Span。

l returned部分包含的Span，是tcmalloc已经通过madvise归还给操作系统空间，调用madvise相当于取消了虚拟内存与物理内存的映射。 tcmalloc之所以还保留returned列表，是因为虽然通过madvise归还给了操作系统，但是操作系统有可能还没有收回这部分内存空间，可以直接利用，如果在操作系统回收前tcmalloc重新使用了这些页面，那么系统就不会再进行回收。并且，即使操作系统已经回收了这部分内存，重新使用这部分空间时内核会引发page fault并将其映射到一块全零的内存空间，不影响使用（代价是会影响性能）。

当调用Span* New(Length n) 申请内存时（n代表的是申请分配的页面数目）：

1、free[kMaxPages]中大于等于n的free list会被遍历一遍，查找是否有合适大小的Span；如果有，则将此Span从free list中移除；如果Span大小比n大，tcmalloc则会将其Carve，将剩余的Span重新放到free_list中。比如，n = 3, 但是系统遍历时发现free[3]对应的索引已经没有空闲Span了，但是在free[4]中找到了空闲Span，这时候此Span会被切分成两份：一份对应3个页面，返回给调用方；一份对应1个页面，挂接到free_[1]中，供下次使用。

2、如果free中的normal和returned链表中都找不到合适的Span，则从large链表中查找大小最合适的Span，这时候需要遍历整个large_的normal和returned列表，时间复杂度为O(n)

3、如果large_中也没有可用Span，则通过tcmalloc_SystemAlloc()向操作系统申请，并返回指定大小Span。(每次都会尝试申请至少128页(kMinSystemAlloc)，以便下次使用)

### 1.2.4    free

**1****、回收流程**

（1）ThreadCache => CentralCache

ThreadCache容量限额：

a、为每一个ThreadCache初始化一个比较小的限额，然后每当ThreadCache由于cache超限而触发object到CentralCache的回收时，就增大限额。

b、预设所有ThreadCache的总容量，一个ThreadCache容量不够时，从其他ThreadCache收刮（轮询）。

c、每个ThreadCache也有最大最小值限制，不能无限增大限额。

d、每个ThreadCache超过限额时，对其每个FreeList回收。

单个FreeList的限额：

a、慢启动。初始长度限制为1，限额1~batch_size之间为慢启动，每次限额+1。

b、超过batch_size，限额按照batch_size整数倍扩展。

c、FreeList限额超限，直接回收batch_size个object。

（2）CentralCache => PageHeap

只要一个span里面的object都空闲了，就将它回收到PageHeap。

（3）PageHead中的normal => returned

a、每当PageHeap回收到N个page的span时（这个过程中可能伴随着相当数目的span分配），触发一次normal到returned的回收，只回收一个span。 b、这个N值初始化为1G内存的page数，每次回收span到returned链之后，可能还会增加N值，但是最大不超过4G。 c、触发回收的过程，每次进来轮询伙伴系统中的一个normal链表，将链尾的span回收即可。

 

**2****、代码逻辑**

函数入口为do_free

```
// The default "do_free" that uses the default callback.
inline void do_free(void* ptr) {
 return do_free_with_callback(ptr, &InvalidFree);
}
```

然后调用do_free_with_callback

```
// This lets you call back to a given function pointer if ptr is invalid.
// It is used primarily by windows code which wants a specialized callback.
inline void do_free_with_callback(void* ptr, void (*invalid_free_fn)(void*)) {
 if (ptr == NULL) return;
 if (Static::pageheap() == NULL) {
   // We called free() before malloc(). This can occur if the
   // (system) malloc() is called before tcmalloc is loaded, and then
   // free() is called after tcmalloc is loaded (and tc_free has
   // replaced free), but before the global constructor has run that
   // sets up the tcmalloc data structures.
  (*invalid_free_fn)(ptr);  // Decide how to handle the bad free request
   return;
}
 const PageID p = reinterpret_cast<uintptr_t>(ptr) >> kPageShift;
 Span* span = NULL;
 size_t cl = Static::pageheap()->GetSizeClassIfCached(p);//查看threadcache是否有class的信息

 if (cl == 0) { //如果没有，需要去pagemap里面查询到span粒度
   span = Static::pageheap()->GetDescriptor(p);
   if (!span) { //如果查询不到span，则认为是指针错误
     // span can be NULL because the pointer passed in is invalid
     // (not something returned by malloc or friends), or because the
     // pointer was allocated with some other allocator besides
     // tcmalloc. The latter can happen if tcmalloc is linked in via
     // a dynamic library, but is not listed last on the link line.
     // In that case, libraries after it on the link line will
     // allocate with libc malloc, but free with tcmalloc's free.
    (*invalid_free_fn)(ptr);  // Decide how to handle the bad free request
     return;
  }
   //取出slab class并缓存
   cl = span->sizeclass;
   Static::pageheap()->CacheSizeClass(p, cl);
}
 if (cl != 0) { //如果是小对象释放
   ASSERT(!Static::pageheap()->GetDescriptor(p)->sample);
   ThreadCache* heap = GetCacheIfPresent(); //获取当前进程的thread cache
   if (heap != NULL) {
     heap->Deallocate(ptr, cl); //回收到thread cache里面
  } else {
     // Delete directly into central cache
     tcmalloc::SLL_SetNext(ptr, NULL);
     Static::central_cache()[cl].InsertRange(ptr, ptr, 1);
  }
} else {
   SpinLockHolder h(Static::pageheap_lock());
   ASSERT(reinterpret_cast<uintptr_t>(ptr) % kPageSize == 0);
   ASSERT(span != NULL && span->start == p);
   if (span->sample) {
     StackTrace* st = reinterpret_cast<StackTrace*>(span->objects);
     tcmalloc::DLL_Remove(span);
     Static::stacktrace_allocator()->Delete(st);
     span->objects = NULL;
  }
   //大对象直接由pageheap释放
   Static::pageheap()->Delete(span);
}
}
```