---
title: '[转载]设计模式之单例模式(c++版)'
date: 2020-07-30 02:26:29
tags: Cpp
categories:
cover: https://encrypted-tbn0.gstatic.com/images?q=tbn%3AANd9GcRGlhOHoIIJBxW84WvSTjbIoVxfiSpaLiLkpA&usqp=CAU
---
<meta name="referrer" content="no-referrer" />

## 动机

> 保证一个类仅有一个实例，并提供一个该实例的全局访问点。 ——《设计模式》GoF

在软件系统中，经常有这样一些特殊的类，必须保证他们在系统中只存在一个实例，才能确保它们的逻辑正确性、以及良好的效率。

所以得考虑如何绕过常规的构造器（不允许使用者new出一个对象），提供一种机制来保证一个类只有一个实例。

**应用场景：**

- Windows的Task Manager（任务管理器）就是很典型的单例模式，你不能同时打开两个任务管理器。Windows的回收站也是同理。
- 应用程序的日志应用，一般都可以用单例模式实现，只能有一个实例去操作文件。
- 读取配置文件，读取的配置项是公有的，一个地方读取了所有地方都能用，没有必要所有的地方都能读取一遍配置。
- 数据库连接池，多线程的线程池。

## 实现一[线程不安全版本]

```cpp
class Singleton{
public:
    static Singleton* getInstance(){
        // 先检查对象是否存在
        if (m_instance == nullptr) {
            m_instance = new Singleton();
        }
        return m_instance;
    }
private:
    Singleton(); //私有构造函数，不允许使用者自己生成对象
    Singleton(const Singleton& other);
    static Singleton* m_instance; //静态成员变量 
};

Singleton* Singleton::m_instance=nullptr; //静态成员需要先初始化
```

这是单例模式最经典的实现方式，将构造函数和拷贝构造函数都设为私有的，而且采用了延迟初始化的方式，在第一次调用`getInstance()`的时候才会生成对象，不调用就不会生成对象，不占据内存。然而，在多线程的情况下，这种方法是不安全的。

分析：正常情况下，如果线程`A`调用`getInstance()`时，将`m_instance` 初始化了，那么线程`B`再调用`getInstance()`时，就不会再执行`new`了，直接返回之前构造好的对象。然而存在这种情况，线程`A`执行`m_instance = new Singleton()`还没完成，这个时候`m_instance`仍然为`nullptr`，线程`B`也正在执行`m_instance = new Singleton()`，这是就会产生两个对象，线程`A`和`B`可能使用的是同一个对象，也可能是两个对象，这样就可能导致程序错误，同时，还会发生内存泄漏。

## 实现二[线程安全，锁的代价过高]

```cpp
//线程安全版本，但锁的代价过高
Singleton* Singleton::getInstance() {
    Lock lock; //伪代码 加锁
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}
```

分析：这种写法不会出现上面两个线程都执行`new`的情况，当线程`A`在执行`m_instance = new Singleton()`的时候，线程`B`如果调用了`getInstance()`，一定会被阻塞在加锁处，等待线程`A`执行结束后释放这个锁。从而是线程安全的。

但这种写法的性能不高，因为每次调用`getInstance()`都会加锁释放锁，而这个步骤只有在第一次`new Singleton()`才是有必要的，只要`m_instance`被创建出来了，不管多少线程同时访问，使用`if (m_instance == nullptr) `进行判断都是足够的（只是读操作，不需要加锁），没有线程安全问题，加了锁之后反而存在性能问题。

## 实现三[双检查锁，由于内存读写reoder导致不安全]

上面的做法是不管三七二十一，某个线程要访问的时候，先锁上再说，这样会导致不必要的锁的消耗，那么，是否可以先判断下`if (m_instance == nullptr) `呢，如果满足，说明根本不需要锁啊！这就是所谓的**双检查锁(DCL)**的思想，DCL即double-checked locking。

```cpp
//双检查锁，但由于内存读写reorder不安全
Singleton* Singleton::getInstance() {
    //先判断是不是初始化了，如果初始化过，就再也不会使用锁了
    if(m_instance==nullptr){
        Lock lock; //伪代码
        if (m_instance == nullptr) {
            m_instance = new Singleton();
        }
    }
    return m_instance;
}
```

这样看起来很棒！只有在第一次必要的时候才会使用锁，之后就和`实现一`中一样了。

在相当长的一段时间，迷惑了很多人，在`2000`年的时候才被人发现漏洞，而且在每种语言上都发现了。原因是内存读写的乱序执行（编译器的问题）。

分析：`m_instance = new Singleton()`这句话可以分成三个步骤来执行：

1. 分配了一个`Singleton`类型对象所需要的内存。
2. 在分配的内存处构造`Singleton`类型的对象。
3. 把分配的内存的地址赋给指针`m_instance`。

可能会认为这三个步骤是按顺序执行的，但实际上只能确定步骤`1`是最先执行的，步骤`2`，`3`却不一定。问题就出现在这。假如某个线程A在调用执行`m_instance = new Singleton()`的时候是按照`1,3,2`的顺序的，那么刚刚执行完步骤`3`给`Singleton`类型分配了内存（此时`m_instance`就不是`nullptr`了）就切换到了线程`B`，由于`m_instance`已经不是`nullptr`了，所以线程`B`会直接执行`return m_instance`得到一个对象，而**这个对象并没有真正的被构造**！！严重bug就这么发生了。

## 实现四[C++ 11版本的跨平台实现]

