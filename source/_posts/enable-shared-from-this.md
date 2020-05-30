---
title: enable_shared_from_this
date: 2020-05-30 08:45:52
tags: Cpp
categories:
cover: https://encrypted-tbn0.gstatic.com/images?q=tbn%3AANd9GcQrJbAooxd-acyxrC5iJ52G-U7OvEr45hKAfcz3vEJt8_8ZZavm&usqp=CAU
---
<meta name="referrer" content="no-referrer" />

>日常怀疑自己真的会写cpp？

```cpp
struct A {
  void func() {
    // only have "this" ptr ?
  }
};

int main() {
  A* a;
  std::shared_ptr<A> sp_a(a);
}
```
当A* a被shared_ptr托管的时候,如何在func获取自身的shared_ptr成了问题.
如果写成:
```cpp
void func() {
  std::shared_ptr<A> local_sp_a(this);
  // do something with local_sp_a
}
```
又用a新生成了一个shared_ptr: local_sp_a, 这个在生命周期结束的时候可能将a直接释放掉.

这里就需要用enable_shared_from_this改写:
```cpp
struct A : public enable_shared_from_this {
  void func() {
    std::shared_ptr<A> local_sp_a = shared_from_this();
    // do something with local_sp
  }
};
```
shared_from_this会从weak_ptr安全的生成一个自身的shared_ptr.