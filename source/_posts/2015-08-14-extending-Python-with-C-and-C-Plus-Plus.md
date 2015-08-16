title: 用 C/C++ 拓展 Python
date: 2015-08-14 15:12:41
tags: 
- Python
- C
- C++
- 语言拓展
categories:
- Develop
---

![pixiv_id=51833071 然而依然图文无关](/media/2015/08/tokyo_7th_sisters-07.jpg)

Python 因为其是一门解释性语言和动态类型的原因，在计算密集型的操作上非常令人拙计，所以对于计算密集型的操作，我们可以使用 C/C++ 等在计算上比 Python 快的语言做拓展。

本文主要讨论用 C/C++ 对 Python 进行拓展。

<!-- more -->

用 C/C++ 对 Python 拓展的主流方式一共有四种，使用 Python 的 ctypes 模块，使用 Python CAPI，使用 Cython 以及使用 SWIG。

### 直接使用 C/C++ 的动态链接库，使用 Python 的 ctypes 模块

ctypes 是 Python 官方提供的 FFI 模块，适用于提供了提供了动态链接库的 C/C++ 模块。

例如：

```python
import ctypes

cdll = ctypes.CDLL("libxx.so.1.0")
cdll.foo()
```

相对于其他方法，优点是相对于直接去构建面向 C/C++ 的动态链接库，在 C/C++ 这一面不需要做任何的多余的改动。

缺点是调用时的处理复杂度基本转移到了 Python，也没有完整的 Python 配套的一些设施：

* 在调用方面，在 Python 这一方面需要用 ctypes 内置的像 `c_int` 这样的函数去进行类型转换。
* 无法使用 Python 内置的异常，也没有对应 Unicode 等相关的类或者函数。
* 在调试方面，也只能使用 C/C++ 配套的调试工具，比如 `gdb`。

### 使用 Python 的 CAPI 拓展

Python，更具体说 CPython，提供了给 C/C++ 提供了一套 API ，可以使用 Python 配套的一些设施，完成与 C/C++ 与 Python 的交互。

通过增加头文件，使我们可以在 C/C++ 中使用 Python 的基础设施。

```C++
#include <Python.h>
```

#### 一个基础的模块

一个简单的例子是：

```C++
static PyObject *
spam_system(PyObject *self, PyObject *args)
{
    const char *command;
    int sts;

    if (!PyArg_ParseTuple(args, "s", &command))
        return NULL;
    sts = system(command);
    return Py_BuildValue("i", sts);
}
```

所有的函数返回类型都是 PyObject 的指针，参数类型是是 PyObject 的两个指针，前者对应 Python 类里方法的 self，后者对应串里的参数。

对于参数，通过 `PyArg_ParseTuple` 来处理传递的 tuple 形式的参数，通过 `PyArg_ParseTupleAndKeywords` 来处理 keywords 形式的参数；对于返回，通过 `Py_BuildValue` 来格式化返回的参数。

完成我们需要的函数之后，需要显示地声明模块暴露在外的方法，接受参数的类型以及文档：

```c++
static PyMethodDef SpamMethods[] = {
    {"system",  spam_system, METH_VARARGS, "Execute a shell command."},
    {NULL, NULL, 0, NULL}
};
```

初始化编写的模块：

```c++
PyMODINIT_FUNC
initspam(void)
{
    (void) Py_InitModule("spam", SpamMethods);
}
```

使用 `distutils` 编译出 pyd 模块，最后直接 import pyd 模块就可以像 Python 模块使用了。

#### 增加异常处理

Python API 提供 Python 异常的支持，我们可以自己创建异常：

```C++
static PyObject *SpamError;
```

在初始化模块的地方也初始化它：

```C++
PyMODINIT_FUNC
initspam(void)
{
    PyObject *m;

    m = Py_InitModule("spam", SpamMethods);
    if (m == NULL)
        return;

    SpamError = PyErr_NewException("spam.error", NULL, NULL);
    Py_INCREF(SpamError);
    PyModule_AddObject(m, "error", SpamError);
}
```

通过 `PyErr_SetString` ，一个调用可以抛出这个异常。

Python API 提供的异常处理部分除了抛出异常，还提供了一些测试异常相关的模块，`PyErr_Occurred` 来查看某个异常是否被抛出过，`PyErr_Clear()` 用来忽略某个异常等。

#### Python 的垃圾回收机制

Python 拥有自己的一套 GC 机制，通过引用计数来决定是否回收某块内存，对应到 C 拓展里就是对 `PyObject` 的自动 `malloc()` 和 `free()`，对应 C++ 拓展就是自动 `new` 和 `delete`。

当我们对 Python 相关对象进行操作时，需要手动去控制计数器来决定合适回收这个对象。

在 C/C++ 拓展里可以通过 `Py_INCREF(x)` 和 `Py_DECREF(x)` 这两个宏来操纵引用计数，当引用计数归为 0 时，Python 就会回收这一块内存。

Python 作为一门弱类型语言，其变量和变量值是分离的。对于一个对象，没有任何的所属，但是一个对象的指针是可以有所属的，这个指针在需要时，应通过 `Py_INCREF(x)` 增加计数器，当不用时，应通过 `Py_DECREF(x)` 来减少计数器，从而在不用的时候被回收。除了拥有一个对象的指针，还可以向别的函数借指针，借用指针的函数不能比指针拥有者拥有这个指针的时间长，也不需要去管理这个指针的计数器。

可以通过对借用的指针 `Py_INCREF(x)` 来使这个函数成为这个指针的独立拥有者（这个操作会重新拷贝一份指针）。

