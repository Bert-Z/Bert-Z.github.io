---
title: Python模块搜索路径
date: 2020-07-19 14:45:27
tags: Python
categories:
cover: https://nakedsecurity.sophos.com/wp-content/uploads/sites/2/2020/01/pylogo-1200.png?w=780&h=408&crop=1
---
<meta name="referrer" content="no-referrer" />

最近在学习python的C++扩展（pybind11)，写完一个扩展模块之后，想要在自己的环境中以后都能自动导入这个模块，而不用手动去添加路径(修改sys.path)应该怎么弄？以前最开始学习Python的时候看过这块内容，然而时间长了总会记忆不清，就再回顾了一遍。
概括来说，Python的自动搜索路径是这样的：

- 程序的根目录
- PYTHONPATH环境变量设置的目录
- 标准库的目录
- 任何能够找到的.pth文件的内容
- 第三方扩展的site-package目录

最终，这五个部分的拼接成为了sys.path，其中第一和第三、第五部分是自动定义的。

### 根目录(自动)

Python首先在根目录搜索要导入的文件。这个根目录的入口依赖于你怎么运行代码；当你运行一个程序时，这个入口就是程序运行入口(top-level script file)文件所在的目录；当你用交互式窗口期运行代码时，这个入口就是你所在的工作目录。

### PYTHONPATH 目录(可配置的)

接下来，python会搜索PYTHONPATH环境变量里列出的所有目录，因为这个搜索在标准库之前，所以要小心不要覆盖一些标准库的同名模块。

### 标准库目录(自动)

这个没什么好说的，pyton会自动搜寻标准库模块所在的目录。

### .pth文件列出的目录(可配置的)

这是用的比较少的一个python特性。它允许用户以每行一个的方式列出搜索路径，它和PYTHONPATH环境变量的不同是会在标准库路径之后搜索；而且它是针对这个python安装的，而不是针对用户的（环境变量会随着用户的不同而不同）。 那么，.pth文件应该放在哪里呢？可以通过以下代码找到.pth文件可以放置的位置：

```python
import site
site.getsitepackages()

```

在我的环境中，输出如下：

```shell
['C:\\Python27', 'C:\\Python27\\lib\\site-packages']
```

### Lib/site-package目录(自动)

最后，python会在搜索路径上自动加上site-packages目录，这一般是第三方扩展安装的地方，一般是由distutils工具发布的。

