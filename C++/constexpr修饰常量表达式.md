const：

1、修饰变量可读，当作为函数形参时是只读变量

2、修饰常量，下面可以使用常量count来定义静态数组

```c++
void func(const int num)
{
    const int count = 24;
    int array[num];            // error，num是一个只读变量，不是常量
    int array1[count];         // ok，count是一个常量

    int a1 = 520;
    int a2 = 250;
    const int& b = a1;
    b = a2;                         // error
    a1 = 1314;
    cout << "b: " << b << endl;     // 输出结果为1314
}
```



constexpr:

常量表达式：指的就是由多个（≥1）常量（值不会改变）组成并且在编译过程中就得到计算结果的表达式

非常量表达式只能在运行阶段计算出结果，但是常量表达式的计算往往发生在程序的编译阶段，这可以极大提高程序的执行效率，因为表达式只需要在编译阶段计算一次，节省了每次程序运行时都需要计算一次的时间。



1、当定义常量时，const和constexpr是等价的

2、对于c++内置类型的数据，可以直接用constexpr修饰，但是自定义的数据类型（struct或者class），不能直接使用constexpr

```c++
struct Test
{
    int id;
    int num;
};

int main()
{
    constexpr Test t{ 1, 2 };
    constexpr int id = t.id;
    constexpr int num = t.num;
    
    t.num += 100;// error，不能修改常量
    cout << "id: " << id << ", num: " << num << endl;

    return 0;
}
```



常量表达式函数：普通函数/类成员函数，类构造函数，模板函数

1、修饰函数

函数必须要有返回值，并且返回必须是常量表达式；

函数在定义前必须要有对应的声明语句；

```c++
#include <iostream>
using namespace std;

constexpr int func1();
int main()
{
    constexpr int num = func1();	// error
    return 0;
}

constexpr int func1()
{
    constexpr int a = 100;
    return a;
}
```

整个函数的函数体中，不能出现非常量表达式之外的语句（using 指令、typedef 语句以及 static_assert 断言、return 语句除外）



2、修饰模板函数

如果 constexpr 修饰的模板函数实例化结果不满足常量表达式函数的要求，则 constexpr 会被自动忽略，即该函数就等同于一个普通函数



3、修饰构造函数

构造函数的函数体必须为空，并且必须采用初始化列表的方式为各个成员赋值

constexpr struct Person p1("luffy", 19);



参考：https://subingwen.cn/cpp/constexpr/#2-3-修饰构造函数

