## 一、基础知识

1、=表达式在判断中的处理，例如if(a=b+c)

假设a = 1, b = 2, c = 1

判断表达式为If(a=b+c) 

a=b 表示将b的值赋值给a，a=b值为2，if(a=b+c) -> if(3)



2、函数参数的传递效率，传指针不一定比传值效率高

```c++
#include <iostream>
using namespace std;

void swap(int a,int b)
{
	int temp = a;
	a = b;
	b = temp;
}
int main()
{
	int a = 0, b = 1;
	cout << a << " " << b << endl;
	swap(a,b);
	cout << a << " " << b << endl;
	return 0;
}
```

传递效率，是说**调用被调函数的代码将实参传递到被调函数体内的过程**，正如上面代码中，这个过程就是函数main()中的 a、b 传递到函数swap()中的过程。这个效率不能一概而论。

对于内建的 int char、short、long、float 等4字节或以下的数据类型而言，实际上传递时也只需要传递 1－4 个字节，而使用指针传递时在 32 位 cpu 中传递的是 32 位的指针，4 个字节，都是一条指令，这种情况下值传递和指针传递的效率是一样的，

而传递 double、long long 等 8 字节的数据时，在 32 位 cpu 中，其传值效率比传递指针要慢，因为 8 个字节需要 2 次取完。而在 64 位的 cpu 上，传值和传址的效率是一样的。

再说引用传递，这个要看编译器具体实现，引用传递最显然的实现方式是使用指针，这种情况下与指针的效率是一样的，而有些情况下编译器是可以优化的，采用直接寻址的方式，这种情况下，效率比传值调用和传址调用都要快，与采用全局变量方式传递的效率相当

参考：https://www.cnblogs.com/multhree/p/10498183.html



3、static_cast将整数转浮点数，会有.0

