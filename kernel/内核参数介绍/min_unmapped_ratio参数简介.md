/proc/sys/vm/min_unmapped_ratio

min_unmapped_ratio:

This is available only on NUMA kernels.

This is a percentage of the total pages in each zone. Zone reclaim will
only occur if more than this percentage of pages are in a state that
zone_reclaim_mode allows to be reclaimed.

If zone_reclaim_mode has the value 4 OR'd, then the percentage is compared
against all file-backed unmapped pages including swapcache pages and tmpfs
files. Otherwise, only unmapped pages backed by normal files but not tmpfs
files and similar are considered.

The default is 1 percent.



zone page frame allocator

https://www.cnblogs.com/LoyenWang/p/11708255.html

概念：

仅用于NUMA系统
参数值表示每个zone中总页面（managed_pages）的百分比。
只有当zone_reclaim_mode设置为允许回收的页面超过这个百分比时，zone回收才会发生



/proc/sys/vm/zone_reclaim_mode：

用来控制内存zone回收模式，在内存分配中，用来管理当一个内存区域内部的内存耗尽时，
是从其内部进行内存回收来满足分配还是直接从其它内存区域中分配内存

```shell
echo 0 > /proc/sys/vm/zone_reclaim_mode
# 意味着关闭zone_reclaim模式，可以从其他zone或NUMA节点回收内存

echo 1 > /proc/sys/vm/zone_reclaim_mode
# 表示打开zone_reclaim模式，这样内存回收只会发生在本地节点内

echo 2 > /proc/sys/vm/zone_reclaim_mode
# 在本地回收内存时，可以将cache中的脏数据写回硬盘，以回收内存。

echo 4 > /proc/sys/vm/zone_reclaim_mode
# 可以用swap方式回收内存。
```

如果zone_reclaim_mode的值为4，内核会比较所有的file-backed（有文件背景的页面）和匿名映射页（不以文件背景的页面），包括swapcache（Memory that once was swapped out, is swapped back in but still also is in the swap file.）占用的页以及tmpfs文件的总内存使用是否超过这个百分比。

其他设置的情况下，只比较基于一般文件的未映射页，不考虑其他相关页

**页面回收(reclaim)**

- 有**文件背景**的数据实际上就是page cache，但page cache不能无限增加，不能说慢慢的所有文件都缓存到内存了。肯定要有一个机制，让不常用的文件数据从page cache刷出去。内核中有一个水位控制的机制，在系统内存不够用的时候，会触发页面回收。
- 对于没有文件背景的页面即**匿名页**，比如堆、栈、数据段，如果没有swap分区，不能与磁盘交换，就要常驻内存了。但是常驻内存的话，就会吃内存，可以通过给硬盘搞一个swap分区或硬盘中创建一个swap文件让匿名页也能交换到磁盘上。可认为是为匿名页伪造的文件背景。swap分区或swap文件实际上最终是到达了增大内存的效果。当然，如果频繁交换的话，被交换出去的数据的访问就会慢一些，因为要有IO操作了。

```c
#ifdef CONFIG_NUMA
static void setup_min_unmapped_ratio(void)
{
        pg_data_t *pgdat;
        struct zone *zone;

        for_each_online_pgdat(pgdat)
                pgdat->min_unmapped_pages = 0;

        for_each_zone(zone)
                zone->zone_pgdat->min_unmapped_pages += (zone->managed_pages *
                                sysctl_min_unmapped_ratio) / 100;
}


int sysctl_min_unmapped_ratio_sysctl_handler(struct ctl_table *table, int write,
        void __user *buffer, size_t *length, loff_t *ppos)
{
        int rc;

        rc = proc_dointvec_minmax(table, write, buffer, length, ppos);
        if (rc)
                return rc;

        setup_min_unmapped_ratio();

        return 0;
}
```

alloc_pages -》

__alloc_pages-》

get_page_from_freelist -》

node_reclaim-》

//计算在回收模式下需要回收多少pagecache页

//如果存在更多未映射页面，则节点回收将变为活动状态

//文件I/O需要一小部分未映射的文件备份页面，否则，如果节点被过度分配，

//文件I/O读取的页面将立即被抛出

//如果未映射的文件备份页面使用的节点少于指定百分比，则我们不会回收

if node_pagecache_reclaimable(pgdat) > pgdat->min_unmapped_pages

__node_reclaim -》

shrink_node



node_page_state(pgdat, NR_FILE_PAGES)-》

node_page_state_pages(pgdate,item)-》

atomic_long_read(pgdat->vm_stat[item])

cat /proc/vmstat -> nr_file_pages



