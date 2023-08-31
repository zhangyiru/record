## vim+cscope使用指南

https://developer.aliyun.com/article/686709



### 1、创建cscope数据库

在代码库的根目录下执行 cscope -Rbq

将产生cscope.out、cscope.in.out和cscope.po.out三个文件



-b：仅构建交叉引用(cross-reference)文件，即数据库，然后退出，而不会进入下面的交互界面：
![image](https://yqfile.alicdn.com/b03525501c43cbdccca603f7e803b4b2fcd19c32.png)
-R：递归解析所有的子目录。
-q：通过倒排索引加速符号的查找过程。该选项会导致cscope额外产生cscope.in.out和cscope.po.out两个文件。



### 2、add(增加一个新的cscope数据库/连接)

:cs add {file|dir} [pre-path] [flags]



把cscope.out加进来，-C指定了搜索时忽略大小写

:cs add ../cscope.out /home/xhb/code/os/test-leveldb/leveldb-master -C



如果当前目录下有cscope.out文件，则打开vim时会自动加载该数据库

通过修改/etc/vimrc文件来自动加载cscope

https://www.codeleading.com/article/95206124467/



### 3、find

cs find {querytype} {name}



```
0或s：查找这个(指name参数，下同)C符号。
1或g：查找这个定义。
2或d：查找被这个函数调用的函数。
3或c：查找调用该函数的函数。
4或t：查找这个文本字符串。
6或e: 查找这个egrep的pattern。
7或f：查找这个文件。
8或i：查找#include了这个文件的所有文件。
```



```
# 当前光标停留在WriteBlock上
:cs find t <cword>
# 效果等同于:cs find t WriteBlock
```



## 相关问题

1、提示缺少curses.h头文件

安装ncurses-devel

https://stackoverflow.com/questions/18277192/how-to-install-curses-h-header-fileor-the-package-including-itin-linux



2、CSCOPE ERROR: E429: File "path/to/file.c" does not exist

https://groups.google.com/g/vim_use/c/Q8st3tN-wOo

在vimrc文件中增加set cscoperelative /etc/vimrc

