---
title: '[转载]在 C++ 中读取字符串中成对定界符中的子串'
date: 2020-05-15 13:46:23
tags: Cpp
categories:
cover: https://cdn.journaldev.com/wp-content/uploads/2020/05/getline_cpp.png
---
<meta name="referrer" content="no-referrer" />

# [转载]在 C++ 中读取字符串中成对定界符中的子串

这篇文章是一个简单的记录，解决类似这样的问题。

假设有一个字符串

```cpp
std::string = "<foo:bar> <baz:qux>";
```

要怎样才能读出其中的 `foo:bar` 以及 `baz:qux` 呢？使用 `regex` 正则库当然是一个办法，不过在规整的情况下，我们还有更优雅的选择。



## `std::getline`

`std::getline` 函数来自头文件 `string`。因此，显而易见它是一个和字符串相关的函数。它有四个重载版本，具体如下：

```cpp
istream& getline (istream&  is, string& str, char delim);
istream& getline (istream&& is, string& str, char delim);
istream& getline (istream&  is, string& str);
istream& getline (istream&& is, string& str);
```

其中第 2 个和第 4 个版本，分别是第 1 个和第 3 个版本对应的右值引用版本。函数基本的作用，是自输入流读入内容，直至遇见定界符（带 `delim` 的版本）或者换行符 `\n`（不带 `delim` 的版本），并将读入的内容写入字符串 `str`（不包括定界符本身）。

值得注意的是，函数会修改输入流 `is` 本身，并且返回它的左值引用。这一特性意味着 `std::getline` 函数可以「串行」执行。

## 实际解析看看

请看代码。

```cpp
#include <iostream>
#include <sstream>
#include <string>

int main() {
    const char LEFT_DELIM  = '<';
    const char RIGHT_DELIM = '>';
    const char MID_DELIM   = ':';
    const std::string test = "<foo:bar> <baz:qux>";

    std::istringstream iss(test);
    std::string skip, value;

    while (std::getline(std::getline(iss, skip, LEFT_DELIM), value, RIGHT_DELIM)) {
        std::cout << "-----" << '\n';
        std::istringstream iss(value);
        std::string key, value;
        while (std::getline(std::getline(iss, key, MID_DELIM), value)) {
            std::cout << key << '\t' << value << '\n';
        }
    }
    return 0;
}
```

在这里，我们将 `test` 与字符串输入流 `iss` 绑定。

而后用 `std::getline` 从 `iss` 中先后将内容读入 `skip` 和 `value`。在这里，内层的 `std::getline(iss, skip, LEFT_DELIM)` 首先执行，将 `LEFT_DELIM` 之前的内容（实际上是空字符串）读入 `skip` 变量，而后修改 `iss` 的状态，并返回它的左值引用作为外层 `std::getline` 的第一个参数。而后，外层的 `std::getline` 将第一次遇到 `RIGHT_DELIM` 之前的内容 `foo:bar` 读入 `value` 变量。如此循环。

进入到外层 `while` 循环的内部，我们采用了类似的策略，分别将冒号前后的内容，存入 `key` 和 `value` 两个变量。读者可以自行分析。