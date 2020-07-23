---
title: '[转载]Linux上C++类的动态加载'
date: 2020-07-23 07:45:48
tags: Cpp
categories:
cover: https://blog-1251688504.cos.ap-shanghai.myqcloud.com/201906/fake_dlopen_flow.png
---
<meta name="referrer" content="no-referrer" />

### 1. 前言

*一种可以让开发者进行更灵活设计的技术。*

Linux操作系统下的开发平台提供了很好的环境：自带丰富的测试工具，健壮的操作环境。令Linux引以为豪的是，它能适应各种编程语言。下面的说法，相信并不为过：对Linux开发者来，在所有的编译语言中，选择C作为开发语言是最多的。由此，像C++这样的语言似乎经常不在Linux开发者的讨论范围内。

类的动态加载技术可以让开发者的设计更灵活。类的动态加载可以使实现更具扩展性，而不牺牲鲁棒性。

本文将设计一个简单的应用，这个应用只有一个类：用于绘图包的形状类。我们将会看到，类的动态加载技术可以让我们平滑地扩展功能——增加新的形状类而不修改原有代码。

### 2. 多态

动态加载类的基本思想就是类的多态性。任何熟悉C++的人对此都不会陌生，这里就只简洁地说明一下。总的来说，多态，就是子类对象可以像父类对象的那样表现。这就是OOP（object-oriented programming，面向对象）中众所周知的“**is a**”关系。比如，下面的代码段中，类**circle**“is a”基类**shape**的子类（见列表1），因而对象**my_circle**可以像shape对象一样操作shape的成员函数**draw**。

**列表1 基类shape的头文件**

 

```
class shape {
public:
  void draw();
};
class circle : public shape { };
 
int main(int argc, char **argv){
  circle my_circle;
  my_circle.draw();
}
```



这包括所有的常规优点（如代码复用）以外，多态性的真正优势在于draw声明为虚函数或者纯虚函数的时候，如下：

 

```
class shape{
public:
  virtual void draw()=0;
};
class circle : public shape {
public:
  void draw();
}
```



 

这里circle定义了自己的draw函数，以确保circle对象有正确的行为。同样，我们可以定义一些新的shape子类，并重写draw函数。这样，全部的子类都实现了shape的接口，进而可以创建一系列表现不同的对象，这些对象都调用同一个方法（调用draw成员函数）。下面是例子：

 

```
shape *shape_list[3];   // the array that will
                            // pointer to our shape objects
shape[0] = new circle;  // three types of shapes
shape[1] = new square;  // we have defined
shape[2] = new triangle;
for(int i = 0; i < 3; i++){
  shape_list[i].draw();
}
```



 

调用draw函数时，并不需要知道列表中对象的其他信息；C++会正确地调用对应的draw函数。这种强大的技术可以让我们的设计更灵活。现在我们可以通过继承shape来实现我们期望的行为。这里的关键是我们可以把接口（shape的原型）和实现分开。

虽然此技术相当强大，但如果我们想添加新的继承函数的话，我们仍不得不重新编译代码。如果能够在运行时加载新类，那将会非常方便。而且，使用我们代码库的人完全可以提供新的形状类（重写draw函数）而不必需要我们的原始代码。好消息是，这是可能的，这也是本文讨论的主题。

### 3. dlopen接口和类的动态加载

虽然Linux下还没有直接的机制可以在运行时加载C++类，但是有一个直接在运行时加载C库的机制：dl函数**dlopen**，**dlsym**，**dlerror**和**dlclose**。这组函数提供了访问动态链接器ld的方法。完整的说明可以参考这些函数的man手册，这里仅简要说明。

函数原型如下：

 

```
void *dlopen(const char * filename, int flag);
void *dlsym(void *handle, char*symbol);
const char *dlerror();
int dlclose(void *handle);
```



 

dlopen函数通过文件名filename打开so文件，文件中的符号通过dlsym函数读取。参数flag可以取下面的值：**RTLD_LAZY**和**RTLD_NOW**。如果flag设置为RTLD_LAZY，那么dlopen会不解析任何符号就返回。如果**flag**设置为**RTLD_NOW**，那么dlopen会尝试解析文件中所有未定义的符号。如果出现解析错误，函数调用失败，返回NULL。dlerror解析失败的原因。dlsym函数用于读取库中函数（或其他符号）的指针。**handle**是指向被引用项的指针，**symbol**是被引用项在实际保存文件中的字符串名字。

假如可以通过这些函数访问C库里的函数，要样才能利用它们访问C++库？要达到这个目的，有几个问题需要解决。一个是必须能够定位到目标库函数的符号。由于C和C++文件中保存的符号是不一样的，导致解决这个问题会比看起来麻烦一些。另一个问题是要如何创建目标类的对象？最后，访问这些对象的便捷方法又是什么？下面将反过来回答这三个问题。

