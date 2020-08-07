---
title: '[转载]C++移动语义及拷贝优化'
date: 2020-08-07 08:54:15
tags: Cpp
categories:
cover: https://img-blog.csdn.net/20170702224228434?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvWm9jdG9wdXNE/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
---
<meta name="referrer" content="no-referrer" />

# C++移动语义及拷贝优化

我们知道在传统C++程序中，如果函数的返回值是一个对象的话，可能需要对函数中的局部对象进行拷贝。如果该对象很大的话，则程序的效率会降低。

在C++ 11以后，出现的移动语义（Move Semantic）及拷贝优化（Copy Elision）都是解决这个问题的方法。这篇博文简单探探这些技术。

## 再谈移动语义

对于C++ 11移动语义的介绍，我之前写过一篇博客《[C++11中的移动语义](https://blog.csdn.net/theonegis/article/details/50512469)》进行了介绍，这里我再进行简单的总结。

### 左值和右值

C++中如何区分一个变量是左值还是右值呢？

1. 左值一般是可寻址的变量，右值一般是不可寻址的字面常量或者是在表达式求值过程中创建的可寻址的无名临时对象；
2. 左值具有持久性，右值具有短暂性。

左值引用的符号为“&”（传统C++中的引用）；右值引用的符号为“&&”（C++ 11中的新特性）

### 移动构造函数和移动赋值函数

移动语义和拷贝语义是相对于的，移动类似于计算机中对文件操作的剪切，而拷贝类似于文件的复制。

我们可以定义拷贝构造函数和赋值函数进行对象的复制，如果没有定义，编译器会帮我们生产默认的实现。要实现转移语义，需要定义转移构造函数，当然还可以定义转移赋值操作符。对于右值的拷贝和赋值会调用转移构造函数和转移赋值操作符。如果转移构造函数和转移拷贝操作符没有定义，那么拷贝构造函数和赋值操作符会被调用。

移动构造函数和移动赋值函数都是形参（Parameter）为右值引用的函数，下面看一个例子。

```
struct Foo {  Foo() { cout << "Constructed" << endl; }  Foo(const Foo &) { cout << "Copy-constructed" << endl; }  Foo(Foo &&) { cout << "Move-constructed" << endl; }  ~Foo() { cout << "Destructed" << endl; }};
```

可以看到第4行的移动构造函数就是一个形参为右值引用的构造器。

我们通过一个示例观察其输出：

```
int main() {     vector<Foo> vec;    vec.push_back(Foo());    return 0; }
```

这里使用`g++`或者`clang++`编译器进行编译运行：`g++-8 foo.cpp -std=c++11 && ./a.out`

我们首先注释掉`Foo`定义中的第4行的移动构造函数，结果如下：

```
ConstructedCopy-constructedDestructedDestructed
```

可以看到拷贝构造函数被调用了。在主函数中的第3上，`Foo()`会生成一个右值对象（调用默认构造函数），然后进行拷贝构造以后传递给`vec`集合。

如果我们加上移动构造函数，则运行结果如下：

```
ConstructedMove-constructedDestructedDestructed
```

这时，因为`Foo()`是右值，所以调用了移动构造函数。

NOTE：拷贝构造函数中是对传进来的对象进行了实实在在的拷贝工作；而移动构造函数中只是对传进来的对象进行了所有权的转让，即掏空传进来的对象，然后把所有权转给当前对象（`this`指针指向的那个对象）。

### std::move函数

编译器只对右值引用才能调用转移构造函数和转移赋值函数，而所有命名对象都只能是左值引。如果已知一个命名对象不再被使用而想对它调用转移构造函数和转移赋值函数，也就是把一个左值引用当做右值引用来使用，怎么实现呢？标准库提供了函数`std::move`，这个函数以非常简单的方式将左值引用转换为右值引用。

`std::move`的实现即使将一个对象强制转型为右值引用类型的对象而已，并不做任何移动工作。

## 拷贝优化

现在说说第二个问题拷贝优化（Copy Elision），这是一个编译器端的技术，而移动语义是代码端的技术。虽然两者都可以减少不必要的拷贝工作。

一般来说，对于支持拷贝优化的编译器会优先执行拷贝优化，如果不能进行拷贝优化，则调用移动构造函数，如果没有定义移动构造函数，则调用拷贝构造函数。当然，拷贝优化效率最高，移动构造次之。

拷贝优化在两种情况下进行：一是对于函数返回值的拷贝优化；而是对于向函数中传递临时对象的优化。

### 返回值的优化

返回值的优化分为Named Return Value Optimization (NRVO)和Regular Return Value Optimization (RVO)

还是以`Foo`为例，我们定义如下两个函数：

```
// Named Return Value Optimization (NRVO)Foo f1() {  Foo foo;  return foo;}// Return Value Optimization (RVO)Foo f2() {    return Foo();}int main() {     f1();    return 0; }
```

运行结果如下：

```
ConstructedDestructed
```

可以看到并没有拷贝构造或者移动构造的发生。虽然理论上说，`f1()`函数的返回值是局部变量，会有一次拷贝构造的发生，但是实际并没有。这是因为编译器帮我们做了优化，减少了不必要的拷贝。

```
g++`和`clang++`都提供了`-fno-elide-constructors`选项可以关闭拷贝优化，我们重新进行编译运行`g++-8 foo.cpp -std=c++11 -fno-elide-constructors && ./a.out
```

结果如下，可以看到发生了一次移动构造（如果没有定义移动构造函数的话，就会调用拷贝构造函数）

```
ConstructedMove-constructedDestructedDestructed
```

`f1()`和`f2()`会有相同的运行结果

我们再来修改一下`main()`函数：

```
int main() {     Foo foo = f1();    return 0; }
```

猜一下，在有拷贝优化和没有拷贝优化的情况下会发生什么？

如果没有拷贝优化的结果如下：

```
ConstructedMove-constructedDestructedMove-constructedDestructedDestructed
```

可以看到发生了两次移动拷贝，第一次是在函数局部对象进行返回的时候拷贝到了一个临时对象中，第二次是将该临时对象用以初始化`foo`变量（注意对象初始化跟赋值的区别）。

而如果有拷贝优化呢？

```
ConstructedDestructed
```

一次移动构造或者拷贝构造都没有，是不是很爽。

### 传递临时对象的优化

对于函数参数传递的优化，示例如下：

```
// Passing a Temporary by Valuevoid f3(Foo f) {    cout << "F3 called" << endl;}int main() {     f3(Foo());    return 0; }
```

没有拷贝优化的结果如下：

```
ConstructedMove-constructedF3 calledDestructedDestructed
```

有拷贝优化的结果如下：

```
ConstructedF3 calledDestructed
```

There is always a but…

拷贝优化不总是生效的，就是有时候拷贝优化不能成功实施。下面举一个例子：

```
// Copy Elision does not always workFoo f4(int i) {    Foo x, y;    if (i > 0) return x;    else return y;}int main() {     Foo foo = f4(0);    return 0; }
```

有拷贝优化的结果：

```
ConstructedConstructedMove-constructedDestructedDestructedDestructed
```

没有拷贝优化的结果：

```
ConstructedConstructedMove-constructedDestructedDestructedMove-constructedDestructedDestructed
```

可以看到，编译器的拷贝优化只是把在`foo`变量初始化过程中的移动构造函数给优化掉了，而`f4()`函数的返回值并没有得到优化。这是因为由于`if...else…`分支结构的存在，编译器不确定`f()`函数具体的返回对象，无法实施优化。

## 结论

C++移动语义即提出了一个右值引用，使用`std::move`可以强制将左值引用转为右值引用。而对于右值引用，程序可以调用移动构造函数进行对象的构造，减少了原来调用拷贝构造函数的时候很大的开销。移动构造函数和移动赋值运算符的实现即是对象所有权的转让，让那些左值对象（临时对象）变成右值对象的过程。

编译器的拷贝优化确实效率很高，但是不能保证总是成功实施的。所以，好的编程习惯应该是对于自定义的类最好添加移动构造函数，重载移动赋值运算符。这样编译器的拷贝优化不成功的时候，可以调用移动构造减轻复制的开销，提高程序运行的效率。

顺便提一下，在C++11以前，我们的编程习惯是为了减少不必要的复制操作，我们可能会把需要返回的对象以对象引用（左值引用，当时还没有右值引用的说法）的形式传进函数，这样在函数之外我们也可以不用拷贝获得该对象。

所以C++移动语义和拷贝优化确实是C++规范中很重要的特征，对我们写程序有很大的影响。

顺便提一下STL中的容器都提供了对右值引用的重载，所以当我们自定义类中实现了移动构造函数，使用STL容器的时候就没有多大的拷贝开销了，效率会有很大的提升。

## 参考文献

1. [右值引用与转移语义](https://www.ibm.com/developerworks/cn/aix/library/1307_lisl_c11/index.html)
2. [Guaranteed Copy Elision](https://jonasdevlieghere.com/guaranteed-copy-elision/)