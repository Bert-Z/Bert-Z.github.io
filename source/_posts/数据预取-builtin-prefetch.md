---
title: 数据预取 __builtin_prefetch()
date: 2021-02-02 13:54:40
tags: ['Linux']
categories:
cover: https://www.naftaliharris.com/blog/2x-speedup-with-one-line-of-code/times.png
---
<meta name="referrer" content="no-referrer" />

# 数据预取 __builtin_prefetch()

__builtin_prefetch() 是 gcc 的一个内置函数。它通过对数据手工预取的方法，减少了读取延迟，从而提高了性能，但该函数也需要 CPU 的支持。

该函数的原型为：

```cpp
void __builtin_prefetch ( const void *addr, ...)
```



其中参数 addr 是个内存指针，它指向要预取的数据，我们人工需要判定这些数据是很快能访问到的，或者说是它们就在最近的内存中 --- 一般来说，对于链表而言，各个节点在内存中基本上是紧挨着的，所以我们容易预取链表节点里的指针项。

该函数还有两个可选参数，rw 和 locality 。

rw 是个编译时的常数，或 1 或 0 。1 时表示写(w)，0 时表示读(r) 。

locality 必须是编译时的常数，也称为“时间局部性”(temporal locality) 。时间局部性是指，如果程序中某一条指令一旦执行，则不久之后该指令可能再被执行；如果某数据被访问，则不久之后该数据会被再次访问。该值的范围在 0 - 3 之间。为 0 时表示，它没有时间局部性，也就是说，要访问的数据或地址被访问之后的不长的时间里不会再被访问；为 3 时表示，被访问的数据或地址具有高 时间局部性，也就是说，在被访问不久之后非常有可能再次访问；对于值 1 和 2，则分别表示具有低 时间局部性 和中等 时间局部性。该值默认为 3 。

用法：

```cpp
for (i = 0; i < n; i++)
{
  a[i] = a[i] + b[i];
  __builtin_prefetch (&a[i+j], 1, 1);
  __builtin_prefetch (&b[i+j], 0, 1);
}
```



另外，参数 addr 是需要用户保证地址的正确，函数不会对不正确的地址做出错误的提示。

在 linux 内核中，经常会使用到这种预抓取技术。通常是通过宏和包装器函数使用预抓取。下面是一个辅助函数示例，它使用内置函数的包装器（见 ./linux/include/linux/prefetch.h）。这个函数为流操作实现预抓取机制。使用这个函数通常可以减少缓存缺失和停顿，从而 提高性能。

```cpp
#ifndef ARCH_HAS_PREFETCH
#define prefetch(x) __builtin_prefetch(x)
#endif
static inline void prefetch_range(void *addr, size_t len) 
{
  #ifdef ARCH_HAS_PREFETCH
  char *cp;
  char *end = addr + len;
  for (cp = addr; cp < end; cp += PREFETCH_STRIDE)
    prefetch(cp);
  #endif
}
```