`java`和`c#`发现这个问题后，就加了一个关键字`volatile`，在声明`m_instance`变量的时候，要加上`volatile`修饰，编译器看到之后，就知道这个地方不能够reorder（一定要先分配内存，在执行构造器，都完成之后再赋值）。

而对于`c++`标准却一直没有改正，所以`VC++`在`2005`版本也加入了这个关键字，但是这并不能够跨平台（只支持微软平台）。

而到了`c++ 11`版本，终于有了这样的机制帮助我们实现跨平台的方案。

```cpp
//C++ 11版本之后的跨平台实现 
// atomic c++11中提供的原子操作
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;

/*
* std::atomic_thread_fence(std::memory_order_acquire); 
* std::atomic_thread_fence(std::memory_order_release);
* 这两句话可以保证他们之间的语句不会发生乱序执行。
*/
Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);//获取内存fence
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            std::atomic_thread_fence(std::memory_order_release);//释放内存fence
            m_instance.store(tmp, std::memory_order_relaxed);
        }
    }
    return tmp;
}
```

## 实现五[pthread_once函数]

在linux中，`pthread_once()`函数可以保证某个函数只执行一次。

```
声明: int pthread_once(pthread_once_t once_control, void (init_routine) (void))；

功能: 本函数使用初值为PTHREAD_ONCE_INIT的once_control
变量保证init_routine()函数在本进程执行序列中仅执行一次。 
```

示例如下：

```cpp
class Singleton{
public:
    static Singleton* getInstance(){
        // init函数只会执行一次
        pthread_once(&ponce_, &Singleton::init);
        return m_instance;
    }
private:
    Singleton(); //私有构造函数，不允许使用者自己生成对象
    Singleton(const Singleton& other);
    //要写成静态方法的原因：类成员函数隐含传递this指针（第一个参数）
    static void init() {
        m_instance = new Singleton();
      }
    static pthread_once_t ponce_;
    static Singleton* m_instance; //静态成员变量 
};
pthread_once_t Singleton::ponce_ = PTHREAD_ONCE_INIT;
Singleton* Singleton::m_instance=nullptr; //静态成员需要先初始化
```

## 实现六[c++ 11版本最简洁的跨平台方案]

实现四的方案有点麻烦，实现五的方案不能跨平台。其实`c++ 11`中已经提供了`std::call_once`方法来保证函数在多线程环境中只被调用一次，同样，他也需要一个`once_flag`的参数。用法和`pthread_once`类似，并且支持跨平台。

实际上，还有一种最为简单的方案！

> 在C++memory model中对static local variable，说道：The initialization of such a variable is defined to occur the first time control passes through its declaration; for multiple threads calling the function, this means there’s the potential for a race condition to define first.

局部静态变量不仅只会初始化一次，而且还是线程安全的。

```cpp
class Singleton{
public:
    // 注意返回的是引用。
    static Singleton& getInstance(){
        static Singleton m_instance;  //局部静态变量
        return m_instance;
    }
private:
    Singleton(); //私有构造函数，不允许使用者自己生成对象
    Singleton(const Singleton& other);
};
```

这种单例被称为`Meyers' Singleton `。这种方法很简洁，也很完美，但是注意：

1. gcc 4.0之后的编译器支持这种写法。
2. C++11及以后的版本（如C++14）的多线程下，正确。
3. C++11之前**不能**这么写。

但是现在都18年了。。新项目一般都支持了`c++11`了。

## 用模板包装单例

从上面已经知道了单例模式的各种实现方式。但是有没有感到一点不和谐的地方？如果我`class A`需要做成单例，需要这么改造`class A`，如果`class B`也需要做成单例，还是需要这样改造一番，是不是有点重复劳动的感觉？利用`c++`的模板语法可以避免这样的重复劳动。

```cpp
template<typename T>
class Singleton
{
public:
    static T& getInstance() {
        static T value_; //静态局部变量
        return value_;
    }

private:
    Singleton();
    ~Singleton();
    Singleton(const Singleton&); //拷贝构造函数
    Singleton& operator=(const Singleton&); // =运算符重载
};
```

假如有`A`，`B`两个类，用`Singleton`类可以很容易的把他们也包装成单例。

```cpp
class A{
public:
    A(){
       a = 1;
    }
    void func(){
        cout << "A.a = " << a << endl;
    }

private:
    int a;
};

class B{
public:
    B(){
        b = 2;
    }

    void func(){
        cout << "B.b = " << b << endl;
    }
private:
    int b;
};

// 使用demo
int main()
{
    Singleton<A>::getInstance().func();
    Singleton<B>::getInstance().func();
    return 0;
}
```

假如类`A`的构造函数具有参数呢？上面的写法还是没有通用性。可以使用`C++11`的可变参数模板解决这个问题。但是感觉实际中这种需求并不是很多，因为构造只需要一次，每次`getInstance()`传个参数不是很麻烦吗。。。

## 总结

单例模式本身十分简单，但是实现上却发现各种麻烦，主要是多线程编程确实是个难点。而对于`c++`的对象模型、内存模型，并没有什么深入的了解，还在一知半解的阶段，仍需努力。

需要注意的一点是，上面讨论的线程安全指的是`getInstance()`是线程安全的，假如多个线程都获取类`A`的对象，如果只是只读操作，完全OK，但是如果有线程要修改，有线程要读取，那么类`A`自身的函数需要自己**加锁防护**，不是说线程安全的单例也能保证修改和读取该对象自身的资源也是线程安全的。

## 参考

1. [C++中多线程与Singleton的那些事儿](https://www.cnblogs.com/liyuan989/p/4264889.html)
2. boolan c++设计模式