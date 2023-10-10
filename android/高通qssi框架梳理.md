QSSI 是 Qualcomm Single System Image 的缩写，并且高通平台从Android Q开始支持。高通的官方文档对此的解释是引入QSSI的概念是为了解决Android碎片化问题，把system.img和vendor.img进一步拆分。

![img](https://cdn.nlark.com/yuque/0/2023/png/29793833/1687141699511-728b22cb-730c-4ed8-8e52-2a92b8398be6.png)

简单来说，QSSI就是高通基于Google发布的AOSP(安卓开源代码项目)构建的一套自己的通用公共system image代码分区，它将针对不同的高通硬件平台（chipset）做兼容，即可以使用一个system image代码同时兼容多个硬件平台的vendor image代码，并且QSSI代码和vendor代码均可以独立更新迭代，但QSSI代码需要向下兼容老的vendor代码。

![img](https://cdn.nlark.com/yuque/0/2023/png/29793833/1687141699353-c505c4ea-666d-4f65-98d6-5ccab3e941aa.png)

高通提供了一种检查QSSI与vendor分区代码兼容性的工具---QIIFA，它是通过指定的接口进行call和callback，通过system代码和vendor代码的调用信息进行判断是否匹配，若不匹配则生成相关log日志。

![img](https://cdn.nlark.com/yuque/0/2023/png/29793833/1687141699357-cde94af1-3e64-453d-bfe1-6d4a4900c534.png)

如上图，以HIDL接口为例，HIDL是基于hwbinder实现的，即在vendor中hal库以binder服务的形式注册到servicemanager当中，system分区代码要调用vendor中的接口时，需要先获取servicemanager中的hal-binder服务，获取之后才可以调用vendor中的接口，此时vendor-hal属于server端，system属于client端。

注意：

1、QSSI可以单独编译；

2、vendor必须依赖QSSI才可以编译。

QSSI编译也和Android原生编译有差别，其差别如下:

![img](https://cdn.nlark.com/yuque/0/2023/png/29793833/1687141699506-77e382ad-98d5-4982-beed-a068285799cf.png)

1、高版本Android非QSSI特性的整体编译流程高版本Android非QSSI特性的编译流程,依然和以前的版本Android编译变化不大，通常是如下的步骤(这个也是读者最为熟悉的了):

source build/envsetup.sh

lunch xx-userdebug

make

这里需要注意的的是通用版本的Android还是可以直接通过make相关的分区进行直接编译的，譬如make superimage或者直接执行make编译

2、具有QSSI特性Android高版本整体编译流程

通过前面看到QSSI特性的固件编译流程也和通用版本的有一定的区别，这里的编译分为两种模式，第一种Android的标准编译模式，另外一种就是高通提供的编译脚本。

2.1、 通过Android内置make命令编译

**初始化编译环境**

source build/envsetup.sh

**编译 system.img**

lunch qssi-userdebug

make target-files-package

**编译除system.img外的其他img**

lunch xx-userdebug

make target-files-package

2.2、 高通提供的build.sh脚本进行编译

编译所有img，包括system和其它img

source build/envsetup.sh

lunch XX-userdebug

./build.sh dist -j32

**编译system.img，产物在qssi目录下**

source build/envsetup.sh

lunch xx-userdebug

./build.sh dist qssi_only -j32

编译super.img

source build/envsetup.sh

lunch xx-userdebug

./build.sh dist merge_only -j32

编译其它img，例如vendorimage，如果不指定会编译其它所有img，产物在XX目录下

source build/envsetup.sh

lunch xx-userdebug

./build.sh vendorimage dist target_only -j32