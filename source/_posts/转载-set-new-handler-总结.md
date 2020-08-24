---
title: '[转载]set_new_handler()总结'
date: 2020-08-24 08:13:57
tags: Cpp
categories:
cover: https://slidesplayer.com/slide/15483914/93/images/61/%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%9A%84%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86+%E6%B2%A1%E6%9C%89%E6%8A%9B%E5%87%BA%E5%BC%82%E5%B8%B8%EF%BC%9A%E4%B8%8D%E9%9C%80%E8%A6%81throw-try-catch%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6+double+%2Aptr%5B50%5D%3B.jpg
---
<meta name="referrer" content="no-referrer" />

在看STL源码时，发现内存分配时会先调用set_new_handler(0);
不知其意，故在网上搜寻了相关资料，总结如下：

　　**函数原型：**
　　new_handler set_new_handler (new_handler new_p) throw();
　　
　　new_handler类型函数将在默认内存申请函数(operator new和operator new[])申请内存失败时被调用。
　　new_handler函数会尝试为新的内存申请请求提供更多的可用空间。当且仅当函数成功地提供了更多的可用空间，它才返回。否则，或者抛出bad_alloc异常（或bad_alloc派生类）或者终止程序（比如调用abort或exit）。
　　如果new_handler函数返回（提供了更多可用空间）后，当内存申请函数申请指定的内存空间失败时，它会被再次调用，直到new_handle函数不返回或被替换。
　　在set_new_handler函数第一次被调用前，或者new_p是一个空指针，默认的内存申请函数在内存申请失败的时候直接抛出bad_alloc异常。

**参数：**
　　new_handler是一个函数指针，不带参数且返回void。
该函数可以提供更多的可用空间，或抛出一个异常，或直接终止程序。如果new_p是一个空指针（null-pointer），new_handler函数被重置为空。

**返回值：**
　　首次调用返回0，之后的调用返回首次设置的的new_handler。

**相关总结：**
1）new_handler类型函数将在默认内存申请函数(operator new和operator new[])申请内存失败时被调用。
2）默认情况下，当内存不能够分配时，new操作将抛出一个bad_alloc的异常。你可以改变这个默认操作，通过set_new_handler设置一个new_handler。你可以使用set_new_handler(0)，获得一个不抛出异常的new.
3）用户定义的my_handler应该可以做以下几几件事之一:
　3.1）释放内存，产生更多可用的内存，可以看下面的例子
　3.2）抛出bad_alloc异常（或bad_alloc派生类）
　3.3）终止程序（比如调用abort或exit）。
　很显然，如果my_handler无法实现以上功能之一，程序将陷入死循环。

下面是stackoverflow上的一个例子:

```cpp
#include <iostream>
#inlucde <new>
/// buffer to be allocated after custom new handler has been installed
char* g_pSafetyBuffer = NULL;

/// exceptional one time release of a global reserve
void my_new_handler()
{
    if (g_pSafetyBuffer) {
        delete [] g_pSafetyBuffer;
        g_pSafetyBuffer = NULL;
        std::cout << "[Free some pre-allocated memory]";
        return;
    }
    std::cout << "[No memory to free, throw bad_alloc]";
    throw std::bad_alloc();
}

/// illustrates how a custom new handler may work
int main()
{
    enum { MEM_CHUNK_SIZE = 1000*1000 }; // adjust according to your system
    std::set_new_handler(my_new_handler);
    g_pSafetyBuffer = new char[801*MEM_CHUNK_SIZE];
    try {
        while (true) {
            std::cout << "Trying another new... ";
            new char[200*MEM_CHUNK_SIZE];
            std::cout << " ...succeeded.\n";
        }
    } catch (const std::bad_alloc& e) {
        std::cout << " ...failed.\n";
    }
    return 0;
}
```

下面是一个可能的输出：
Trying another new… …succeeded.
Trying another new… …succeeded.
Trying another new… …succeeded.
Trying another new… …succeeded.
Trying another new… …succeeded.
Trying another new… [Free some pre-allocated memory] …succeeded.
Trying another new… …succeeded.
Trying another new… …succeeded.
Trying another new… …succeeded.
Trying another new… [No memory to free, throw bad_alloc] …failed.

Process returned 0 (0x0) execution time : 0.046 s
Press any key to continue.

　　相信这个例子让大家有了更好的了解了吧。