---
title: '[转载]STL源码剖析之算法：copy & copy_backward'
date: 2020-08-31 06:18:55
tags: Cpp
categories:
cover: https://s4.51cto.com/attachment/201303/135652149.png
---
<meta name="referrer" content="no-referrer" />


  copy() 是一个调用频率非常高的函数，所以SGI STL的copy算法用尽各种办法，包括函数重载(function overloading)、型别特性(type traits)、偏特化(partial specialization) 编程技巧，无所不用其极地加强效率。下图是整个copy()操作的脉络。

[![img](https://s4.51cto.com/attachment/201303/135652149.png)](https://s4.51cto.com/attachment/201303/135652149.png)

  copy算法将输入区间[first, last)内的元素复制到result指向的输出区间内，赋值操作是向前推进的。如果输入区间和输出区间重叠，复制顺序需要多加讨论。当result位于[first, last)之内时，也就是说，如果输出区间的首部与输入区间重叠，copy的结果可能不正确，建议选用copy_backward；如果输出区间的尾部如输入区间重叠，copy_backward的结果可能不正确，建议选用copy。当然，如果两区间完全不重叠，copy和copy_backward都可以选用。

  copy算法根据输出迭代器的特性决定是否调用memmove()来执行任务，memmove()会先将整个输入区间的内容复制下来，然后再复制到输入区间。这种情况下，即使输入区间和输出区间有重叠时，copy的结果也是正确的。这也回答了上文中提调的，为什么result位于[first, last)之内时，copy的结果只是“可能不正确”。

  copy为输出区间内的元素赋予新值，而不是产生新元素，它不能改变输出区间的长度。换句话说，copy不能用来直接将元素插入到空容器中。如果你想要将元素插入序列之中，要么使用序列容器的insert成员函数，要么使用copy算法并搭配insert_iterator。

  下面是copy算法唯三的对外接口，包括一个完全泛化版本和两个重载函数：

```cpp
template <class InputIterator, class OutputIterator> 
inline OutputIterator
copy(InputIterator first, InputIterator last, OutputIterator result) { 
    return __copy_dispatch<InputIterator, OutputIterator>()(first, last, result); 
} 
inline char* copy(const char* first, const char* last, char* result) { 
    memmove(result, first, last - first); 
    return result + (last - first); 
} 
inline wchar_t* copy(const wchar_t* first, const wchar_t* last, wchar_t* result) { 
    memmove(result, first, last - first); 
    return result + (last - first); 
} 
```

  copy()函数的泛化版本中调用了一个__copy_dispatch()的仿函数（老方法了^_^），此仿函数有一个完全泛化版本和两个偏特化版本：

```cpp
template <class InputIterator, class OutputIterator>  
struct __copy_dispatch {  
    OutputIterator operator(){InputIterator first, InputIterator last,  
OutputIterator result) {  
        return __copy(first, last, result, iterator_category(first);  
} ; 
 
/*  
 *  __copy_dispatch()的完全泛化版本根据迭代器种类的不同，调用 
 *  不同的__copy()，为的是不同的迭代器使用的循环条件不同，有快
 * 慢之别 
 */ 
 
template <class InputIterator, class OutputIterator> 
inline OutputIterator __copy(InputIterator first, InputIterator last, 
OutputIterator result, input_iterator_tag) { 
    for( ; first != last; ++first, ++result) 
        *result = *first; 
    return result; 
} 
template <class RandomAccessIterator, class OutputIterator> 
inline OutputIterator __copy(RandomAccessIterator first, RandomAccessIterator last, 
OutputIterator result, random_access_iterator_tag) { 
    __return __copy_d(first, last, result, distance_typ(first)); 
} 
template <class RandomAccessIterator, class OutputIterator, class Distance> 
inline OutputIterator __copy_d(RandomAccessIterator first, RandomAccessIterator last, 
OutputIterator result, Distance*) { 
    // 以n决定循环次数，速度快 
    for(Distance n = last - first; n > 0; --n, ++result, ++first) { 
        *result = *first; 
    return result; 
} 
    
template <class T>  
struct __copy_dispatch<T*, T*> {  
    T* operator()(T* first, T* last, T* result) {  
        typedef typename __type_traits<T>::has_trivial_assignment_operator t;  
        return __copy_t(first, last, result, t());  
};  
template <class T>  
struct __copy_dispatch<const T*, T*> {  
    T* operator()(T* first, T* last, T* result) {  
        typedef typename __type_traits<T>::has_trivial_asssignment_operator t;  
        return __copy_t(first, last, result, t());  
};  
/*
* 这两个偏特化版本是在“参数为原生指针形式”的前提下，利用 
 * __type_traits<>编程技巧来探测指针所指向之物是否有 
 * trivial assignment operator. 如果指针所指对象拥有 
 * trivial assignment operator，则可以通过memmove()进行复制，速度要比 
 * 利用赋值操作赋快许多
*/
template <class T> 
inline T* __copy_t(const T* first, const T* last, T* result, __true_type)  { 
    memmove(result, first, sizeof(T) *(last - first); 
    return result + (last - first); 
} 
template <class T> 
inline T* __copy_t(const T* first, const T* last, T* result, __false_type) { 
    return __copy_d(first, last, result, (ptrdiff_t *)0); 
} 

```

  以上就是copy()的故事，一个无所不用其极强化效率的故事。

  copy_backward()的实现与copy()极为相似，不同是它将[first, last)区间内的每一个元素，以逆行的方向复制到，以result-1为起点，方向同样为逆行的区间上。