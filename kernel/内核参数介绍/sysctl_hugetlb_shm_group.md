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