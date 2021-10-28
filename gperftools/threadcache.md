# ThreadCache

每个线程都有自己的缓存，线程间的小内存申请和释放不需要全局锁抢锁，提高了分配效率，并且ThreadCache还有内存回收策略，尽可能地做到线程间平衡。



## 调用流程：

do_malloc -> GetCache -> GetThreadHeap / CreateCacheIfNecessary

已有局部缓存heap，调用GetThreadHeap；

未获取到局部缓存heap，调用CreateCacheIfNecessary



获取到ThreadCache对象后，通过调用Allocate和Deallocate申请和释放小内存。



## 线程局部缓存

ThreadCache用到了线程局部缓存的概念，主要包括以下几部分：

1、初始化

TCMallocGuard构造函数中就调用初始化函数InitTSD()，函数中会 定义一个pthread_key_t变量heap\_key\_，即perftools_pthread_key_create(&heap_key_, DestroyThreadCache)

2、定义局部缓存heap

每个线程初始化时（通过perftools_pthread_getspecific获取不到heap时），通过perftools\_pthread\_setspecific来定义自己的局部缓存heap

GetCache -> CreateCacheIfNecessary -> perftools_pthread_setspecific(heap_key_, heap);

3、获取局部缓存heap

GetThreadHeap -> perftools_pthread_getspecific(heap_key_)

4、释放局部缓存heap

BecomeIdle -> perftools_pthread_setspecific(heap_key_, NULL);



### ThreadCache结构体

public:



 // All ThreadCache objects are kept in a linked list (for stats collection)

 ThreadCache* next_;

 ThreadCache* prev_;