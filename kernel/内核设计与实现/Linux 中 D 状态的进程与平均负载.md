# Linux 中 D 状态的进程与平均负载

- 平均负载的概念





![image-20210915213430711](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210915213430711.png)

第一行有一个 load average 字段，由三个数字表示，依次表示过去 1 分钟、5 分钟、15 分钟的平均负载（Load Average）

平均负载并不是指 CPU 的负载，毕竟系统资源并不是只有 CPU 这一个。简单来看，平均负载是指单位时间内，系统处于`可运行状态`和`不可中断状态`的平均进程数，也就是平均活跃进程数。

源码见 https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c



从直观的角度理解，如果平均负载为 2，在 4 核的机器上，表示有 50% 的 CPU 空闲；在 2 核的机器上，表示刚好没有 CPU 空闲，如果是单核的机器，那表明 CPU 竞争激烈，有一半的进程竞争不到 CPU



- D 状态的进程是什么

TASK_UNINTERRUPTIBLE 在 top 命令中显示为 D 标记。

顾名思义，处于 TASK_UNINTERRUPTIBLE 状态的进程不能被信号唤醒，只能由 wakeup 唤醒。

既然 TASK_UNINTERRUPTIBLE 不能被信号唤醒，自然也不会响应 kill 命令，就算是必杀 kill -9 也不例外。

“不可中断”指的是当前正处于内核中的关键流程，不可以被打断，比较常见的是读取磁盘文件的过程中被打断去处理信号，读到的内容就是不完整的。



从侧面来看，磁盘的驱动是工作在内核中的，如果磁盘出现了故障，磁盘读不到数据，内核就陷入了很尴尬的两难局面，这个锅只能自己扛着，将进程标记为不可中断，谁让磁盘驱动是跑在内核中呢。

需要注意的是 D 状态的进程会增加系统的平均负载



- 如何编写内核模块模拟 D 状态进程

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <asm/uaccess.h>

#define DEVICE_NAME "mychardev"
int major_num;
struct class *my_class_class;
static char msg[] = "helloworld\n";
char *p;

static int my_device_open(struct inode *inode, struct file *file) {
  printk("%s\n", __func__);
  static int counter = 0;
  if(counter==2)
  {
        __set_current_state(TASK_UNINTERRUPTIBLE); // change state to sleep
        schedule();
  }
  p = msg;
  ++counter;
  return 0;
}

static int my_device_release(struct inode *inode, struct file *file) {
  printk("%s\n", __func__);
  return 0;
}

static ssize_t my_device_read(struct file *file,
                              char *buffer, size_t length, loff_t *offset) {
  printk("%s %u\n", __func__, length);

  int read_bytes = 0;
  if(*p==0) return 0;
  while(length && *p)
  {
        put_user(*(p++),buffer++);
        length--;
        read_bytes++;
  }
  return read_bytes;
}

static ssize_t my_device_write(struct file *file,
                               const char *buffer, size_t length, loff_t *offset) {
  printk("%s %u\n", __func__, length);
  return length;
}

struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = my_device_open,
    .release = my_device_release,
    .read = my_device_read,
    .write = my_device_write,
};

static int __init my_module_init(void) {
  printk("my module loaded\n");
  // register_chrdev 函数的 major 参数如果等于 0，则表示采用系统动态分配的主设备号
  major_num = register_chrdev(0, DEVICE_NAME, &fops);
  if (major_num < 0) {
    printk("Registering char device failed with %d\n", major_num);
    return major_num;
  }
  // 接下来使用 class_create 和 device_create 自动创建 /dev/mychardev 设备文件
  my_class_class = class_create(THIS_MODULE, DEVICE_NAME);
  if(my_class_class==NULL)
  {
        return -1;
  }
  device_create(my_class_class, NULL, MKDEV(major_num, 0), NULL, DEVICE_NAME);
  return 0;
}

/**
 * 内核模块卸载
 */
static void __exit my_module_exit(void) {
  if(my_class_class!=NULL)
  {
        device_destroy(my_class_class, MKDEV(major_num, 0));
        class_destroy(my_class_class);
        unregister_chrdev(major_num, DEVICE_NAME);
        printk("my module unloaded\n");
  }
}

module_init(my_module_init);
module_exit(my_module_exit);
```



- Linus 对 D 状态进程的看法

https://lore.kernel.org/lkml/Pine.LNX.4.33.0208011315220.12103-100000@penguin.transmeta.com/