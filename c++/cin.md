## cin.get

cin.get有多个函数，函数入参个数不同

1、cin.get(ch)

作用是从输入流中读取一个字符，赋给字符变量ch

2、cin.get(字符指针, 字符个数n, 终止字符)

作用是从输入流中读取n-1个字符，赋值给指定的字符数组a，如果在读取n-1字符之前遇到指定的终止字符，则提前结束读取。

```c++
cin.get(a,n,'');
```



源码实现：

https://code.woboq.org/gcc/libstdc++-v3/include/std/istream.html

参考：

https://www.cxybb.com/article/qq_40707451/86416110