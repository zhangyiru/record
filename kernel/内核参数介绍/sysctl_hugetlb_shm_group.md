```
/proc/sys/vm/sysctl_hugetlb_shm_group
```

 This writable file contains a group ID that is allowed to allocate memory using huge pages. 

 If a process has a filesystem group ID or any supplementary group ID that matches this group ID, then it can make huge-page allocations without holding the CAP_IPC_LOCK capability;



ulimit -l 65536

当分配共享内存为8UL\*1024\*1024时，
[zyr@localhost ~]$ gcc hugepage-shm.c 
[zyr@localhost ~]$ ./a.out 
shmid: 0xb
shmaddr: 0x7f4bace00000
Starting the writes:
.
Starting the Check...Done.



当分配共享内存为128UL\*1024\*1024时，
[zyr@localhost ~]$ gcc hugepage-shm.c 
[zyr@localhost ~]$ ./a.out 
shmget: Operation not permitted



只有被写在sysctl_hugetlb_shm_group里的用户才能使用大页共享内存。

但是目前当分配的共享内存大小小于限制内存的数时可以使用，这个功能需要废弃

https://lore.kernel.org/linux-mm/d14533d8-eb49-9ac0-2f46-a1c452e82f0e@oracle.com/