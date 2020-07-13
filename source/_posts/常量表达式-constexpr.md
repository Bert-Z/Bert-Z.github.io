---
title: 常量表达式(constexpr)
date: 2020-07-13 06:38:39
tags: Cpp
categories:
cover: https://dev.to/social_previews/article/284081.png
---
<meta name="referrer" content="no-referrer" />

## **一. const 和constexpr的区别**

（一）修饰变量时，**const为“运行期常量”**，即运行期数据是只读的。而**constexpr为“编译期”常量**，这是const无法保证的。两者都是对象和函数接口的组成部分。

（二）修饰函数时，与const关键字相比，**constexpr关键字不仅可以修饰变量和指针，还可以修饰函数(含构造函数)**。注意constexpr用于定义自定义类的对象时，要求该类具有常量构造函数,而使用const定义类对象时，则无此要求。

（三）两者在**修饰指针时**，行为有所差异。**const放在\*号前，表示指针指向的内容不能被修改．const放在\*号后，表示指针不能被修改**；而**constexpr关键字只能放在\*号前面，并且表示指针所指的内容不能被修改**。

## **二、常量表达式函数**

（一）构成constexpr函数的条件

　　1. 函数体只有单一的return语句（可以通过“：”或递归来扩展）。**在C++14中这条己经不再是限制。**

　　2. 函数体必须有返回值（C++11中要求不能是void函数，但C++14己不再限制）

　　3、constexpr函数内部不能调用非常量表达式的函数，会造成编译失败。

　　4. return返回语句表达式中不能使用非常量表达式的函数、全局数据，且必须是一个常量表达式。

（二）注意事项

　　1. 在使用常量表达式函数前，必须先被定义。不能先声明，然后在函数调用后再定义。

　　2. constexpr函数在调用时**若传入的实参均为编译期己知的，则返回编译期常量**。只要**任何一个实参在编译期未知，则它的运作与普通函数无异**，将产生**运行期结果**。

　　3. constexpr函数的构成条件不满足时，就会变成一个普通的函数。同时**constexpr函数可以同时应用于编译期语境或运行期语境**（编译期语境如constexpr int a = func_constexpr(x, y)。运行期语境如int a = func_constexpr(x, y)）。

【编程实验】const和constexpr的差异

```cpp
#include <iostream>
#include <array>

using namespace std;

int g_count = 0;

int normalfunc(int x)
{
    return x;
}

//constexpr函数（即可当编译期常量表达式函数，也可以当作普通函数使用）
constexpr int func1(int x)
{
    return x + 2;
}

constexpr void func2(){} //普通函数，constexpr必须有非void的返回值类型
constexpr int func3();   //注意这里只声明，定义在main()函数之后
//constexpr int func4()    //编译失败
//{
//    int x = normalfunc(2); //调用了非constexpr函数！
//    return x;
//}

//幂函数：用于计算base的exp次幂
constexpr int pow(int base, int exp)
{
    //版本1：递归（一条语句）----> C++11要求有且只有一条语句（即return语句)。
    //return (exp == 0) ? 1 : base * pow(base, exp - 1);

    //版本2：非递归（多条语句） --->C++14以上
    auto ret = 1;
    for (int i = 0; i < exp; ++i) {
        ret *= base;
    }

    return ret;
}

int main()
{
    //1.1 修饰变量时
    int x = 0;                 //非constexpr变量
    const int y = 0;
    constexpr int a = 2;       //a为编译期常量
    constexpr int b = a + 4;   //b为编译期常量
    constexpr int c = y + 1;   //c为编译期常量，y直接取自符号表，而不是来自内存值。

    //constexpr auto m = x;    //error，因为x不是常量
    const auto n = x;          //ok, d为x的副本

    constexpr auto N = 10;
    //std::array<int, n> arr1; //error, n不是编译期常量
    std::array<int, N> arr2;   //ok, N为编译期常量
    std::array<int, c> arr3;   //ok, c为编译期常量  

    constexpr auto exp = 5;
    std::array<int, pow(3, exp)> arr4;   //ok! pow是constexpr函数
    //std::array<int, pow(3, x)> arr5;   //error! x为运行期变量，从而导致pow返回值为运行期的值

    //1.2 constexpr修饰指针时，表示指针本身不可修改！！！（constexpr只能放在*的左侧）
    constexpr int* ptr2 = &g_count; //ok，全局变量，编译期地址可确定
    //ptr2 = nullptr;     //注意，与const不同，这里表示ptr2是常量。
    *ptr2 = 100;

    //constexpr int* ptr1 = &x;  //error，因为x在栈上分配，在编译期不能确定其地址。

    //1.3 修饰函数时
    int var1 = func3();               //ok！ func3先声明，使用后定义。
    //constexpr int var2 = func3();   //error，在调用函数前，func3必须定义好了（不能只看到声明）。

    constexpr int c1 = func1(N);      //constexpr语境： 返回值赋值constexpr变量，是constexpr函数（实参也为constexpr）。
    int c2 = func1(x);                //非constexpr语境：func1用于返回值赋值给普通变量时，当作普通函数使用！
    int c3 = func1(N);                //非constexpr语境：func1用于返回值赋值给普通变量时，当作普通函数使用！

    return 0;
}

constexpr int func3() //声明放在main之前，这里是定义！
{
    return 0;
}
```

