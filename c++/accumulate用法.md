# accumulate用法

## 1、定义

\#include<numeric>

 

## 2、作用

（1）累加求和

（2）自定义类型数据的处理



## 3、累加求和

三个形参：头两个形参指定要累加的元素范围，第三个形参则是累加的初值

将第三个形参设置为指定的初始值，然后在此初值上累加输入范围内所有元素的值



## 4、自定义数据类型的处理

手写一个回调函数来实现自定义数据的处理，让它作为accumulate()的第四个参数

```c
template<class _InIt,
	class _Ty,
	class _Fn2> inline
	_Ty _Accumulate(_InIt _First, _InIt _Last, _Ty _Val, _Fn2 _Func)
	{	// return sum of _Val and all in [_First, _Last), using _Func
	for (; _First != _Last; ++_First)
		_Val = _Func(_Val, *_First);
	return (_Val);
	}
```

举例：

```c++
#include <vector>
#include <string>
using namespace std;

struct Grade
{
	string name;
	int grade;
};

int main()
{
	Grade subject[3] = {
		{ "English", 80 },
		{ "Biology", 70 },
		{ "History", 90 }
	};

	int sum = accumulate(subject, subject + 3, 0, [](int a, Grade b){return a + b.grade; });
	cout << sum << endl;
	 
	system("pause");
	return 0;

}
```

解释：

根据模板定义

```c
_Val = _Func(_Val, *_First);
```

a就是0，依次累加subject里的grade

