---
title: 'Effective Modern C++ Note [1] : Smart Pointer'
date: 2020-06-19 10:36:56
tags: [Cpp,Effective Modern C++]
categories:
cover: https://m.media-amazon.com/images/S/aplus-media/mg/297b2377-6138-4ecf-9f6f-f1ee4fe08eaf.png
---
<meta name="referrer" content="no-referrer" />

## 前言

- `std::auto_ptr` 使用了它的复制操作来完成移动任务。这导致了令人奇怪的代码（拷贝一个`std::auto_ptr`会将它本身设置为null！）和令人沮丧的使用限制（比如不能将`std::auto_ptr`放入容器）。
- 除非是C++98编译的场景，否则使用`std::unique_ptr`来替换`std::auto_ptr`，不要回头。

## 条款18: 使用std::unique_ptr管理具备专属所有权的资源

- 默认情况下大小和裸指针基本相同。
- `std::unique_ptr`实现的是专属所有权语义。
- 是个只移型别，不允许复制。
- 当一个工厂函数返回的`std::unique_ptr`被移动到一个容器中，该容器的元素又被移动到某对象的数据成员中，而稍后该对象又被析构这样的场景。在这种情况下，该对象的`std::unique_ptr`数据成员将随着对象主体的析构而被析构，从而引发工厂函数返回资源也被析构。
- 默认地，资源析构采用delete运算符实现，但可以指定自定义删除器。有状态的删除器和采用函数指针实现的删除器会增加`std::unique_ptr`型别的对象尺寸。所以使用无捕获的lambda表达式是更好的选择。
- `std::unique_ptr`可以高效地转换成`std::shared_pte`，正是这一特性使得`std::unique_ptr`如此适合作为工厂函数的返回型别。

