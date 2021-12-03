## 例题

以下atomic用法正确的是：

A。
std::atomic<int> y(0);
auto m = y;
cout << m << endl;

B。
std::atomic<int> y(0);
std::atomic<int> m(move(y));
cout << m << endl;

C。
std::atomic<int> y(0);
std::atomic<int> m(y.load());
cout << m << endl;

D。
std::atomic<int> y(0);
std::atomic<int> m;
m.store(y);
cout << m << endl;



## 解析：

c++17前，不能把atomic类型的赋值给auto，因为会调用拷贝构造函数，而atomic的拷贝构造函数是deleted；<br/>
c++17后，这种写法不会错误，但是会初始化对象，而不是调用拷贝构造函数
// https://stackoverflow.com/questions/66449441/in-c-why-does-auto-not-work-with-stdatomic
