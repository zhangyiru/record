## 一、调用栈结果如下：

模式串：(?xi)|?+#

字符串：*

在unwind_alts中出现断言错误

![image-20211022103548983](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20211022103548983.png)



## 二、打印分析

打印发现此时的状态转移到syntax_element_toggle_case



由于模式串中存在"i"，所以“|”后的表达式要沿用具备不区分大小写的状态，所以要增加syntax_element_toggle_case，但是parse_repeat中没有对syntax_element_toggle_case这个状态的判断



社区给出的修复补丁是直接将断言修改为判断

![image-20211022103850608](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20211022103850608.png)

https://github.com/boostorg/regex/commit/b10c089aba7fbeb21fc0cdd193ea4172090c59bb



## 三、结果验证

合入补丁重编后问题解决，未存在其他问题

![image-20211022103141730](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20211022103141730.png)