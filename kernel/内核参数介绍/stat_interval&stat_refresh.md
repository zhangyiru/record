stat_interval
更新虚拟内容统计信息的时间间隔

stat_refresh
任何读或写操作（仅限root用户）都会将所有per-cpu虚拟内存统计信息刷新到其全局总数中

```c
#ifdef CONFIG_SMP
	{
		.procname	= "stat_interval",
		.data		= &sysctl_stat_interval,
		.maxlen		= sizeof(sysctl_stat_interval),
		.mode		= 0644,
		.proc_handler	= proc_dointvec_jiffies,
	},
	{
		.procname	= "stat_refresh",
		.data		= NULL,
		.maxlen		= 0,
		.mode		= 0600,
		.proc_handler	= vmstat_refresh,
	},
#endif
```

vmstat_refresh->refresh_vm_stats->refresh_cpu_vm_stats

zone->pageset

*这个数组用于实现每个CPU的热/冷页帧列表。*

*内核使用这些列表来保存可用于满足实现的“新鲜”页。*

*但冷热页帧对应的高速缓存状态不同：有些页帧很可能在高速缓存中，因此可以快速访问，故称之为热的；*

*未缓存的页帧与此相对，称之为冷的。*



```c
/*
 * Update the zone counters for the current cpu.
 *
 * Note that refresh_cpu_vm_stats strives to only access
 * node local memory. The per cpu pagesets on remote zones are placed
 * in the memory local to the processor using that pageset. So the
 * loop over all zones will access a series of cachelines local to
 * the processor.
    此函数会获取节点局部内存。
    远程zone上的per_cpu页面集放置在使用该页面集的处理器本地内存中
    所有zone上的循环会访问处理器本地的一系列cacheline
 *
 * The call to zone_page_state_add updates the cachelines with the
 * statistics in the remote zone struct as well as the global cachelines
 * with the global counters. These could cause remote node cache line
 * bouncing and will have to be only done when necessary.
 *
	对于zone_page_state_add的调用会更新在远程zone结构体上的cacheline信息和全局计数器的全局cacheline
    这些操作可能会导致远程cache line反弹，并且只有在必要时才能执行
 * The function returns the number of global counters updated.
 	此函数会返回全局计数器更新的数目
 */
static int refresh_cpu_vm_stats(bool do_pagesets)
{
	struct pglist_data *pgdat;
	struct zone *zone;
	int i;
	int global_zone_diff[NR_VM_ZONE_STAT_ITEMS] = { 0, };
#ifdef CONFIG_NUMA
	int global_numa_diff[NR_VM_NUMA_STAT_ITEMS] = { 0, };
#endif
	int global_node_diff[NR_VM_NODE_STAT_ITEMS] = { 0, };
	int changes = 0;
	
    //用于迭代所有内存区域的helper宏，除了不是populated_zone（zone有memory）都迭代
    //迭代所有有memory的zone
	for_each_populated_zone(zone) {
		struct per_cpu_pageset __percpu *p = zone->pageset;

		for (i = 0; i < NR_VM_ZONE_STAT_ITEMS; i++) {
			int v;

			v = this_cpu_xchg(p->vm_stat_diff[i], 0);
			if (v) {

				atomic_long_add(v, &zone->vm_stat[i]);
				global_zone_diff[i] += v;
#ifdef CONFIG_NUMA
				/* 3 seconds idle till flush */
				__this_cpu_write(p->expire, 3);
#endif
			}
		}
#ifdef CONFIG_NUMA
		for (i = 0; i < NR_VM_NUMA_STAT_ITEMS; i++) {
			int v;

			v = this_cpu_xchg(p->vm_numa_stat_diff[i], 0);
			if (v) {

				atomic_long_add(v, &zone->vm_numa_stat[i]);
				global_numa_diff[i] += v;
				__this_cpu_write(p->expire, 3);
			}
		}

		if (do_pagesets) {
			cond_resched();
			/*
			 * Deal with draining the remote pageset of this
			 * processor
				处理耗尽此远程pageset的处理器
			 *
			 * Check if there are pages remaining in this pageset
			 * if not then there is nothing to expire.
				检查是否在这个pageset中保留页面，如果没有就没有过期的内容
            */
            
			if (!__this_cpu_read(p->expire) ||
			       !__this_cpu_read(p->pcp.count))
				continue;

			/*
			 * We never drain zones local to this processor.
			 	不会将本地zone遗漏到处理器
			 */
			if (zone_to_nid(zone) == numa_node_id()) {
				__this_cpu_write(p->expire, 0);
				continue;
			}

			if (__this_cpu_dec_return(p->expire))
				continue;

			if (__this_cpu_read(p->pcp.count)) {
				drain_zone_pages(zone, this_cpu_ptr(&p->pcp));
				changes++;
			}
		}
#endif
	}
	//迭代所有online节点
	for_each_online_pgdat(pgdat) {
		struct per_cpu_nodestat __percpu *p = pgdat->per_cpu_nodestats;

		for (i = 0; i < NR_VM_NODE_STAT_ITEMS; i++) {
			int v;

			v = this_cpu_xchg(p->vm_node_stat_diff[i], 0);
			if (v) {
				atomic_long_add(v, &pgdat->vm_stat[i]);
				global_node_diff[i] += v;
			}
		}
	}

#ifdef CONFIG_NUMA
    //将差异叠加到全局计数器中，返回更新的计数器数目
	changes += fold_diff(global_zone_diff, global_numa_diff,
			     global_node_diff);
#else
	changes += fold_diff(global_zone_diff, global_node_diff);
#endif
	return changes;
}
```