因为并不知道动态加载的类的类型，在代码中应该如何访问？答案的诀窍在于前面提到的多态。可以通过基类提供的公用接口来使用新类的功能。延续上面的例子，新的shape类会重写draw函数，并在实际的对象中正确调用自己的draw函数。

好，现在可以通过基类的指针访问子类的对象了。怎样在开始的地方创建这些对象？除了知道它们可以调用shape的接口外，并不知道目标类的其他相关信息。举个例子，假如动态地加载了一个库，这个库中有个类**hexapod**，如果不能提前知道类名，不能这样写：

 

```
shape *my_shape = new hexapod;
```



 

解决办法是，主程序不负责创建对象，至少不是直接创建。库中提供的shape子类必须提供一个创建对象的方法。这个可以用**工厂类**来实现，就像工厂模式那样，或者直接调用一个函数创建。为了简化说明，这里直接用一个函数来创建。所有形状类的函数返回值都一样：

 

```
shape *maker();
```



 

**maker**函数没有参数，返回构建成功的对象指针。针对前面的hexapod类，maker函数如下：

 

```
shape *maker(){
  return new hexapod;
}
```



 

使用**new**来创建新的对象是绝对合法的，因为maker函数和hexapod定义是在同一个文件中。

现在，先用dlopen加载一个库，然后用dlsym得到对应类的maker函数指针。使用这个指针创建对应类的对象。例，假如想要动态链接库libnewshapes.so，并应用其中的hexapod类。可以像这样子进行：

 

```
void *hndl =dlopen("libnewshapes.so", RTLD_NOW);
if(hndl == NULL){
  cerr << dlerror() << endl;
  exit(-1);
}
void *mkr = dlsym(hndl,"maker");
```



 

指向maker的指针必须是**void\***类型的，因为dlsym返回的就是这个类型。现在，可以通过调用**mkr**创建hexapod类的对象。

 

```
shape *my_shape = static_cast<shape*()>(mkr)();
```



 

调用时要记得将mkr的返回强制转换成shape*类型。

读到这里，可能有读者发现代码有个问题：调用dlsym可能失败了，无法解析“maker”。这个问题根源在于C++为了实现重载，把函数名改了，因而库中的maker函数可能变了。可以通过解析改名的规则来找到改过名的符号，不过有一个更简单的方法。只需要用extern "C"标识来告诉编译器使用C风格的链接包，如下所示：

**列表 2. 类circle的头文件和源文件**

 

```
#ifndef __CIRCLE_H
#define __CIRCLE_H
#include "shape.hh"
class circle : public shape {
public:
  void draw();
};
#endif // __CIRCLE_H
 
 
#include <iostream> #include"circle.hh"
void circle::draw(){
  // simple ascii circle
  cout << "
";
  cout << "     ****
";
  cout << "    *      *
";
  cout << "   *        *
";
  cout << "   *        *
";
  cout << "   *        *
";
  cout << "    *      *
";
  cout << "     ****
";
  cout << "
";
}
extern "C" {
shape *maker(){
  return new circle;
}
class proxy {
public:
  proxy(){
     // register the maker with the factory
     factory["circle"] = maker;
  }
};
// our one instance of the proxy
proxy p;
}
```



 

### 4. 自动注册

