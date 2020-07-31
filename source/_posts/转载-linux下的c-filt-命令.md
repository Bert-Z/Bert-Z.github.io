---
title: '[转载]linux下的c++filt 命令'
date: 2020-07-31 06:40:14
tags: Cpp
categories:
cover: https://man.kohod.fr/wp-content/uploads/sites/4/2016/11/x86_64-linux-gnu-cfilt.png
---
<meta name="referrer" content="no-referrer" />

一个简单的linux命令， 确实不值得大费周折， 但是， 如果能与实际开发工作联系起来， 解决实际开发中的困惑， 在生动的实际场景中学习命令， 那无疑是棒棒哒的感觉.

​    最近刚好用c++filt解决了相关实际问题， 故而分享如下：



   我们知道， 在C++中， 是允许函数重载的， 也就引出了编译器的name mangling机制， 今天我们要介绍的c++filt命令便与此有关。

   对于从事linux开发的人来说， 不可不知道c++filt命令的使用。



   在linux开发中， 如果要调用基础模块库， 就要包含对应的头文件， 并在makefile中指定头文件路径和对应的库。

​    之前我们说过了：

1. 如果没有指定对应的头文件， 则编译会报错， 提示找不到头文件。 

2. 如果指定了库路径， 但实际没有库， 则会报找不错库文件的错误。 

3. 如果没有指定库路径(因各种原因啦)， 则编译不会报错， 运行的时候才会报错， 提示dlopen失败。



   针对3中的问题， 我们之前也说过， 完全不用等到运行阶段才去发现问题， 我们可以在编译出so库后， 用ldd -r命令来找出undefined的函数名(当然也可以用nm命令)， 比如用ldd -r test.so查出缺少_ZNK4Json5ValueixEPKc(这就是name mangling后的函数名), 那怎么知道这个name mangling后的名字的原函数名称呢?  我们可以大致猜测， 但这并不靠谱， 怎么办呢？c++file命令就是专门干这个的， 如下：



```shell
[taoge@localhost test]$ c++filt _ZNK4Json5ValueixEPKc
Json::Value::operator[](char const*) const
```

   这样， 就更清楚是哪个函数了。 然后就可以在工程中搜索了， 然后就可以找到对应的库了， 然后就可以修改makefile来指定库了， 酱紫就解决问题了


   就这样。