jiffies记录了系统启动以来，经过了多少tick。

一个tick代表多长时间，在内核的CONFIG_HZ中定义。比如CONFIG_HZ=200，则一个jiffies对应5ms时间。所以内核基于jiffies的定时器精度也是5ms。

```c
static void vmstat_update(struct work_struct *w)
{
	if (refresh_cpu_vm_stats(true)) {
		/*
		 * Counters were updated so we expect more updates
		 * to occur in the future. Keep on running the
		 * update worker thread.
		 	计数器已更新，因此我们预计将来会有更多更新。继续运行更新工作线程
		 */
        //延迟后在特定CPU上排队工作
		queue_delayed_work_on(smp_processor_id(), mm_percpu_wq,
				this_cpu_ptr(&vmstat_work),
				round_jiffies_relative(sysctl_stat_interval));
	}
}
```



```c
int vmstat_refresh(struct ctl_table *table, int write,
		   void *buffer, size_t *lenp, loff_t *ppos)
{
	long val;
	int err;
	int i;

	/*
	 * The regular update, every sysctl_stat_interval, may come later
	 * than expected: leaving a significant amount in per_cpu buckets.
	 定期更新，对于每个sysctl_stat_interval，可能会比预期要晚：在per_cpu桶中留下大量数据
	 * This is particularly misleading when checking a quantity of HUGE
	 * pages, immediately after running a test.
     在运行测试后立即检查大页数目，尤其容易产生错误
	 * /proc/sys/vm/stat_refresh,
	 * which can equally be echo'ed to or cat'ted from (by root),
	 * can be used to update the stats just before reading them.
	 可以使用root读写，用来在读取之前更新统计信息
	 *
	 * Oh, and since global_zone_page_state() etc. are so careful to hide
	 * transiently negative values, report an error here if any of
	 * the stats is negative, so we know to go looking for imbalance.
	 */
	err = schedule_on_each_cpu(refresh_vm_stats);
	if (err)
		return err;
	for (i = 0; i < NR_VM_ZONE_STAT_ITEMS; i++) {
		/*
		 * Skip checking stats known to go negative occasionally.
		 跳过对于负数统计信息的检查
		 */
		switch (i) {
		case NR_ZONE_WRITE_PENDING:
		case NR_FREE_CMA_PAGES:
			continue;
		}
		val = atomic_long_read(&vm_zone_stat[i]);
		if (val < 0) {
			pr_warn("%s: %s %ld\n",
				__func__, zone_stat_name(i), val);
		}
	}
	for (i = 0; i < NR_VM_NODE_STAT_ITEMS; i++) {
		/*
		 * Skip checking stats known to go negative occasionally.
		 */
		switch (i) {
		case NR_WRITEBACK:
			continue;
		}
		val = atomic_long_read(&vm_node_stat[i]);
		if (val < 0) {
			pr_warn("%s: %s %ld\n",
				__func__, node_stat_name(i), val);
		}
	}
	if (write)
		*ppos += *lenp;
	else
		*lenp = 0;
	return 0;
}
#endif /* CONFIG_PROC_FS */
```

