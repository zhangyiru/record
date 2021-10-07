nullptr 表示指针空值类型

NULL 在c++中表示0，c语言中是(void*)0



Int* p1 = nullptr;

char* p2 = nullptr; 

double *p3 = nullptr;



nullptr无法隐式转换为整型，但是可以隐式匹配指针类型。

使用nullptr初始化空指针可以使得程序更健壮。