![image](https://user-images.githubusercontent.com/15630061/136392961-5b12cce9-1768-49f3-8f45-f505c6341a65.png)

将int强制转换成double是没有风险的，是一种简单类型转换为复杂类型，但是将复杂类型转换为简单类型，就有丢失精度的风险了

4、异常与捕获

捕获方式：不带引用，带引用，带右值引用

推荐带引用

```c++
#include <iostream>
using namespace std;

class Base {
public:
	Base() {
		cout << "Base's constructor" << endl;
	}

	Base(const Base& rb) {
		cout << "Base's copy constructor" << endl;
	}

	virtual void print() {
		cout << "Base" << endl;
	}
};

class Derived :public Base {
public:
	Derived() {
		cout << "Derived's constructor" << endl;
	}

	Derived(const Derived& rd):Base(rd) {
		cout << "Derived's copy constructor" << endl;
	}

	virtual void print() {
		cout << "Derived" << endl;
	}
};

void throwFunc() {
	Derived d;
	throw d;
}

int main() {
	try {
		throwFunc();
	}
	catch (Base b) {
		cout << "Base catched" << endl;
		b.print();
	}
	catch (Derived d) {
		cout << "Derived catched" << endl;
		d.print();
	}
	cout << "---------------" << endl;
	try {
		throwFunc();
	}
	catch (Base& b) {
		cout << "Base catched" << endl;
		b.print();
	}
	catch (Derived& d) {
		cout << "Derived catched" << endl;
		d.print();
	}
}
```

输出：

![image](https://user-images.githubusercontent.com/15630061/136393174-bf7d718f-df5b-46dc-b485-166ce40a0cf3.png)

值传递会复制两次，一次传递给 catch,另一次抛出时复制

（1）程序中在函数throwFunc()中构造对象d，先后分别调用基类Base和派生类Derived的构造函数完成对象d的初始化，分别输出Base’s constructor与Derived’s constructor；<br/>
（2）C++标准要求被作为异常抛出的对象必须被拷贝复制，导致异常对象d在离开作用域时，触发一次临时对象的拷贝构造，程序输出从结果来看，先后调用了基类Base的拷贝构造函数和派生类Derived的拷贝构造函数，分别输出Base’s copy constructor与Derived’s copy constructor；<br/>
（3）按引用捕获异常比按值捕获异常更加高效。分隔线以上按值捕获异常，导致对象d在传递时再次被拷贝一次，输出Base’s copy constructor，降低了系统效率，使用引用捕获异常可以避免额外的拷贝操作；<br/>
（4）使用引用捕获异常，可以通过基类对象实现虚函数的虚调用，在运行时提现多态性。<br/>



throw d被Base& b捕获是由于顺序问题。<br/>

如果基类和子类都被做为异常捕获，则子类的catch代码块必须出现在基类之前。<br/>
如果把基类放在前面，则子类的catch代码块永远都不会被调用。<br/>

<br/>

参考：

1、捕获基类与子类的异常<br/>

https://blog.csdn.net/shltsh/article/details/46039417

2、使用引用捕获异常<br/>

https://blog.csdn.net/K346K346/article/details/81461007

<br/>

5、char a[], char a[5]的区别

![image](https://user-images.githubusercontent.com/15630061/136393211-ed5f0b48-0439-41f6-a893-544044fecce8.png)

char a[5]是存放在栈上，可以通过指针去访问和修改数组内容；赋值是在运行时确定的；

char b[] 虽然没有指明字符串的长度，但是系统已经开好了空间，大小为6，strlen(b)是不计'\0'



6、在一文件中static声明一变量，在另一文件中extern的访问问题

static是静态存储类型，属于局部变量，只能用于同一个函数内，在其他函数内使用是错误的。

extern是外部存储类型，属于全局变量，可以用于从它定义开始的后序所有函数内。

假设a.c中定义static int A;在b.c中，不能引用a.c中的A，但是可以用A做变量名，虽然是同名但是不同变量

进一步地说，就是a.c和b.c中都定义了A，在引用时，a.c只能引用自己范围内的A，b.c也只能引用自己范围内的A，不会引用到另个文件的



7、override关键字

override关键字可以避免派生类中忘记重写虚函数的错误。

显式的在派生类中声明，哪些成员函数需要被重写，如果没被重写，则编译器会报错。

参考：https://www.cnblogs.com/xinxue/p/5471708.html

override关键字需要编译期支持c++11，如果使用gcc编译器，需要加入命令行参数-std=c++11

参考：https://blog.csdn.net/liyuanbhu/article/details/43816371



8、explicit与重新定义继承函数

explicit关键字只能用于修饰只有一个参数的类构造函数，作用是表明该构造函数是显式的，而非隐式的。

对应的另一个关键字是implicit，类构造函数默认情况下声明为implicit。

```c++
class CxString  // 没有使用explicit关键字的类声明, 即默认为隐式声明  
{  
public:  
    char *_pstr;  
    int _size;  
    CxString(int size)  
    {  
        _size = size;                // string的预设大小  
        _pstr = malloc(size + 1);    // 分配string的内存  
        memset(_pstr, 0, size + 1);  
    }  
    CxString(const char *p)  
    {  
        int size = strlen(p);  
        _pstr = malloc(size + 1);    // 分配string的内存  
        strcpy(_pstr, p);            // 复制字符串  
        _size = strlen(_pstr);  
    }  
    // 析构函数这里不讨论, 省略...  
};  
  
    // 下面是调用:  
  
    CxString string1(24);     // 这样是OK的, 为CxString预分配24字节的大小的内存  
    CxString string2 = 10;    // 这样是OK的, 为CxString预分配10字节的大小的内存  
    CxString string3;         // 这样是不行的, 因为没有默认构造函数, 错误为: “CxString”: 没有合适的默认构造函数可用  
    CxString string4("aaaa"); // 这样是OK的  
    CxString string5 = "bbb"; // 这样也是OK的, 调用的是CxString(const char *p)  
    CxString string6 = 'c';   // 这样也是OK的, 其实调用的是CxString(int size), 且size等于'c'的ascii码  
    string1 = 2;              // 这样也是OK的, 为CxString预分配2字节的大小的内存  
    string2 = 3;              // 这样也是OK的, 为CxString预分配3字节的大小的内存  
    string3 = string1;        // 这样也是OK的, 至少编译是没问题的, 但是如果析构函数里用free释放_pstr内存指针的时候可能会报错, 完整的代码必须重载运算符"=", 并在其中处理内存释放  
```

两点需要注意：

（1）CxString string2 = 10 这句为什么可以？

在c++中，如果构造函数只有一个参数时，在编译时就会有一个缺省的转换操作：将该构造函数对应数据类型的数据转换为该类对象，即将整型10转换为CxString类对象，等同于以下操作：

CxString string2(10); or CxString temp(10); CxString string2 = temp;

（2）_size代表的是字符串内存分配的大小，那么CxString string2 = 10和CxString string6 = 'c';就会产生疑惑

使用explicit关键字防止类构造函数的隐式自动转换。

explicit只对有一个参数的类构造函数有效，如果类构造函数参数超过2个时，是不会产生隐式转换的，那么explcit也就失效了（当除了第一个参数以外的其他参数都有默认值时，explicit依然有效）

effective c++中说：被声明为explicit的构造函数通常比其non-explicit兄弟更受欢迎。因为它们禁止编译器执行非预期（往往也不被期望）的类型转换。除非我有一个好理由允许构造函数被用于隐式类型转换，否则我会把它声明为explicit，鼓励大家遵循相同的政策。

参考：https://www.cnblogs.com/rednodel/p/9299251.html



9、虚函数参数是否可用缺省值（涉及静态绑定&动态绑定）

```c++
 
#include <iostream>
 
using namespace std;
 
class A  
{ 
public: 
    virtual void Fun(int number = 10)  
    {  
        cout << "A::Fun with number " << number;  
    }    
};  
  
   
  
class B: public A    
{    
public:  
    virtual void Fun(int number = 20)
    {   
        cout << "B::Fun with number " << number << endl;  
    }  
};  
    
  
int main()    
{ 
    B b; 
    A &a = b; 
    a.Fun(); 
 
    return 0;
}  
```

1、多态

A &a = b；表示a是b对象的引用，所以输出的前半部分是B::Fun with number

2、静态绑定 & 动态绑定

静态绑定和动态绑定，与静态类型和动态类型相关。

静态类型是指编译时类型；动态类型是指运行时类型。基类类型的指针和引用可以绑定到派生类型的对象，这时静态类型是基类引用（或者指针），动态类型是派生类引用（或者指针）。

```c++
struct C {
public:
    virtual void f(int a = 7)
    {
        cout<<"C: "<<a<<endl;
    }
};
struct D: public C {
public:
    void f(int a)
    {
        cout<<"D: "<<a<<endl;
    }

};

int main()
{
    D* pd = new D;
    C* pc = pd;
    pc->f();  // OK, calls pa->D::f(7)
    pd->f(); // error: wrong number of arguments for D::f()
    return 0;
}
```

拆解程序：

1、定义一个D类型的pd对象，此时pd的静态类型是D类型指针，动态类型也是D类型指针。

2、定义一个C类型的pc对象指向pd对象，此时pc的静态类型是C类型指针，动态类型是D类型指针。

动态绑定是指针延迟到运行时才选择运行那个函数，即基于引用或指针绑定的对象的基础类型选择运行哪个虚函数

3、pc->f() 这里调用的是D的成员函数f()，被调用的是与pc的动态类型相对应的函数，即动态绑定



重点：

c++中，虽然虚函数的调用是通过动态绑定来确定的，但是虚函数的缺省参数却是通过静态绑定通过的。

上面第一段程序中，a的静态类型是A的引用，动态类型是B的引用，当a调用虚函数Fun()时，根据动态绑定原则，调用的是B的成员函数Fun()，而对于虚函数的缺省参数，根据静态绑定原则，将number确定为A中给出的缺省值10。



参考：https://blog.csdn.net/qtyl1988/article/details/37604011



10、void* 与nullptr

void*表示任意类型的指针，但是不一定为空

nullptr是c++11引入的新标准，使用nullptr来指向空指针；NULL专门用来表示数值0（c语言不通）

![image](https://user-images.githubusercontent.com/15630061/136393312-bc362d82-5fe3-49a5-87ba-67c30899135c.png)

a定义为空指针，直接取值会报错，赋值前后地址不变；

b赋值为NULL，输出为0；

c定义为void*指针，指向a



11、x为一表达式，#x输出？

#将参数变成字符串，##是连接作用



12、未inline声明的函数一定不会被展开吗？

关于内联说法正确的是（BD）

A、含有inline的函数一定会被内联展开；

B、在内联函数中，所有函数定义中的局部静态对象在所有翻译单元间共享

C、未包含inline的函数一定不被内联展开  //类成员函数默认展开

D、函数的内联替换会避免函数的调用开销（传递实参并返回结果），但可能导致更大的执行文件，因为函数体必须被复制多次



13、condition-variable与mutux，共享锁

std::mutex和std::shared_mutex等锁对象提供lock和unlock来加锁，解锁。但是直接调用这两种方法容易造成执行不到unlock的错误。

使用std::lock_guard和std::unique_lock来加解锁。



14、表达式中运算符的优先级

a = b + c & d

\+ > & > =

-> a = ( ( b + c ) & d )



15、拷贝构造函数与移动构造函数，定义了拷贝构造函数，编译器会自动生成移动构造函数吗？反之呢？

移动构造函数如果没有通过=default生成的话，只有在以下三种情况都符合时才会自动生成

1、没有自定义拷贝构造函数

2、没有自定义operator=

3、没有自定义析构函数



当类中没有定义拷贝构造函数时，编译器会默认提供一个拷贝构造函数，进行成员变量之间的拷贝（浅拷贝）

使用浅拷贝时，当同类型的类间相互赋值时会导致运行时double free错误，原因是默认拷贝构造函数是浅拷贝，拷贝者和被拷贝者是同一地址，当其中一个拷贝类执行析构函数释放堆空间后，另一个拷贝类也执行析构函数时会造成重复释放，程序运行崩溃。

解决方法是为拷贝类的指针变量分配空间，实现深度拷贝（拷贝者和被拷贝者指向不同地址）

深浅拷贝：https://blog.csdn.net/qq_29344757/article/details/76037255



左值与右值：左值是变量，右值不能被赋值

左值是指表达式结束后依然存在的持久化对象，右值是指表达式结束时就不再存在的临时对象。

所有的具名变量或者对象都是左值，而右值不具名



左值引用：通过&来获得；

右值引用：为了支持移动操作，通过&&来获得右值引用；延长右值的生命期

```c++
int i = 42;
int &r = i;       //正确，r引用i
int &&rr = i;     //错误，不能将一个右值绑定到一个左值（变量）上
int &r2 = i*42;   //错误，不能将一个左值绑定到一个右值上
const int &r3 = i*42; //正确，可以将一个const的引用绑定到一个右值上
int &&rr2 = i*42;    //正确，将rr2绑定到一个乘法结果上
int &&rr2 = 42;    //正确，字面值常量是一个右值
```

总结：

1、右值不能绑定到左值上；右值可以绑定到一个结果上；右值可以绑定到字面量常量

2、左值不能绑定到右值上；可以将const引用绑定到右值上

std::move是将对象的状态或者所有权从一个对象转移到另一个对象，只是转移，没有内存的搬迁或者内存拷贝，用于将左值引用转换为右值引用

```c++
//摘自https://zh.cppreference.com/w/cpp/utility/move
#include <iostream>
#include <utility>
#include <vector>
#include <string>
int main()
{
    std::string str = "Hello";
    std::vector<std::string> v;
    //调用常规的拷贝构造函数，新建字符数组，拷贝数据
    v.push_back(str);
    std::cout << "After copy, str is \"" << str << "\"\n";
    //调用移动构造函数，掏空str，掏空后，最好不要使用str
    v.push_back(std::move(str));
    std::cout << "After move, str is \"" << str << "\"\n";
    std::cout << "The contents of the vector are \"" << v[0]
                                         << "\", \"" << v[1] << "\"\n";
}
```

![image](https://user-images.githubusercontent.com/15630061/136393389-09bcd8ce-4dc4-4ade-939f-03129b67eeef.png)

通过移动构造，b指向a的资源，a不再拥有资源（可以是动态申请的内存，网络链接，打开的文件，或者是上面的代码段中的string）

移动后再访问a的行为是未定义的，比如，资源是动态内存，再次访问可能是空指针；如果是string，a的资源为空字符串，所以上面输出str为空



参考：

1、c++右值引用

https://zhuanlan.zhihu.com/p/94588204



16、ostringstream用法与string， const char*，及它们的.str()，.c_str()

1、ostringstream与str()

实现任意类型转string

头文件：#include <sstream>

ostringstream oss;

oss << str;

string str = oss.str();



2、string与c_str()

c_str()： 生成一个const char*指针，指向以空字符为终止的数组

```c++
//标准库的string类提供了三个成员函数来从一个string得到c类型的字符数组
//主要介绍c_str
//c_str()：生成一个const char*指针，指向以空字符终止的数组。
//这个数组应该是string类内部的数组
#include <iostream>
//需要包含cstring的字符串
#include <cstring>
using namespace std;
 
int main()
{
	//string-->char*
	//c_str()函数返回一个指向正规C字符串的指针, 内容与本string串相同
 
	//这个数组的数据是临时的，当有一个改变这些数据的成员函数被调用后，其中的数据就会失效。
	//因此要么现用先转换，要么把它的数据复制到用户自己可以管理的内存中
	const char *c;
	string s = "1234";
	c = s.c_str();
	cout<<c<<endl;
	s = "abcde";
	cout<<c<<endl;
}
```



17、typedef与using关键字，定义指向数组的指针

typedef：定义一个p指针，指向包含10个int变量的数组

```c++
typedef int (*p)[10];
int main()
{
    int arr[10] = {1,2,3,4};
    p a = &arr;
  
    for(int i=0; i<10; ++i)
        cout << (*a)[i] << " ";
  
    return 0;
}
```

using: 定义一个p2指针，执行包含10个int变量的数组

```c++
using p2 = int(*)[10];
int main()
{
    
    int arr[10] = {1,2,3,4};
    p2 a2 = &arr;
    
    for(int i=0; i<10; ++i)
        cout << (*a2)[i] << " ";
        
    return 0;
}
```



18、noexcept关键字

c++11中，声明一个函数不可以抛出任何异常使用关键字noexcept

如果被noexcept修饰的函数抛出了异常，编译器直接选择调用标准库里面的std::terminate()终止程序，比throw()在处理异常的效率更高些，有效的防止了异常的扩散传播



19、constexpr

允许将变量声明为constexpr类型，让编译器来验证变量的值是否是一个常量表达式。

const与constexpr：

1、const并未区分出编译期常量和运行期常量

2、constexpr限定在了编译期常量；声明为constexpr的变量一定是常量；constexpr修饰的函数的返回值不一定是编译器常量

```c++
#include <iostream>
#include <array>
using namespace std;

constexpr int foo(int i)
{
    return i + 5;
}

int main()
{
    int i = 10;
    std::array<int, foo(5)> arr; // OK
    
    foo(i); // Call is Ok
    
    // But...
    std::array<int, foo(i)> arr1; // Error
   
}
```

constexpr修饰的函数，如果其传入的参数可以在编译时计算出来，这个函数就会产生编译时的值，如foo(5)；

如果传入的参数不能在编译时计算出来，那么constexpr修饰的函数和普通函数一样。

检测constexpr函数是否产生编译时期值的方法很简单，就是利用std::array需要编译时常量才能编译通过的小技巧



参考：

c++ const和constexpr的区别？

**https://www.zhihu.com/question/35614219**



20、const int a; 编译错误，需要给定初始值



21、



## 二、开发者测试

1、测试覆盖中，覆盖程度最高的是 B

A 条件覆盖

B 条件组合覆盖

C 语句覆盖

D 条件及判定覆盖



## 三、编程规范



1、Class 枚举：大驼峰

enum class Color = { RED, GREEN, BLUE};

2、constexpr

3、字面量



## 四、stl容器相关

1、vector的begin和end指向的位置，vector为空的情况判断

begin()返回迭代器，指向容器的第一个元素；

end()返回迭代器，指向容器的最后一个元素的下一个位置；

rbegin()返回一个逆序迭代器，指向容器的最后一个元素；

rend()返回一个逆序迭代器，指向容器的第一个元素前面的位置



vector判空：empty() or size()==0



2、让vector提升性能 （ACD）

A. 用reserve改变大小（避免vector扩充时进行拷贝）；

B. 用at代替[]（会检查是否越界，降低性能）；

C. 用emplace_back代替push_back；

push_back(临时对象)：调用构造函数构造这个临时对象；调用拷贝构造函数将这个临时对象放入容器；释放临时变量。这样造成临时变量申请的资源浪费。

引入右值引用，转移构造函数后，push_back()右值时就会调用构造函数和转移构造函数；

之后进一步优化产生了emplace_back，不需要触发拷贝构造和转移构造，直接根据参数初始化临时对象的成员

D. 用emplace代替insert

emplace调用了拷贝构造函数，在容器创建元素时，直接根据需要插入的元素进行构造；

insert先构造了元素，调用了重载运算符函数，对函数进行了赋值，相比emplace比较耗时



## 五、gdb

gdb速查手册：https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf



1、不退出gdb指向dir命令用什么命令 dir



2、gdb查看调用栈的命令 bt

查看调用栈最外三层的命令： bt 3

查看调用栈内三层的命令： bt -3

3、gdb监控变量修改并退出的命令

watch，用于观察某个变量/内存地址的状态，可以监控变量/内存值是否被程序读/写的情况



4、gdb查看指定位置起始的内存内容，以16进制单字节

x/1xb

x/ <n/f/u> <addr>

1、n是一个正整数，表示显示内存的长度，即从当前地址向后显示几个地址的内容；

2、f表示显示的格式，默认使用十六进制格式；

x （hexadecimal）按十六进制格式显示变量。 
d （signed decimal）按十进制格式显示变量。 
u （unsigned decimal）按十进制格式显示无符号整型。 
o （octal）按八进制格式显示变量。 
t （binary）按二进制格式显示变量。 
a （address）按十六进制格式显示地址，并显示距离前继符号的偏移量(offset)。常用于定位未知地址(变量)。 
c （character）按字符格式显示变量。 
f （floating）按浮点数格式显示变量。

3、u表示从当前地址往后请求的位宽大小，默认是4个字节。b表示单字节，w表示双字节，w表示4字节，g表示8字节



5、设置了断点，继续执行的命令是？

continue，next，step



6、gdb set [arg1 arg2] 可以改变运行时的参数



7、gdb show args可以运行的时候展现参数
