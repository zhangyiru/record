## 题目

去掉move效率更高的是
class Foo {
public:
Foo(int size) :buf(size){}
private:
vector buf;
};
class Bar {
public:
Bar(int x, int y) : x(x), y(y){}
private:
int x;
int y;
};
A Foo dst(move(Foo(5)));
B Foo src(5)
Foo dst(move(src));
C Bar dst(move(Bar(1,2)));
D Bar src(1,2)
Bar dst(move(src));



AC可以只调用构造函数，但是多了move会多调用移动构造函数，BD的话是把移动构造函数改成了拷贝构造函数，效率是没有变化。



## 代码

```c++
#include <iostream>
#include <vector>
using namespace std;

class Foo {
public:
    Foo(int size) :buf(size){
        cout << "Foo(int size)" <<endl;
    }
    Foo(Foo &f){
        cout << "Foo(Foo &f)" <<endl;
        buf = f.buf;
    }
    Foo &operator=(Foo &f) {
        cout << "Foo &operator=(Foo &f)" <<endl;
        buf = f.buf;
        return *this;
    }
    Foo(Foo &&f) {
        cout << "Foo(Foo &&f)" <<endl;
        buf = move(f.buf);
    }
private:
    vector<int> buf;
};

class Bar {
public:
    Bar(int x,int y) :x(x),y(y){
        cout << "Bar(int x,int y)" <<endl;
    }
    Bar(Bar &f){
        cout << "Bar(Bar &f)" <<endl;
        x = f.x;
        y = f.y;
    }
    Bar &operator=(Bar &f) {
        cout << "Bar &operator=(Bar &f)" <<endl;
        x = f.x;
        y = f.y;
        return *this;
    }
    Bar(Bar &&f) {
        cout << "Bar(Bar &&f)" <<endl;
        x = move(f.x);
        y = move(f.y);
    }
private:
    int x;
    int y;
};
int main() {

    cout <<"=====1======"<<endl;
    Foo e(move(Foo(0)));

    cout <<"=====2======"<<endl;
    Foo f(Foo(0));

    cout <<"=====3======"<<endl;
    Foo a(1);
    Foo b(a);

    cout <<"=====4======"<<endl;
    Foo c(1);
    Foo d(move(a));

    cout <<"=====5======"<<endl;
    Bar dst(move(Bar(1,2)));

    cout <<"=====6======"<<endl;
    Bar dst1(Bar(1,2));

    cout <<"=====7======"<<endl;
    Bar dst2(1,2);
    Bar dst22(move(dst2));

    cout <<"=====8======"<<endl;
    Bar dst3(1,2);
    Bar dst33(dst3);
}
```