---
title: '[转载]Linux动态库生成与使用指南'
date: 2020-07-23 06:41:53
tags: Cpp
categories:
cover:  https://images2017.cnblogs.com/blog/1234526/201709/1234526-20170915092632078-926508601.png
---
<meta name="referrer" content="no-referrer" />

Linux下动态库文件的文件名形如 `libxxx.so`，其中so是 Shared Object 的缩写，即可以共享的目标文件。

在链接动态库生成可执行文件时，并不会把动态库的代码复制到执行文件中，而是在执行文件中记录对动态库的引用。

程序执行时，再去加载动态库文件。如果动态库已经加载，则不必重复加载，从而能节省内存空间。

Linux下生成和使用动态库的步骤如下：

1. 编写源文件。
2. 将一个或几个源文件编译链接，生成共享库。
3. 通过 `-L<path> -lxxx` 的gcc选项链接生成的libxxx.so。
4. 把libxxx.so放入链接库的标准路径，或指定 `LD_LIBRARY_PATH`，才能运行链接了libxxx.so的程序。

下面通过实例详细讲解。

## 编写源文件

建立一个源文件： max.c，代码如下：

```cpp
int max(int n1, int n2, int n3)
{
    int max_num = n1;
    max_num = max_num < n2? n2: max_num;
    max_num = max_num < n3? n3: max_num;
    return max_num;
}
```

编译生成共享库：

```shell
gcc -fPIC -shared -o libmax.so max.c
```

我们会得到libmax.so。

实际上上述过程分为编译和链接两步， -fPIC是编译选项，PIC是 Position Independent Code 的缩写，表示要生成位置无关的代码，这是动态库需要的特性； -shared是链接选项，告诉gcc生成动态库而不是可执行文件。

上述的一行命令等同于：

```shell
gcc -c -fPIC max.c
gcc -shared -o libmax.so max.o
```

## 为动态库编写接口文件

为了让用户知道我们的动态库中有哪些接口可用，我们需要编写对应的头文件。

建立 max.h ，输入以下代码：

```cpp
#ifndef __MAX_H__
#define __MAX_H__

int max(int n1, int n2, int n3);

#endif
```

## 测试，链接动态库生成可执行文件

建立一个使用`max`函数的test.c，代码如下：

```cpp
#include <stdio.h>
#include "max.h"

int main(int argc, char *argv[])
{
    int a = 10, b = -2, c = 100;
    printf("max among 10, -2 and 100 is %d.\n", max(a, b, c));
    return 0;
}
```

`gcc test.c -L. -lmax` 生成a.out，其中`-lmax`表示要链接`libmax.so`。
`-L.`表示搜索要链接的库文件时包含当前路径。

注意，如果同一目录下同时存在同名的动态库和静态库，比如 `libmax.so` 和 `libmax.a` 都在当前路径下，
则gcc会优先链接动态库。

## 运行

运行 `./a.out` 会得到以下的错误提示。

```shell
./a.out: error while loading shared libraries: libmax.so: cannot open shared object file: No such file or directory
```

找不到libmax.so，原来Linux是通过 `/etc/ld.so.cache` 文件搜寻要链接的动态库的。
而 `/etc/ld.so.cache` 是 ldconfig 程序读取 `/etc/ld.so.conf` 文件生成的。
（注意， `/etc/ld.so.conf` 中并不必包含 `/lib` 和 `/usr/lib`，`ldconfig`程序会自动搜索这两个目录）

如果我们把 `libmax.so` 所在的路径添加到 `/etc/ld.so.conf` 中，再以root权限运行 `ldconfig` 程序，更新 `/etc/ld.so.cache` ，`a.out`运行时，就可以找到 `libmax.so`。

但作为一个简单的测试例子，让我们改动系统的东西，似乎不太合适。
还有另一种简单的方法，就是为`a.out`指定 `LD_LIBRARY_PATH`。

```shell
LD_LIBRARY_PATH=. ./a.out
```

程序就能正常运行了。`LD_LIBRARY_PATH=.` 是告诉 `a.out`，先在当前路径寻找链接的动态库。

> 对于elf格式的可执行程序，是由ld-linux.so*来完成的，它先后搜索elf文件的 `DT_RPATH` 段, 环境变量 `LD_LIBRARY_PATH`, /etc/ld.so.cache文件列表, /lib/,/usr/lib目录, 找到库文件后将其载入内存. (http://blog.chinaunix.net/uid-23592843-id-223539.html)

## makefile让工作自动化

编写makefile，内容如下：

```makefile
.PHONY: build test clean

build: libmax.so

libmax.so: max.o
	gcc -o $@  -shared $<

max.o: max.c
	gcc -c -fPIC $<

test: a.out

a.out: test.c libmax.so
	gcc test.c -L. -lmax
	LD_LIBRARY_PATH=. ./a.out

clean:
	rm -f *.o *.so a.out
```

`make build`就会生成`libmax.so`， `make test`就会生成`a.out`并执行，`make clean`会清理编译和测试结果。