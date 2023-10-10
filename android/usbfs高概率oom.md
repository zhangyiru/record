

### 背景

kmalloc分配不出连续的物理内存

```c
3182_2: page allocation failure: order:4, mode:0x6040c0(GFP_KERNEL|__GFP_COMP), nodemask=(null)

#define GFP_KERNEL      (__GFP_RECLAIM | __GFP_IO | __GFP_FS)

#define ___GFP_RECLAIMABLE      0x10u
#define ___GFP_IO               0x40u
#define ___GFP_FS               0x80u
#define ___GFP_COMP             0x4000u
```



```
#define ___GFP_NOFAIL           0x800u
```



### _alloc_pages_nodemask是伙伴系统的核心，处理实质的内存分配工作。

（1）先进行参数初始化:alloc_mask, alloc_flags和struct alloc_context ac，用于决定内存块的分配配条件。
（2） get_page_from_freelist:内核内存环境良好，直接进行快速分配，若成功返回获取free内存块
（3）__alloc_pages_slowpath:当前内存环境恶劣时，进入慢分配流程，若成功返回free内存块
（4）获取空间内存块后对内存块和系统环境做检查，满足预定要求则返回申请的内存给内核使用



```c
4515 static inline struct page *
4516 __alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
4517                                                 struct alloc_context *ac)
4518 {
4519         bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
4520         const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
4521         struct page *page = NULL;
4522         unsigned int alloc_flags;
4523         unsigned long did_some_progress;
4524         enum compact_priority compact_priority;
4525         enum compact_result compact_result;
4526         int compaction_retries;
4527         int no_progress_loops;
4528         unsigned int cpuset_mems_cookie;
4529         int reserve_flags;
4530
4531         /*
4532          * We also sanity check to catch abuse of atomic reserves being used by
4533          * callers that are not in atomic context.
4534          */
4535         if (WARN_ON_ONCE((gfp_mask & (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)) ==
4536                                 (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)))
4537                 gfp_mask &= ~__GFP_ATOMIC;
4538
4539 retry_cpuset:
4540         compaction_retries = 0;
4541         no_progress_loops = 0;
4542         compact_priority = DEF_COMPACT_PRIORITY;
4543         cpuset_mems_cookie = read_mems_allowed_begin();
4544
4545         /*
4546          * The fast path uses conservative alloc_flags to succeed only until
4547          * kswapd needs to be woken up, and to avoid the cost of setting up
4548          * alloc_flags precisely. So we do that now.
4549          */
4550         alloc_flags = gfp_to_alloc_flags(gfp_mask);
4551
4552         /*
4553          * We need to recalculate the starting point for the zonelist iterator
4554          * because we might have used different nodemask in the fast path, or
4555          * there was a cpuset modification and we are retrying - otherwise we
4556          * could end up iterating over non-eligible zones endlessly.
4557          */
4558         ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
4559                                         ac->high_zoneidx, ac->nodemask);
4560         if (!ac->preferred_zoneref->zone)
4561                 goto nopage;
4562
4563         if (alloc_flags & ALLOC_KSWAPD)
4564                 wake_all_kswapds(order, gfp_mask, ac);
4565
4566         /*
4567          * The adjusted alloc_flags might result in immediate success, so try
4568          * that first
4569          */
4570         page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
4571         if (page)
4572                 goto got_pg;
4573
4574         /*
4575          * For costly allocations, try direct compaction first, as it's likely
4576          * that we have enough base pages and don't need to reclaim. For non-
4577          * movable high-order allocations, do that as well, as compaction will
4578          * try prevent permanent fragmentation by migrating from blocks of the
4579          * same migratetype.
4580          * Don't try this for allocations that are allowed to ignore
4581          * watermarks, as the ALLOC_NO_WATERMARKS attempt didn't yet happen.
4582          */
4583         if (can_direct_reclaim &&
4584                         (costly_order ||
4585                            (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
4586                         && !gfp_pfmemalloc_allowed(gfp_mask)) {
4587                 page = __alloc_pages_direct_compact(gfp_mask, order,
4588                                                 alloc_flags, ac,
4589                                                 INIT_COMPACT_PRIORITY,
4590                                                 &compact_result);
4591                 if (page)
4592                         goto got_pg;
4593
4594                 /*
4595                  * Checks for costly allocations with __GFP_NORETRY, which
4596                  * includes THP page fault allocations
4597                  */
4598                 if (costly_order && (gfp_mask & __GFP_NORETRY)) {
4599                         /*
4600                          * If compaction is deferred for high-order allocations,
4601                          * it is because sync compaction recently failed. If
4602                          * this is the case and the caller requested a THP
4603                          * allocation, we do not want to heavily disrupt the
4604                          * system, so we fail the allocation instead of entering
4605                          * direct reclaim.
4606                          */
4607                         if (compact_result == COMPACT_DEFERRED)
4608                                 goto nopage;
4609
4610                         /*
4611                          * Looks like reclaim/compaction is worth trying, but
4612                          * sync compaction could be very expensive, so keep
4613                          * using async compaction.
4614                          */
4615                         compact_priority = INIT_COMPACT_PRIORITY;
4616                 }
4617         }
4618
4619 retry:
4620         /* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
4621         if (alloc_flags & ALLOC_KSWAPD)
4622                 wake_all_kswapds(order, gfp_mask, ac);
4623
4624         reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
4625         if (reserve_flags)
4626                 alloc_flags = reserve_flags;
4627
4628         /*
4629          * Reset the nodemask and zonelist iterators if memory policies can be
4630          * ignored. These allocations are high priority and system rather than
4631          * user oriented.
4632          */
4633         if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
4634                 ac->nodemask = NULL;
4635                 ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
4636                                         ac->high_zoneidx, ac->nodemask);
4637         }
4638
4639         /* Attempt with potentially adjusted zonelist and alloc_flags */
4640         page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
4641         if (page)
4642                 goto got_pg;
4643
4644         /* Caller is not willing to reclaim, we can't balance anything */
4645         if (!can_direct_reclaim)
4646                 goto nopage;
4647
4648         /* Avoid recursion of direct reclaim */
4649         if (current->flags & PF_MEMALLOC)
4650                 goto nopage;
4651
4652         if (fatal_signal_pending(current) && !(gfp_mask & __GFP_NOFAIL) &&
4653                         (gfp_mask & __GFP_FS))
4654                 goto nopage;
4655
4656         /* Try direct reclaim and then allocating */
4657         page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
4658                                                         &did_some_progress);
4659         if (page)
4660                 goto got_pg;
4661
4662         /* Try direct compaction and then allocating */
4663         page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
4664                                         compact_priority, &compact_result);
4665         if (page)
4666                 goto got_pg;
4667
4668         /* Do not loop if specifically requested */
4669         if (gfp_mask & __GFP_NORETRY)
4670                 goto nopage;
4671
4672         /*
4673          * Do not retry costly high order allocations unless they are
4674          * __GFP_RETRY_MAYFAIL
4675          */
4676         if (costly_order && !(gfp_mask & __GFP_RETRY_MAYFAIL))
4677                 goto nopage;
4678
4679         if (should_reclaim_retry(gfp_mask, order, ac, alloc_flags,
4680                                  did_some_progress > 0, &no_progress_loops))
4681                 goto retry;
4682
4683         /*
4684          * It doesn't make any sense to retry for the compaction if the order-0
4685          * reclaim is not able to make any progress because the current
4686          * implementation of the compaction depends on the sufficient amount
4687          * of free memory (see __compaction_suitable)
4688          */
4689         if ((did_some_progress > 0 ||
4690                         IS_ENABLED(CONFIG_HAVE_LOW_MEMORY_KILLER)) &&
4691                         should_compact_retry(ac, order, alloc_flags,
4692                                 compact_result, &compact_priority,
4693                                 &compaction_retries))
4694                 goto retry;
4695
4696         if (order <= PAGE_ALLOC_COSTLY_ORDER && should_ulmk_retry(gfp_mask))
4697                 goto retry;
4698
4699         /* Deal with possible cpuset update races before we start OOM killing */
4700         if (check_retry_cpuset(cpuset_mems_cookie, ac))
4701                 goto retry_cpuset;
4702
4703         /* Reclaim has failed us, start killing things */
4704         page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
4705         if (page)
4706                 goto got_pg;
4707
4708         /* Avoid allocations with no watermarks from looping endlessly */
4709         if (tsk_is_oom_victim(current) &&
4710             (alloc_flags == ALLOC_OOM ||
4711              (gfp_mask & __GFP_NOMEMALLOC)))
4712                 goto nopage;
4713
4714         /* Retry as long as the OOM killer is making progress */
4715         if (did_some_progress) {
4716                 no_progress_loops = 0;
4717                 goto retry;
4718         }
4719
4720 nopage:
4721         /* Deal with possible cpuset update races before we fail */
4722         if (check_retry_cpuset(cpuset_mems_cookie, ac))
4723                 goto retry_cpuset;
4724
4725         /*
4726          * Make sure that __GFP_NOFAIL request doesn't leak out and make sure
4727          * we always retry
4728          */
4729         if (gfp_mask & __GFP_NOFAIL) {
4730                 /*
4731                  * All existing users of the __GFP_NOFAIL are blockable, so warn
4732                  * of any new users that actually require GFP_NOWAIT
4733                  */
4734                 if (WARN_ON_ONCE(!can_direct_reclaim))
4735                         goto fail;
4736
4737                 /*
4738                  * PF_MEMALLOC request from this context is rather bizarre
4739                  * because we cannot reclaim anything and only can loop waiting
4740                  * for somebody to do a work for us
4741                  */
4742                 WARN_ON_ONCE(current->flags & PF_MEMALLOC);
4743
4744                 /*
4745                  * non failing costly orders are a hard requirement which we
4746                  * are not prepared for much so let's warn about these users
4747                  * so that we can identify them and convert them to something
4748                  * else.
4749                  */
4750                 WARN_ON_ONCE(order > PAGE_ALLOC_COSTLY_ORDER);
4751
4752                 /*
4753                  * Help non-failing allocations by giving them access to memory
4754                  * reserves but do not use ALLOC_NO_WATERMARKS because this
4755                  * could deplete whole memory reserves which would just make
4756                  * the situation worse
4757                  */
4758                 page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_HARDER, ac);
4759                 if (page)
4760                         goto got_pg;
4761
4762                 cond_resched();
4763                 goto retry;
4764         }
4765 fail:
4766         warn_alloc(gfp_mask, ac->nodemask,
4767                         "page allocation failure: order:%u", order);
4768 got_pg:
4769         return page;
4770 }
```



