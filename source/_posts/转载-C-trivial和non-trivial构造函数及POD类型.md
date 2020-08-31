---
title: '[转载]C++ trivial和non-trivial构造函数及POD类型'
date: 2020-08-31 05:54:24
tags: Cpp
categories:
cover: https://images2015.cnblogs.com/blog/623573/201610/623573-20161024233936593-703493344.png
---
<meta name="referrer" content="no-referrer" />


最近正纠结这个问题就转过来了，做了点补充（参考《深度探索C++对象模型》）

trivial意思是无意义，这个trivial和non-trivial是对类的四种函数来说的：

- 默认构造函数(default constructor)
- 拷贝构造函数(copy constructor)
- 赋值函数(copy assignment operator)
- 析构函数(destructor)

如果至少满足下面3条里的一条：

1. 显式(explict)定义了这四种函数。
2. 类里有非静态非POD的数据成员。
3. 有基类。

那么上面的四种函数是non-trivial函数，比如叫non-trivial constructor、non-trivial copy constructor…，也就是说有意义的函数，里面有以下必要的操作，比如类成员的初始化，释放内存等。

POD意思是Plain Old Data，也就是C++的内建类型或传统的C结构体类型(C风格的struct结构体定义的数据结构)。POD类型必然有trivial constructor/ destructor/ copy constructor / copy assignment operator四种函数。

> The copy assignment operator for class `T` is trivial if all of the following is true:
>
> - it is not user-provided (meaning, it is implicitly-defined or defaulted) ;
> - `T` has no virtual member functions;
> - `T` has no virtual base classes;
> - the copy assignment operator selected for every direct base of `T` is trivial;
> - the copy assignment operator selected for every non-static class type (or array of class type) member of `T` is trivial.
>
> A trivial copy assignment operator makes a copy of the object representation as if by [std::memmove](https://en.cppreference.com/w/cpp/string/byte/memmove). All data types compatible with the C language (POD types) are trivially copy-assignable.

```cpp
//整个T是POD类型
class T
{
    //没有显式定义ctor/dtor/copy/assignemt所以都是trivial
    int a; //POD类型
};

//整个T1是非POD类型
class T1
{
    T1() //显式定义了构造函数，所以是non-trivial ctor
    {}
    //没有显式定义ctor/dtor/copy/assignemt所以都是trivial
    int a;//POD类型
    std::string b; //非POD类型
};
```

那这有什么用处呢？

如果这个类都是constructor/ destructor/ copy constructor / copy assignment operator函数，我们对这个类进行构造、析构、拷贝和赋值时可以采用最有效率的方法，不调用无所事事正真的那些consructor/destructor等，而直接采用内存操作如malloc()、memcpy()等提高性能，这也是SGI STL内部干的事情。

比如STL的copy算法最基本的想法是这样的：

 

```cpp
// 非POD重载指针数值
template <class T> void copy(T* source, T* destination, int n, __false_type)
{
    // 省略异常处理
    for (; n > 0; n--,source++,destination++)
    {
        // 调用source的复制构造函数
        constructor(source, *destination);
    }
}

// POD重载指针数值
template <class T> void copy(T* source, T* destination, int n, __false_type)
{
    // 省略异常处理
    memmove(source, destination, n);
}
```

当然实际的copy比这个复杂多了，有非常多的特化等，这个只是其中一方面而已。

**用户自定义型别，都被编译器视为拥有non-trivial ctor/dtor/operator=。如果我们确知某个class具备的是trivial ctor/dtor/operator=，就得自己动手为它做特性设定，才能保证编译器知道它的身份。**