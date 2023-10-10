### config配置项

源码地址：./LINUX/android/out/target/product/kona/obj/kernel/msm-4.19/.config



机器获取：

https://www.cnblogs.com/AmoklauF/articles/16833046.html

adb pull /proc/config.gz 本地路径



#### 1.android内核config默认文件

【msm-4.19/arch/arm64/configs/vendor/kona-perf_defconfig】

make ARCH=arm64 vendor/kona-perf_defconfig



#### 2.make menuconfig机制

https://blog.csdn.net/lanhuazui10/article/details/130791691

Linux内核根目录下的scripts文件夹

arch/$ARCH/Kconfig文件、各层目录下的Kconfig文件

Linux内核根目录下的makefile文件、各层目录下的makefile文件

Linux内核根目录下的的.config文件、arm/$ARCH/下的config文件

Linux内核根目录下的 include/generated/autoconf.h文件



（1）scripts文件夹存放的是跟make menuconfig配置界面的图形绘制相关的文件，我们作为使用者无需关心这个文件夹的内容



（2）当我们执行make menuconfig命令出现上述蓝色配置界面以前，系统帮我们做了以下工作：

   首先系统会读取arch/$ARCH/目录下的Kconfig文件生成整个配置界面选项（Kconfig是整个linux配置机制的核心），那么ARCH环境变量的值等于多少呢？它是由linux内核根目录下的makefile文件决定的，在makefile下有此环境变量的定义：

![img](https://img-blog.csdnimg.cn/img_convert/cbb9e865d2490ea176d17f63a80280c7.gif)

或者通过 make ARCH=arm menuconfig命令来生成配置界面，默认生成的界面是所有参数都是没有值的



（2）默认配置选项存放在arch/$ARCH/configs下，对于arm来说就是arch/arm/configs文件夹：

![img](https://img-blog.csdnimg.cn/img_convert/f91c3fbf9cad01db087abaf643a4dd7a.gif)

   此文件夹中有许多选项，系统会读取哪个呢？内核默认会读取linux内核根目录下.config文件作为内核的默认选项，**我们一般会根据开发板的类型从中选取一个与我们开发板最接近的系列到Linux内核根目录下**（选择一个最接近的参考答案）cp arch/arm/configs/s3c2410_defconfig .config



### Android添加系统属性

https://blog.csdn.net/a546036242/article/details/119182420



在 Android 9.0 之后，google不推荐把厂家定制的 property 加到 /system 分区了，更推荐把厂家私有属性添加到 /vendor/build.prop 中。



./qssi12/out/target/product/qssi/system/build.prop

./qssi12/out/target/product/qssi/vendor/build.prop
