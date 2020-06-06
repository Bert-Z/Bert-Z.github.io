---
title: '[转载]【C++基础之十】友元函数和友元类'
date: 2020-05-30 08:30:29
tags: Cpp
categories:
cover: https://img-blog.csdnimg.cn/20190618214113419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZlaXQyNDE3,size_16,color_FFFFFF,t_70
---
<meta name="referrer" content="no-referrer" />

# [转载]【C++基础之十】友元函数和友元类

## 1.概述

友元提供了一种 普通函数或者类成员函数 访问另一个类中的私有或保护成员 的机制。也就是说有两种形式的友元：

（1）友元函数：普通函数对一个访问某个类中的私有或保护成员。

（2）友元类：类A中的成员函数访问类B中的私有或保护成员。



## 2.特性

优点：提高了程序的运行效率。

缺点：破坏了类的封装性和数据的透明性。



## 3.实现

### 3.1.友元函数

#### 3.1.1.声明和定义

在类声明的任何区域中声明，而定义则在类的外部。

> friend <类型><友元函数名>(<参数表>);
>
> 注意，友元函数只是一个普通函数，并不是该类的类成员函数，它可以在任何地方调用，友元函数中通过对象名来访问该类的私有或保护成员。

#### 3.1.2.示例

```cpp
// TestFriend.cpp

#include "stdafx.h"

class A
{
public:
	A(int _a):a(_a){};
  friend int getA_a(A &_classA);//友元函数

private:
	int a;
};

int getA_a(A &_classA)
{
	return _classA.a;//通过对象名访问私有变量
}

int _tmain(int argc, _TCHAR* argv[])
{
	A _classA(3);
	std::cout<<getA_a(_classA);//友元函数只是普通函数，可以在任意地方调用
	return 0;
}
```

### 3.2.友元类

#### 3.2.1.声明和定义

友元类的声明在该类的声明中，而实现在该类外。

> friend class <友元类名>;

友元类的实例则在main函数中定义。

#### 3.2.2.示例

```cpp
// TestFriend.cpp

#include "stdafx.h"

class B
{
public:
	B(int _b):b(_b){};

	friend class C;//声明友元类C

private:
	int b;
};

class C//实现友元类C
{
public:
	int getB_b(B _classB){
		return _classB.b;//访问友元类B的私有成员
	};
};


int _tmain(int argc, _TCHAR* argv[])
{
	B _classB(3);
	C _classC;//定义友元类实例
	std::cout<<_classC.getB_b(_classB);
	return 0;
}
```

## 4.注意

### 4.1.友元关系没有继承性

假如类B是类A的友元，类C继承于类A，那么友元类B是没办法直接访问类C的私有或保护成员。

### 4.2.友元关系没有传递性

加入类B是类A的友元，类C是类B的友元，那么友元类C是没办法直接访问类A的私有或保护成员，也就是不存在“友元的友元”这种关系。

