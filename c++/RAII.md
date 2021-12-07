# RAII Resource Acquisition is Initialization



## 【概念】

c++的一种管理资源，避免泄露的方法。
RAII的做法是使用一个对象，在其构造时获取对应的资源，在对象声明周期内控制对资源的访问，最后在对象析构时释放构造时获取的资源



## 【背景】

程序复杂时，需要把所有new分配的内存delete掉，效率会下降

```cpp
#include <iostream> 
 
using namespace std; 
 
int main() 
 
{ 
    int *testArray = new int [10]; 
    // Here, you can use the array 
    delete [] testArray; 
    testArray = NULL ; 
    return 0; 
}
```



## 【如何使用】

把资源用类封装起来，对资源操作都封装在类的内部，通过析构函数来释放资源。

```cpp
#include <iostream> 
using namespace std; 
 
class ArrayOperation 
{ 
public : 
    ArrayOperation() 
    { 
        m_Array = new int [10]; 
    } 
 
    void InitArray() 
    { 
        for (int i = 0; i < 10; ++i) 
        { 
            *(m_Array + i) = i; 
        } 
    } 
 
    void ShowArray() 
    { 
        for (int i = 0; i <10; ++i) 
        { 
            cout<<m_Array[i]<<endl; 
        } 
    } 
 
    ~ArrayOperation() 
    { 
        cout<< "~ArrayOperation is called" <<endl; 
        if (m_Array != NULL ) 
        { 
            delete[] m_Array;  
            m_Array = NULL ; 
        } 
    } 
 
private : 
    int *m_Array; 
}; 
 
bool OperationA(); 
bool OperationB(); 
 
int main() 
{ 
    ArrayOperation arrayOp; 
    arrayOp.InitArray(); 
    arrayOp.ShowArray(); 
    return 0;
}
```

## 【典型案例】


（1）资源管理：智能指针，实现自动的内存管理；类似的还有文件打开关闭，句柄的获取和释放。
（2）状态管理：线程同步时，unique_lock或者lock_guard对互斥量mutex进行状态管理。在互斥量lock和unlock之间的代码可能会出现异常，会导致互斥量不会正确的unlock，导致线程死锁。在创建lock_guard对象时，会对mutex对象进行lock，当lock_guard对象超出作用域时，会自动对mutex对象进行unlock。



## 【总结】

核心思想是将资源或者状态与对象的生命周期绑定。




## 【参考链接】

1、C++RAII机制

https://blog.csdn.net/quinta_2018_01_09/article/details/93638251

2、c++11中lock_guard和unique_lock使用浅析

https://blog.csdn.net/guotianqing/article/details/104002449
