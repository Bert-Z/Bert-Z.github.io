---
title: '[转载]在 C++ 中分割字符串'
date: 2020-05-15 13:43:48
tags: Cpp
categories: 
cover: https://www.fluentcpp.com/wp-content/uploads/2017/04/how_to_split_string.png
---
<meta name="referrer" content="no-referrer" />

# [转载]在 C++ 中分割字符串

昨天在网上看到，C++ 至今为止没有官方实现的字符串分割函数。相比 Python、Java 等语言，多少是有些不便的。

这里我们来在 C++ 中实现字符串分割函数。



## 利用来自 C 的 `strtok` 函数

C 语言的 `string.h` 中提供了名为 `strtok` 函数，用于对 C 风格的字符串进行分割。其函数签名为

```cpp
char* strtok(char* str, const char* delim);
```

当 `str` 不是空指针时，`strtok` 会从头开始寻找第一个合法的分隔符，而后将分隔符替换成 `\0`，并将分隔符的位置保存在一个静态变量中，最后返回 `str`。这样，按照 C 风格的字符串，我们就能获取分割得到的第一个 token。当 `str` 是空指针时，`strtok` 将会从记录的空指针处继续尝试分割。

因此，我们可以定义这样的 `split` 函数。

```cpp
#include <iostream>
#include <cstring>
#include <string>
#include <vector>
#include <algorithm>

void split(const std::string& s,
    std::vector<std::string>& sv,
                  const char* delim = " ") {
    sv.clear();                                 // 1.
    char* buffer = new char[s.size() + 1];      // 2.
    buffer[s.size()] = '\0';
    std::copy(s.begin(), s.end(), buffer);      // 3.
    char* p = std::strtok(buffer, delim);       // 4.
    do {
        sv.push_back(p);                        // 5.
    } while ((p = std::strtok(NULL, delim)));   // 6.
    delete[] buffer;
    return;
}

int main() {
    std::string s("abc:def::ghi");
    std::vector<std::string> sv;

    split(s, sv, ":");

    for (std::vector<std::string>::const_iterator iter = sv.begin();
                                                   iter != sv.end();
                                                             ++iter) {
        std::cout << *iter << std::endl;
    }

    return 0;
}
```

对于 `split` 函数来说，它无法预知传入的 `sv` 变量的情况。因此，在 (1) 处，我们将 `sv` 这个 `std::vector` 清空备用。由于 `std::strtok` 函数需要修改传入的 `str` 的内容，所以它需要 `char*` 类型的参数。故而，在 (2)(3) 两处，我们将 `std::string` 当中的内容复制一份。(4)(6) 两处对 `std::strtok` 的调用，帮助我们将 token 逐个压入 `sv` 当中。

> 在 C++11 及更高版本中，(5) 可替换为 `sv.emplace_back(p)`，以避免额外的拷贝。

上述代码的结果是：

```
abc
def
ghi
```

## 利用 C++ 中的流

实际上，利用纯 C++ 风格的代码，也是可以实现一个优雅的字符串分割函数的。

在[前作](https://liam.page/2017/03/13/read-substrings-from-paired-delimiters-in-Cpp/)中，我们介绍了 C++ 的 `std::getline` 函数。它接收一个输入流，将输入流至行末/分隔符部分的字符串保存在临时的字符串中；同时，返回输入流的左值引用。考虑到输入流本身可以用作条件判断，我们可以将 `std::getline` 与 `while` 循环联用，达成目的。

简单实现如下：

```cpp
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

void split(const std::string& s,
    std::vector<std::string>& sv,
                   const char delim = ' ') {
    sv.clear();
    std::istringstream iss(s);
    std::string temp;

    while (std::getline(iss, temp, delim)) {
        sv.emplace_back(std::move(temp));
    }

    return;
}

int main() {
    std::string s("abc:def:ghi");
    std::vector<std::string> sv;

    split(s, sv, ':');

    for (const auto& s : sv) {
        std::cout << s << std::endl;
    }

    return 0;
}
```

代码中，我们借助字符串输入流 `istringstream` 处理带分割的字符串 `s`。而后将各个 `delim` 之间的内容，保存在临时字符串 `temp` 当中，并移动到向量 `sv` 的末尾。

上述代码的结果是：

```
abc
def
ghi
```