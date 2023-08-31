### umount代码流程

```c
ksys_umount
    user_path_mountpoint_at
    	path_mountpoint(struct nameidata *nd, unsigned flags, struct path *path)

struct nameidata *nd
	struct filename	*name;
		const char *name
```



```c
crash> struct nameidata -x -o | grep filename
  [0xb8] struct filename *name;
```



### 抓取umount的路径

```shell
echo 0 > /sys/kernel/debug/tracing/events/kprobes/enable
echo 'p:path_mountpoint path_mountpoint name=+0(+0(+0xb8(%x0))):string' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable
echo > /sys/kernel/debug/tracing/trace
```

