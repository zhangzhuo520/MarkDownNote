#[C++11] std::functional
-----------------------
C++11中std::functional最常用的就是用来实现函数回调。这里做一些补充:
std::functional是一种通用、多态的函数封装.

###示例：
```
#include <iostream>
#include <functional>
using namespace std;

std::function<int(int)> Func;

// 普通函数
int ordinaryFunc(int a)
{
    return a;
}

// 匿名函数
auto lambdaFunc = [](int a)->int{return a;};

// 仿函数
class Functor
{
public:
    int operator()(int a)
    {
        return a;
    }
};

// 成员函数和静态函数
class Class
{
public:
    int ClassMember(int a){return a;}
    static int StaticMember(int a){return a;}
};

int main()
{
    Func = ordinaryFunc;
    int result;
    result = Func(10);
    cout << "普通函数" << result << endl;

    Func = lambdaFunc;
    result = Func(20);
    cout << "Lambda函数" << result << endl;

    Functor m_functor;
    Func = m_functor;
    result = Func(30);
    cout << "仿函数" << result << endl;

    Class m_obj;
    Func = std::bind(&Class::ClassMember, m_obj, std::placeholders::_1);
    result = Func(40);
    cout << "成员函数" << result << endl;

    Func = Class::StaticMember;
    result = Func(50);
    cout << "静态函数" << result << endl;

    return 0;
}
```