#### 使用 CAPI 拓展 Python 的优点和缺点

优点是相对于直接使用动态链接库，有了一部分 Python 的基础设施，不再需要在 Python 里显式得转换类型了，也可以使用 Python 的异常等等。

但是在调试方面依然需要采用 C/C++ 的一套工具链，而且因为调用了 Python 底层的 CAPI，有时候需要手动去操控 Python 的引用计数来完成 Python 的 GC 机制，在这一点上依然是比较麻烦的。

### 使用 Cython

Cython 是一种面向 Python 和 Cython 语言的一个优化过的静态编译器。

Cython 会将 Python 和 Cython 翻译成 C/C++，然后编译成一个静态链接库供 Python 使用。

对于一下的纯 Python 代码，如果用 Cython 编译后调用就会提升将近 35% 的速度：

```python
def f(x):
    return x**2-x

def integrate_f(a, b, N):
    s = 0
    dx = (b-a)/N
    for i in range(N):
        s += f(a+i*dx)
    return s * dx
```

而 Cython 语言的语言也基本和 Python 是相仿的，而相对于纯 Python 提供了一些 CAPI 来调用，可以通过这些 CAPI 来获取更快的运行速度。

```python
from libcpp.vector cimport vector

cdef foo(x):
    cdef vector[int] v
    for index in range(x):
        v.push_back(index)
    return v
```

Cython 除了提供 CAPI 之外，还提供了大部分的封装好的模块和函数，像是 C++ 的 STL 等等，而对于 Unicode 或者 Python 的异常机制，Cpython 会帮助你自动翻译成 C/C++，这样来看 Cython 的易用性是相当的好的。

对于运行速度来说，由于是 Cython 做了翻译相关的工作，粒度相对较粗，所以无法做一些粒度更小的优化。

对于编译时检查来说，Cython 会把 Cython 文件翻译后，把语法检查工作交给 C/C++ 编译器来做，在增加了一堆自定义宏、函数和头文件后，一些语法的错误可能是不那么好定位到 Cython 上的，而对于前两种拓展方法，每一行 C/C+++ 都是自己的写的，检查要方便一些。

对于调试来说，Cython 还提供了自己的 `cygdb` 去 debug Cython，相对纯的 `gdb`，要好用一些。


### 使用 SWIG

SWIG 是 Simplified Wrapper and Interface Generator 的简称，SWIG 并不只是提供 C/C++ 对 Python 的拓展，还支持对 Java、Perl 等其他语言的拓展，就是写一个拓展可以在多处使用。

如果不需要使用 Python 的异常或者其他特性，使用 SWIG 来拓展 Python 需要做的只是写一个接口文件。打个比方，对于以下这个函数：

```c++
/* File: myadd.c */
int add(int a, int b) {
    return a + b;
}
```

在编写了接口文件之后：

```cpp
%module myadd
%{
extern int add(int a, int b);
%}

extern int add(int a, int b);
```

经过简单的编译之后：

```bash
% swig -python myadd.i
% gcc -c myadd.c myadd_wrap.c -I /usr/local/include/python2.7
% ld -shared myadd.o myadd_wrap.o -o _myadd.so 
```

然后就可以在 Python 里直接使用了：

```bash
>>> import myadd
>>> myadd.myadd(1 + 1)
2
```

通过在接口文件增加一些 Python CAPI 调用异常的语句并描述异常的 handlers，就可以调用异常等 Python 特性，例如：

```cpp
%init %{
    pMyException = PyErr_NewException("_mylibrary.MyException", NULL, NULL);
    Py_INCREF(pMyException);
    PyModule_AddObject(m, "MyException", pMyException);
%}

%except(python) {
    try {
        $action
    } catch (MyException &e) {
        PyErr_SetString(pMyException, const_cast<char*>(e.what()));
        return NULL;
    }
}
```

随后就可以在 C/C++ 里面直接抛出异常：

```c++
throw MyException("Highly irregular condition...");
```

使用 SWIG 的最大优点是可以使同一套代码拓展多种语言，但是对于某些特性，依然还是要针对 Python 在配置文件中显式地写出一些 Python CAPI 相关的调用，在调试方面，也是只能使用 C/C++ 配套的调试工具，比如 `gdb`。


### 总结

那么对于这几种拓展方式，都分别有谁在用呢？

| 拓展方法 | 使用项目 |
|---------|---------|
| 使用 Python 的 ctypes 模块 | libsvm
| 使用 Python CAPI | numpy, scipy, gevent
| 使用 Cython | scikit-learn, pandas
| 使用 SWIG | PyQT

如果仅仅是像简单地想使用 C/C++ 来提高运行速度，又不想有太大的更改成本，那么推荐用 Cython。

如果是想从底层与 Python 交互，或者是想更低粒度地去优化速度，那么推荐用 Python 的 CAPI 拓展。

如果想写一个拓展给很多门语言，顺便估计 Python，那么推荐 SWIG。

至于直接使用 ctypes 去调用动态链接库，一般是对于不提供 python binding 但是提供 cdll 的库，做自己的一套 wrapper 时用。

------

参考资料：

1. https://jakevdp.github.io/blog/2014/05/09/why-python-is-slow/
2. https://docs.python.org/2/library/ctypes.htm
3. https://docs.python.org/2/extending/extending.html
4. http://cython.org/
5. http://docs.cython.org/src/userguide/debugging.html
6. http://www.swig.org/tutorial.html
7. http://www.zhihu.com/question/23003213/answer/56121859
8. http://stackoverflow.com/questions/1394484/how-do-i-propagate-c-exceptions-to-python-in-a-swig-wrapper-library