可以将maker函数载入到一个专门存放maker函数的数组中。不过，用更灵活的关联数组来保存maker函数在某些情况下或许会更有用。可以用标准模板库(Standard Template Library , [STL](http://www.it165.net/pro/))中的map类，通过键关联到maker函数，就能根据键值来访问。例如，给各个类命名，并通过名字来调用合适的maker函数。本例中，建立如下映射：

 

```
typedef shape *maker_ptr();
map <string, maker_ptr> factory;
```



 

现在，要创建一个指定的形状时，可以通过形状的名称调用合适的maker函数：

 

```
shape *my_shape = factory[
```



 

还可以把这种技术扩展得更灵活。载入类的maker函数时，与其显式地赋值，何不让类的设计者来做这个事？稍微费点心思，让工厂方法自动注册maker函数，这样能让类设计者自由确定名字。（这里有两个忠告：所有的键类型必须相同，绝对不能出现重复的键。）

完成这个工作的第一步是，每个形状库必须包含maker函数，并且每次打开时，都调用这个函数。（通过dlopen的man手册可以知道，如果库定义导出了一个_init函数，那么库每次打开的时候都会调用它。这看起来是一个注册maker函数的好地方，不幸的是目前在linux系统上不能正常工作。问题在于一个标准的链接对象文件crt.o导出了一个__init函数，这就导致了冲突。）至此，我们延续了这个函数名，这个机制仍正常工作。我个人赞成摒弃那种方法，更希望采用一打开库就能注册maker函数的方法。这种方法就是Jim Beveridge引入的著名的“自注册对象”（参考文献）。

可以创建一个代理类单例用于注册maker函数。注册过程会在类的构建函数中进行，因而只需要创建一个代理类实例来处理这件事。这个类的原型如下：

 

```
class proxy {
public:
  proxy(){
     factory["shape name"] = maker;
  }
};
```



 

这里假设工厂实例是主程序里定义导出的全局map实例。使用gcc/egcs，可以设置rdynamic选项来强制主程序导出能够用dlopen函数载入的符号表。

下一步，只定义一个代理实例：

 

```
proxy p;
```



 

现在，打开库时，传参数RTLD_NOW给dlopen函数，p就会被实例化，并注册目标的maker函数。如果想创建一个circle，可以这样调用circle的maker函数：

 

```
shape *my_circle =factory["circle"];
```



 

自动注册处理是非常强大的，利用这个功能设计出的主程序也能支持信息缺乏的类。例如，主程序动态载入了某个形状库之后，通过工厂实例里注册的键，就能够创建一个形状选择菜单。用户从菜单列表中选择"circle"，程序就立刻能调用正确的maker函数。这里，circle类已经支持shape的API，并且maker函数也正确定义了，所以主程序根本不需要其它的信息。

**列表 3. 类Square的头文件和源文件**

 

```
#ifndef __SQUARE_H
#define __SQUARE_H
#include "shape.hh"
class square : public shape {
public:
  void draw();
};
#endif // __SQUARE_H
 
 
#include <iostream>
#include "square.hh"
void square::draw(){
  // simple ascii square
  cout << "
";
  cout << "   *********
";
  cout << "    *       *
";
  cout << "    *       *
";
  cout << "    *       *
";
  cout << "    *       *
";
  cout << "   *********
";
  cout << "
";
}
extern "C" {
shape *maker(){
  return new square;
}
class proxy {
public:
  proxy(){
     // register the maker with the factory
     factory["square"] = maker;
  }
};
// our one instance of the proxy
proxy p;
}
```



 

 

到这里，把列表1到列表5放到一起，相关的概念已经清晰了。列表1的shape类是所有形状类的基类。列表2和列表3分别是circle类和square类的源代码，它们支持库的动态加载。

**列表 4. 主程序代码，调用可以动态载入的类circle和类square**

 

```
#include <iostream>
#include <map>
#include <list>
#include <vector>
#include <string>
#include <dlfcn.h>
#include <stdio.h>
#include <unistd.h>
#include "shape.hh"
// size of buffer for reading indirectory entries
static unsigned int BUF_SIZE = 1024;
// our global factory for making shapes
map<string, maker_t *,less<string> > factory;
int main(int argc, char **argv){
  FILE *dl;   // handle to readdirectory
  char *command_str = "ls *.so"; // command
               // string to get dynamic libnames
  char in_buf[BUF_SIZE]; // input buffer for lib
                          // names
  list<void *> dl_list; // list to hold handles
                               // for dynamiclibs
  list<void *>::iterator itr;
vector<string> shape_names;  // vector of shape
               // types used to build menu
  list<shape *> shape_list; // list of shape
               // objects we create
  list<shape *>::iterator sitr;
map<string, maker_t *,less<string> >::iterator fitr;
  // get the names of all the dynamic libs (.so
              // files) in the current dir
  dl = popen(command_str, "r");
  if(!dl){
     perror("popen");
     exit(-1);
  }
  void *dlib;
  char name[1024];
  while(fgets(in_buf, BUF_SIZE, dl)){
     // trim off the whitespace
     char *ws = strpbrk(in_buf, " 	
");
     if(ws) *ws = '';
     // append ./ to the front of the lib name
     sprintf(name, "./%s", in_buf);
     dlib = dlopen(name, RTLD_NOW);
     if(dlib == NULL){
        cerr << dlerror() << endl;
        exit(-1);
     }
     // add the handle to our list
     dl_list.insert(dl_list.end(), dlib);
  }
  int i = 0;
  // create an array of the shape names
  for(fitr=factory.begin(); fitr!=factory.end();
       fitr++){
     shape_names.insert(shape_names.end(),
       fitr->first);
     i++;
  }
  int choice;
  // create a menu of possible shapes to create and let the user make some
  while(1){
     i = 1;
     for(fitr=factory.begin();
           fitr!=factory.end(); fitr++){
        cout << i << " - Create " << fitr->first
            << endl;
        i++;
     }
     cout << i << " - Draw created shapes
";
     i++; i
     cout << i << " - Exit
";
     cout << "> ";
     cin >> choice;
     if(choice == i){
        // destroy any shapes we created
        for(sitr=shape_list.begin();
              sitr!=shape_list.end();sitr++){
            delete *sitr;
        }
        // close all the dynamic libs we opened
        for(itr=dl_list.begin(); itr!=dl_list.end(); itr++){
           dlclose(*itr);
        }
        exit(1);
     }
     if(choice == i - 1){
        // draw the shapes
        for(sitr=shape_list.begin();
              sitr!=shape_list.end();sitr++){
            (*sitr)->draw();
        }
     }
     if(choice > 0 && choice < i - 1){
        // add the appropriate shape to the shape list
        shape_list.insert(shape_list.end(),
            factory[shape_names[choice-1]]());
     }
  }
}
```



 

 

列表4是可扩展的动态加载库主程序。程序扫描当前目录下的所有so文件（库文件）并打开。这些库会用主程序提供的全局工厂对象注册自身的maker函数。然后，主程序根据库注册过的名字动态地创建形状菜单给用户。通过菜单，用户可以创建各种形状，画形状，或直接退出程序。列表5是用于编译项目的Makefile。

**列表 5.Makefile**

 

```
CC = g++
LIBS = -ldl
.cc .o:
  $(CC) -ggdb -c $<
default:
  make testdcl
OBJS = testdcl.o
testdcl: testdcl.o
  $(CC) -rdynamic -o testdcl testdcl.o $(LIBS)
libcircle.so:  circle.o
  g++ -shared -Wl,-soname,libcircle.so -o libcircle.so circle.o
libsquare.so:  square.o
  g++ -shared -Wl,-soname,libsquare.so -o libsquare.so square.o
all: testdcl libcircle.so libsquare.so
clean:
  rm -f *.so *.o testdcl
```



 

 

### 5. 应用案例

最近，本人有用到了这个技术的两个案例。第一个案例，开发移动物体的模拟器。需要在不能访问主要源代码的情况下，让用户添加移动物体的新类型。要完成这个功能，定义了一个entity基础类，这个类提供了模拟移动物体的所有接口。一个简化版的entity定义如下：

 

```
class entity {
private:
  float xyz[3];  // position of theobject
public:
  activate(float)=0; // tell the object to move
  render()=0;  // tell the object todraw itself
};
```



 

所有的entity都至少有三维坐标，都可以画出自身。大部分entity除了位置之外还有其他的状态变量，同时也不仅仅只有activate函数和render函数，但是这些并不能通过entity接口访问。

可以根据用户期望的动作来定义新的entity类型。运行时，程序载入子目录Entity下的所有库，使它们在模拟的过程能被调用。

第二个案例是最近的项目，创建一个能加载/保存多种图片格式的库。这个库要可扩展的，因此我们新建了一个image_handler基类用于加载和保存图片。

 

```
class image_handler{
public:
  virtual Image loadImage(char *)=0;
  virtual int saveImage(char *, Image &)=0;
};
```



 

image_handler有两个公有函数，分别用于加载和保存图片。Image类是库中图片的基本类型，可以访问图片的数据成员和一些基本的图片操作函数。

在这个案例中，不关心不同类型的image_handler类的多个对象，只要求image_handler类对象可以加载和保存对应类型的图片。因而只需在库中创建这个handler的单例，而不是给每个handler注册一个maker函数，创建的这个handler指针会注册到一个全局映射里。这个全局映射也不是工厂对象，更像是一个普通的图片加载/保存操作器。这里把文件后缀(tiff, jpg等)作为键。一种图片格式可能会有多种文件名后缀（如tiff，TIFF），因此每个handler可能会在全局映射中注册多次，一个后缀一次。

解析出文件后缀后，这个库能够让主程序简单地通过来调用正确的处理函数进行图片的载入和保存。

使用这个库，主程序可以根据文件后缀调用适当的处理函数载入或保存图片。

 

```
map <string, handler,less<string>> handler_map;
char *filename ="flower.tiff";
char ext[MAX_EXT_LEN];
howEverYouWantToParseTheExtensions(filename,ext);
// after parsing"flower.tiff" ext = "tiff"
Image img1 =handler_map[ext]->loadImage(filename);
// process data here
handler_map[ext]->saveImage(filename,img1);
```



 

### 6. 结论

利用类的动态加载技术可以实现更具扩展性更强健的代码。只要设计出考虑周全的类，运用动态加载类的技术，就能够把扩展代码的实用方法提供给用户。