### 碎片化程度

/sys/kernel/debug/extfrag/extfrag_index

值越大 碎片化越严重

/sys/kernel/debug/extfrag/unusable_index



### 记一次内存分配失败导致的自动重启

https://juejin.cn/post/7071439581928751140



//显示所有zone下不同order空闲数目统计信息* 

//'U'：不可移动 

//'M'：可移动

//'E'：可回收

//'H'：等同于MIGRATE_PCPTYPES

//'C'：CMA区域页面



### 内存打印信息函数

show_free_areas

![image-20230912194309059](/Users/xayahion/Library/Application Support/typora-user-images/image-20230912194309059.png)



### 定位进展

9.12

分析日志发现是在慢速分配内存时分配不出，怀疑是碎片化导致，等再次复现上环境查看碎片化相关指标（/sys/kernel/debug/extfrag/extfrag_index）；

仅复现一次，下午多次尝试均未复现；

疑点：不管是normal还是movable，其中关于64kb的内存块都非0，buddy系统按照order块管理，按逻辑应该可以申请

32*64kB (UMEC)   

1*64kB (M) 



```c
static inline unsigned int
gfp_to_alloc_flags(gfp_t gfp_mask)
{
        unsigned int alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET;

        /* __GFP_HIGH is assumed to be the same as ALLOC_HIGH to save a branch. */
        BUILD_BUG_ON(__GFP_HIGH != (__force gfp_t) ALLOC_HIGH);

        /*
         * The caller may dip into page reserves a bit more if the caller
         * cannot run direct reclaim, or if the caller has realtime scheduling
         * policy or is asking for __GFP_HIGH memory.  GFP_ATOMIC requests will
         * set both ALLOC_HARDER (__GFP_ATOMIC) and ALLOC_HIGH (__GFP_HIGH).
         */
        alloc_flags |= (__force int) (gfp_mask & __GFP_HIGH);

        if (gfp_mask & __GFP_ATOMIC) {
                /*
                 * Not worth trying to allocate harder for __GFP_NOMEMALLOC even
                 * if it can't schedule.
                 */
                if (!(gfp_mask & __GFP_NOMEMALLOC))
                        alloc_flags |= ALLOC_HARDER;
                /*
                 * Ignore cpuset mems for GFP_ATOMIC rather than fail, see the
                 * comment for __cpuset_node_allowed().
                 */
                alloc_flags &= ~ALLOC_CPUSET;
        } else if (unlikely(rt_task(current)) && !in_interrupt())
                alloc_flags |= ALLOC_HARDER;

        if (gfp_mask & __GFP_KSWAPD_RECLAIM)
                alloc_flags |= ALLOC_KSWAPD;

#ifdef CONFIG_CMA
        if ((gfpflags_to_migratetype(gfp_mask) == MIGRATE_MOVABLE) &&
                                (gfp_mask & __GFP_CMA))
                alloc_flags |= ALLOC_CMA;
#endif
        return alloc_flags;
}
```



