# __builtin_expect

1、概念

指令是gcc引入的，作用是**允许程序员将最有可能执行的分支告诉编译器**

这个指令的写法为：`__builtin_expect(EXP, N)`。意思是：EXP==N的概率很大。



2、使用方法

一般的使用方法是将`__builtin_expect`指令封装为`likely`和`unlikely`宏。这两个宏的写法如下.

```cpp
#define likely(x) __builtin_expect(!!(x), 1) //x很可能为真       
#define unlikely(x) __builtin_expect(!!(x), 0) //x很可能为假
```

内核中的 likely() 与 unlikely()。首先要明确：

```c
if(likely(value))  //等价于 if(value)
if(unlikely(value))  //也等价于 if(value)
```

gcc编译的指令会先预取likely(x)后语句的指令。系统运行时就减少重新取指。



3、另外tcmalloc中也用到了这个方法，封装成了PREDICT_TRUE和PREDICT_FALSE

```
#if defined(__GNUC__)
#define PREDICT_TRUE(x) __builtin_expect(!!(x), 1)
#define PREDICT_FALSE(x) __builtin_expect(!!(x), 0)
#else
#define PREDICT_TRUE(x) (x)
#define PREDICT_FALSE(x) (x)
#endif
```