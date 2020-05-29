---
title: typedef vs using
date: 2020-05-29 01:53:11
tags: Cpp
categories:
cover: https://iq.opengenus.org/content/images/2019/06/Untitled-Diagram-2.png
---
<meta name="referrer" content="no-referrer" />

-   对于定义**一般类型**的别名，二者没有区别。
-   对于定义**模板**的别名，只能用**using**。(c++11 还是鼓励用using)

```cpp
#include <iostream>

template <typename T>
class A
{
public:
    A() { std::cout << "A " << typeid(T).name() << std::endl; }
};

template <typename T>
using B = A<T>;

template <typename T>
typedef A<T> C;

int main()
{
    A<int> a;
    B<int> b; // OK, B is an alias of class A.
    C<int> c; // Syntax Error, C cannot be recognized as a type.
    return 0;
}

```