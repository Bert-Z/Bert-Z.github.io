---
title: '[转载]为什么C++11引入了std::ref？'
date: 2020-08-12 05:19:41
tags: Cpp
categories:
cover: https://isocpp.org/files/img/fbgohNwY5aexuWMq9qJ95DEN.jpg
---
<meta name="referrer" content="no-referrer" />

　　C++本身有引用（&），为什么C++11又引入了std::ref？

　　主要是考虑函数式编程（如std::bind）在使用时，是对参数直接拷贝，而不是引用。如下例子：

```cpp
#include <functional>
#include <iostream>
 
void f(int& n1, int& n2, const int& n3)
{
    std::cout << "In function: " << n1 << ' ' << n2 << ' ' << n3 << '\n';
    ++n1; // increments the copy of n1 stored in the function object
    ++n2; // increments the main()'s n2
    // ++n3; // compile error
}
 
int main()
{
    int n1 = 1, n2 = 2, n3 = 3;
    std::function<void()> bound_f = std::bind(f, n1, std::ref(n2), std::cref(n3));
    n1 = 10;
    n2 = 11;
    n3 = 12;
    std::cout << "Before function: " << n1 << ' ' << n2 << ' ' << n3 << '\n';
    bound_f();
    std::cout << "After function: " << n1 << ' ' << n2 << ' ' << n3 << '\n';
}
```

```
Output:
Before function: 10 11 12
In function: 1 11 12
After function: 10 12 12
```

　　上述代码在执行std::bind后，在函数f()中n1的值仍然是1，n2和n3改成了修改的值。说明std::bind使用的是参数的拷贝而不是引用。具体为什么std::bind不使用引用，可能确实有一些需求，使得C++11的设计者认为默认应该采用拷贝，如果使用者有需求，加上std::ref即可。