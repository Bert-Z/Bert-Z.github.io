---
title: '[转载]C++构造函数和析构函数的调用顺序'
date: 2020-08-06 15:37:43
tags: Cpp
categories:
cover: https://images2018.cnblogs.com/blog/1074126/201808/1074126-20180807204732052-1411619377.png
---
<meta name="referrer" content="no-referrer" />

C++如果没有分配资源那么就不一定需要析构函数，否则就需要释放资源，如下：

```cpp
class Vector {
public:
Vector(int s) :elem{new double[s]}, sz{s} { }; // constructor: acquire memory
˜Vector() { delete[] elem; } // destructor: release memory

// ...
private:
double∗ elem; // elem points to an array of sz doubles
int sz; // sz is non-negative
};
```

 

析构函数的调用顺序是从底往上的：
[1] first, the constructor invokes its base class constructors,

首先，是调用该类的基类构造函数
[2] then, it invokes the member constructors, and

然后是该成员类的构造函数（重要记忆，有些书好像没有提到）
[3] finally, it executes its own body.

最后才是本类的构造函数
而析构函数的调用顺序刚好是相反

A destructor ‘‘tears down’’ an object in the reverse order:
[1] first, the destructor executes its own body,

首先调用本类的析构函数
[2] then, it invokes its member destructors, and

然后调用成员类的析构函数
[3] finally, it inv okes its base class destructors.

最后才是基类的析构函数

特别指出：虚基类是在其他任何类之前调用构造函数的，而在所有其他类之后调用析构函数的。

这样的调用顺序是为了保证基类或者成员类没有在他们创建之前被调用，或者在析构之后还可能被调用。

构造函数根据声明顺序，执行成员类和基类的构造函数，而不是根据初始化列表（initializers）顺序。

 

没定义构造函数的时候，可以使用默认构造函数，定义了之后就会发生错误了，如下：

```cpp
struct S1 {
string s;
};
S1 x; // OK: x.s is initialized to ""
struct X { X(int); };
struct S2 {
X x;
};
S2 x1; // error :
S2 x2 {1}; // OK: x2.x is initialized with 1
```


参考：

The C++ programming language