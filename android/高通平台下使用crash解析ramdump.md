## 高通平台下使用crash解析ramdump

[具体操作参考第6点，其他都是过程记录](#6.crash解析操作方法)



### 1.下载crash工具

github官网：https://github.com/crash-utility/crash



解析64位Android kernel使用如下指令编译：

make target=ARM64

make install



### 2.解压ramdump文件

安装：

```
 apt-get install p7zip-full
```

解压7z：

```
 7z x file.7z
```

解压出来就是文件夹.



### 3.gdb依赖包

texinfo bison libncurses-dev



### 4.高通ramdump内存解析脚本适配【记录过程，不用care~】

#### （1）尝试使用github上python2版本（https://github.com/emonti/qualcomm-opensource-tools）

##### python2依赖包

enum pyelftools prettytable

##### 需要安装0.29版本的pyelftools

https://stackoverflow.com/questions/77067937/python-modulenotfounderror-no-module-named-elftools-common-py3compat



##### python2没有collections.abc模块，修改为import collections



##### local_settings报错

https://blog.csdn.net/m0_46250244/article/details/112428261



##### sched_info.py

from utils.anomalies import Anomaly

修改为 from anomalies import anomaly



##### pip报错sys.stderr.write(f"ERROR: {exc}")

curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py

python get-pip.py

【参考：https://www.cjavapy.com/article/1605/】



##### python2不太行~~~

【参考：https://www.cnblogs.com/rainey-forrest/p/12162216.html】

报错 supportid



#### （2）尝试用源码中的tools来进行解析

sxr2130p_repo/emdoor/LINUX/android/vendor/qcom/opensource/tools/linux-ramdump-parser-v2



##### python3依赖包

numpy anomalies prettytable



##### python3遇到问题

![image-20230918160608018](/Users/xayahion/Library/Application Support/typora-user-images/image-20230918160608018.png)

解决方法：重装prettytable即可



##### 直接使用crash解析还有bug😅

尝试了2个dump和vmlinux都有类似的问题...

```shell
crash ../vmlinux --kaslr 0x1b2d400000 DDRCS0_0.BIN@0x80000000,DDRCS1_0.BIN@0x200000000,DDRCS0_1.BIN@0x100000000,DDRCS1_1.BIN@0x280000000
```

![image-20230919112603976](/Users/xayahion/Library/Application Support/typora-user-images/image-20230919112603976.png)



```shell
crash ../EM-B652-V1.0.0-20230905-033443-userdebug/vmlinux --kaslr 0x222c000000 DDRCS0_0.BIN@0x80000000,DDRCS1_0.BIN@0x200000000,DDRCS0_1.BIN@0x100000000,DDRCS1_1.BIN@0x280000000
```

![image-20230919164706982](/Users/xayahion/Library/Application Support/typora-user-images/image-20230919164706982.png)



```c
#crash/tools源码
readmem(node + OFFSET(radix_tree_node_slots) + sizeof(void *) * off, KVADDR, &slot, sizeof(void *), "radix_tree_node.slot[off]", FAULT_ON_ERROR);
```



看上去是读取内存信息错误

```c
#define READ_ERRMSG      "read error: %s address: %llx  type: \"%s\"\n"

readmem {
switch (READMEM(fd, bufptr, cnt, (memtype == PHYSADDR) || (memtype == XENMACHADDR) ? 0 : addr, paddr))
case READ_ERROR:
	if (PRINT_ERROR_MESSAGE)
		error(INFO, READ_ERRMSG, memtype_string(memtype, 0), addr, type);

}

#define READMEM  pc->readmem

pc->readmem = read_ramdump;
```



查看源码，paddr不在内存起始范围内，可能是少读取了内存块

![image-20230926142057306](/Users/xayahion/Library/Application Support/typora-user-images/image-20230926142057306.png)



##### 修改为以下成功🤏（增加了DDRCS0_2.BIN和DDRCS1_2.BIN）

```shell
crash ../vmlinux-2 --kaslr=0x2eea000000 DDRCS0_0.BIN@0x80000000,DDRCS1_0.BIN@0x200000000,DDRCS0_1.BIN@0x100000000,DDRCS1_1.BIN@0x280000000,DDRCS0_2.BIN@180000000,DDRCS1_2.BIN@300000000
```



### 5.高通内存解析脚本操作方法

#### （1）下载工具链

包括aarch64-linux-android-gdb，aarch64-linux-android-nm，aarch64-linux-android-objdump

https://gitlab.com/TeeFirefly/prebuilts/-/tree/master/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin

#### （2）编辑bash脚本

```bash
ramdump=$1
vmlinux=$2
ramparse_dir=$3/ramparse.py
outdir=$4

gdb="/home/zyr/prebuilts-master-gcc-linux-x86-aarch64/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android-gdb"
nm="/home/zyr/prebuilts-master-gcc-linux-x86-aarch64/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android-nm"
objdump="/home/zyr/prebuilts-master-gcc-linux-x86-aarch64/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android-objdump"

echo $1,$2,$ramparse_dir,$4
python3 $ramparse_dir --vmlinux $vmlinux -g $gdb -n $nm -j $objdump -a $ramdump -o $outdir -x
```



#### （3）执行脚本，举例如下：

```shell
#参数1：内存dump文件目录
#参数2：vmlinux文件目录
#参数3：高通平台自带内存解析脚本
#参数4：输出文件夹名（存放解析结果）
./dumpparse.sh Port_COM77 EM-B652-V1.0.0-20230905-033443-userdebug/vmlinux linux-ramdump-parser-v2 ./out77/
```



#### （4）解析结果

输出文件夹里最重要的是dmesg_TZ.txt文件，可以根据调用栈快速定界是哪个模块的问题；

除此之外，还可以看到其他信息，如：

tasks.txt 所有进程调用栈信息；

pagetypeinfo.txt 页块信息；

memory.txt 各进程内存占用信息

mem_stat.txt 整体内存占用信息



### 6.crash解析操作方法

（1）解压ramdump文件

```shell
7z x *.7z
```

（2）获取kaslr地址

```shell
hexdump -e '16/4 "%08x " "\n"' -s 0x03f6d4 -n 8 OCIMEM.BIN
```

![image-20230926143031353](/Users/xayahion/Library/Application Support/typora-user-images/image-20230926143031353.png)

由于arm架构下大小端，组合为0000002eea000000

（3）确认ramdump的加载偏移量，借助dump_info.txt确认

![image-20230926143254155](/Users/xayahion/Library/Application Support/typora-user-images/image-20230926143254155.png)

（4）组合命令即可

```shell
crash ../vmlinux-2 --kaslr=0x0000002eea000000 DDRCS0_0.BIN@0x80000000,DDRCS1_0.BIN@0x200000000,DDRCS0_1.BIN@0x100000000,DDRCS1_1.BIN@0x280000000,DDRCS0_2.BIN@180000000,DDRCS1_2.BIN@300000000
```



### 7.参考链接

1.基于crash工具搭建分析ramdump的平台

https://blog.csdn.net/u014175785/article/details/112868957

2.高通平台RAMDUMP分析

https://www.tsz.wiki/tools/optimize/crash/qcom-ramdump/qcom-ramdump.html#_3-ramdump