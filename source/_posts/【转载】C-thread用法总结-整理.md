---
title: 【转载】C++ thread用法总结(整理)
date: 2020-07-18 15:13:01
tags: Cpp
categories:
cover: https://www.bogotobogo.com/cplusplus/C11/images/thread_1_intro/real_fork_join.png
---
<meta name="referrer" content="no-referrer" />

## 简介

C++11中加入了<thread>头文件，此头文件主要声明了std::thread线程类。C++11的标准类std::thread对线程进行了封装，定义了C++11标准中的一些表示线程的类、用于互斥访问的类与方法等。应用C++11中的std::thread便于多线程程序的移值。

std::thread类成员函数：

(1)、get_id：获取线程ID，返回一个类型为std::thread::id的对象。

(2)、joinable：检查线程是否可被join。检查thread对象是否标识一个活动(active)的可行性线程。缺省构造的thread对象、已经完成join的thread对象、已经detach的thread对象都不是joinable。

(3)、join：调用该函数会阻塞当前线程。阻塞调用者(caller)所在的线程直至被join的std::thread对象标识的线程执行结束。

(4)、detach：将当前线程对象所代表的执行实例与该线程对象分离，使得线程的执行可以单独进行。一旦线程执行完毕，它所分配的资源将会被释放。

(5)、native_handle：该函数返回与std::thread具体实现相关的线程句柄。native_handle_type是连接thread类和操作系统SDK API之间的桥梁，如在Linux g++(libstdc++)里，native_handle_type其实就是pthread里面的pthread_t类型，当thread类的功能不能满足我们的要求的时候(比如改变某个线程的优先级)，可以通过thread类实例的native_handle()返回值作为参数来调用相关的pthread函数达到目录。This member function is only present in class thread if the library implementation supports it. If present, it returns a value used to access implementation-specific information associated to the thread.

(6)、swap：交换两个线程对象所代表的底层句柄。

(7)、operator=：moves the thread object

(8)、hardware_concurrency：静态成员函数，返回当前计算机最大的硬件并发线程数目。基本上可以视为处理器的核心数目。

另外，std::thread::id表示线程ID，定义了在运行时操作系统内唯一能够标识该线程的标识符，同时其值还能指示所标识的线程的状态。Values of this type are returned by thread::get_id and this_thread::get_id to identify threads.

有时候我们需要在线程执行代码里面对当前调用者线程进行操作，针对这种情况，C++11里面专门定义了一个命名空间this_thread，此命名空间也声明在<thread>头文件中，其中包括get_id()函数用来获取当前调用者线程的ID；yield()函数可以用来将调用者线程跳出运行状态，重新交给操作系统进行调度，即当前线程放弃执行，操作系统调度另一线程继续执行；sleep_until()函数是将线程休眠至某个指定的时刻(time point),该线程才被重新唤醒；sleep_for()函数是将线程休眠某个指定的时间片(time span)，该线程才被重新唤醒，不过由于线程调度等原因，实际休眠实际可能比sleep_duration所表示的时间片更长。

1.创建一个线程

创建线程比较简单，使用std的thread实例化一个线程对象就创建完成了，示例：

```cpp
#include <iostream>
#include <thread>
#include <stdlib.h> //sleep

using namespace std;

void t1()  //普通的函数，用来执行线程
{
    for (int i = 0; i < 10; ++i)
    {
        cout << "t1111\n";
        sleep(1);
    }
}
void t2()
{
    for (int i = 0; i < 20; ++i)
    {
        cout << "t22222\n";
        sleep(1);
    }
}
int main()
{
    thread th1(t1);  //实例化一个线程对象th1，使用函数t1构造，然后该线程就开始执行了（t1()）
    thread th2(t2);

    th1.join(); // 必须将线程join或者detach 等待子线程结束主进程才可以退出
    th2.join(); 
     
    //or use detach
    //th1.detach();
    //th2.detach();
     
    cout << "here is main\n\n";
     
    return 0;
}
```

上述提到的问题，还可以使用detach来解决，detach是用来和线程对象分离的，这样线程可以独立地执行，不过这样由于没有thread对象指向该线程而失去了对它的控制，当对象析构时线程会继续在后台执行，但是当主程序退出时并不能保证线程能执行完。如果没有良好的控制机制或者这种后台线程比较重要，最好不用detach而应该使用join。

## mutex和std::lock_guard的使用

头文件是#include <mutex>，mutex是用来保证线程同步的，防止不同的线程同时操作同一个共享数据。

但使用lock_guard则相对安全，它是基于作用域的，能够自解锁，当该对象创建时，它会像m.lock()一样获得互斥锁，当生命周期结束时，它会自动析构(unlock)，不会因为某个线程异常退出而影响其他线程。示例：

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <stdlib.h>

int cnt = 20;
std::mutex m;
void t1()
{
    while (cnt > 0)
    {
        std::lock_guard<std::mutex> lockGuard(m);
        // std::m.lock();
        if (cnt > 0)
        {
            //sleep(1);
            --cnt;
            std::cout << "1:" << cnt << std::endl;
        }
        // std::m.unlock();
    }
}
void t2()
{
    while (cnt > 0)
    {
        std::lock_guard<std::mutex> lockGuard(m);
        // std::m.lock();
        if (cnt > 0)
        {
            --cnt;
            std::cout << "2:" << cnt << std::endl;
        }
        // std::m.unlock();
    }
}

int main(void)
{
    std::thread th1(t1);
    std::thread th2(t2);

    th1.join(); //等待t1退出
    th2.join(); //等待t2退出

    std::cout << "here is the main()" << std::endl;

    return 0;
}
```

输出结果，cnt是依次递减的，没有因为多线程而打乱次序：：

```shell
1:19
1:18
1:17
1:16
1:15
1:14
1:13
1:12
1:11
1:10
1:9
1:8
1:7
1:6
1:5
1:4
1:3
1:2
1:1
1:0
here is the main()
```