## **三、constexpr的应用**

（一）定义自定义类的constexpr对象。

　　1. 自定义类的**构造函数须为constexpr函数**。

　　2. constexpr不能用于修饰virtual函数。因为virtual是运行时的行为，与constexpr的意义冲突。

　　3. C++11中被声明为constexpr成员函数会自动设为const函数，但C++14中不再自动设为const函数。

（二）函数模板： 实例化后的函数是否为constexpr函数，在编译期是不确定的，**取决于传入的实参是否为constexpr类型**。

（三）constexpr类型的函数递归

　　1. 利用constexpr进行编译期运算的编程方式称为constexpr元编程

　　2 .基于模板编译期运算的编程方式，叫模板元编程。

【编程实验】constexpr的应用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```cpp
#include <iostream>
using namespace std;

//1. constexpr构造函数和成员函数
class Point
{
    double x, y;
public:
    //常量表达式构造函数
    constexpr Point(double x=0, double y=0) noexcept : x(x),y(y){}

    //成员函数（注意：constexpr不允许用于修饰virtual函数）
    constexpr double getX() const noexcept { return x;} //constexpr函数，在C++11中默认为const函数。但C++14中取消了，这里加const
    constexpr double getY() const noexcept { return y;}

    //以下两个函数在C++11中无法声明为constexpr（C++14可以！），原因如下：
    //1.C++11中被声明为constexpr成员函数会自动设为const函数，而这两个函数需要更改成员变量的值，这与const成员函数
    //却不允许需改成员的值，会产生编译错误。C++14中constexpr函数不再默认为const函数，因此可以修改成员变量。
    //2. C++11中constexpr函数不能返回void型，但C++14中不再限制。
    constexpr void setX(double newX) noexcept { x = newX; } //C++11中须去掉constexpr;
    constexpr void setY(double newY) noexcept { y = newY; }
};

//计算中点坐标
constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
{
    //p1和p2均为const对象，会调用const版本的getX()和getY()
    return { (p1.getX() + p2.getX()) / 2, (p1.getY() + p2.getY()) / 2 };
}

//返回p相对于原点的中心对称点（C++14）
constexpr Point reflection(const Point& p) noexcept
{
    Point ret;
    ret.setX(-p.getX());
    ret.setY(-p.getY());
    return ret;
}

//2. 函数模板：（constexpr元编程）
//函数是否为constexpr函数，编译期未知。取决于传入的实参t是否为constexpr变量/对象
template<typename T>
constexpr T func(T t) 
{
    return t;
}

struct NotLiteral  
{
    int i;
    NotLiteral():i(5){} //注意：构造函数不是constexpr类型！
};

//3.constexpr类型的递归函数
//3.1 求斐波那契数
constexpr int Fib(int n)
{
    return (n == 1) ? 1 : ((n == 2) ? 1 : Fib(n-1) + Fib(n -2));
}

//3.2 模板递归（模板元编程）
template<long N>
struct Fibonacci
{
    static const long val = Fibonacci<N - 1>::val + Fibonacci<N - 2>::val;
};

//特化
template<> struct Fibonacci<2> { static const long val = 1; };
template<> struct Fibonacci<1> { static const long val = 1; };
template<> struct Fibonacci<0> { static const long val = 0; };

void printArray(int a[], int len)
{
    cout << "Fibonacci： ";
    for (int i=0; i<len; ++i)
    {
        cout << a[i] << " ";
    }
    cout << endl;
}

int main()
{
    //1. 用constexpr定义自定义类的对象（要求该类具有constexpr构造函数）
    constexpr Point p1(9.4, 27.7);    //ok, p1为编译期常量
    constexpr Point p2{ 28.8, 5.3 };  //ok，同上
    //1.1求中点
    constexpr auto mid = midpoint(p1, p2);  //mid为编译期常量，mid.getX()也是编译期常量！！！
    //1.2求关于原点的对称点
    constexpr auto reflectedMid = reflection(mid);

    //2. constexpr型的函数模板
    NotLiteral n1;             //非constexpr对象
    NotLiteral n2 = func(n1);  //传入实参为非constexpr对象，func成为普通函数！
    //constexpr NotLiteral n3 = func(n1); //error，由于NotLiteral的构造函数不是constexpr类型，
                                          //不能用constexpr定义该类型的constexpr的对象！！！
    constexpr int a = func(1); //ok，1为constexpr对象

    //3. constexpr类型递归函数
    int fib1[]{ Fib(11), Fib(12), Fib(13), Fib(14) };
    printArray(fib1, 4);

    int fib2[] = { Fibonacci<11>::val, Fibonacci<12>::val, Fibonacci<13>::val, Fibonacci<14>::val };
    printArray(fib1, 4);

    return 0;
}
/*输出结果
Fibonacci： 89 144 233 377
Fibonacci： 89 144 233 377
*/
```

