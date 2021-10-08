# Memleaks in boost::regex_match分析定位



## 一、调用栈如下：

泄漏点会在extend_stack中

![image-20210918113120816](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210918113120816.png)

异常信息可以看到Ran out of stack space try to match the regular expression，表示栈空间不足，内存

会有used_block_count作为判断，如果减到0表示空间不足，会走异常流程，不会释放内存

![image-20210918113213719](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210918113213719.png)



## 二、分析：

regex是缓存内存池的机制，所以used_block_count控制内存的申请和释放。

调用get_mem_block的函数都会将used_block_count值减1，表示获取内存，内存池减少；

对应的put_mem_block的函数应该将used_block_count值加1，表示释放内存，内存池增加



## 三、查看源码：

unwind_extra_block中并没有对used_block_count加1，所以内存池中会消耗为0，导致走到异常流程，内存还未释放，导致泄漏

![image-20210918113800234](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210918113800234.png)



## 四、结果

修改后重编后，内存泄漏解决，异常信息为正则表达式的模式串过于复杂

![image-20210918114156090](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20210918114156090.png)