GFP_KERNEL|__GFP_COMP

```c
#define GFP_MOVABLE_MASK (__GFP_RECLAIMABLE|__GFP_MOVABLE)
static inline int gfpflags_to_migratetype(const gfp_t gfp_flags)
{
        VM_WARN_ON((gfp_flags & GFP_MOVABLE_MASK) == GFP_MOVABLE_MASK);
        BUILD_BUG_ON((1UL << GFP_MOVABLE_SHIFT) != ___GFP_MOVABLE);
        BUILD_BUG_ON((___GFP_MOVABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_MOVABLE);

        if (unlikely(page_group_by_mobility_disabled))
                return MIGRATE_UNMOVABLE;

        /* Group based on mobility */
        return (gfp_flags & GFP_MOVABLE_MASK) >> GFP_MOVABLE_SHIFT;
}
```



```c
#define ___GFP_MOVABLE          0x08u
#define ___GFP_RECLAIMABLE      0x10u
#define ___GFP_CMA              0x1000000u

GFP_KERNEL| __GFP_COMP | (__GFP_RECLAIMABLE|__GFP_MOVABLE) >> 3
  
0x6040c0 | 0x08 | 0x10 -> 0x6040d8
0x6040d8 >> 3 -> 0xc081b != MIGRATE_MOVABLE
  
//故不会设置ALLOC_CMA
```



分析日志发现是在慢速分配内存时分配不出，怀疑是碎片化导致，等再次复现上环境查看碎片化相关指标（/sys/kernel/debug/extfrag/extfrag_index）；

仅复现一次，下午多次尝试均未复现；

疑点：不管是normal还是movable，其中关于64kb的内存块都非0，buddy系统按照order块管理，按逻辑应该可以申请

32*64kB (UMEC)   

1*64kB (M) 



1.page allocation failure: order:4, mode:0x6040c0(GFP_KERNEL|__GFP_COMP), nodemask=(null)
GFP_KERNEL|__GFP_COMP不会去设置ALLOC_CMA
2.不管是normal还是movable，其中关于64kb的内存块都非0，buddy系统按照order块管理，按逻辑应该可以申请
32*64kB (UMEC)   
1*64kB (M) 
3.单numa