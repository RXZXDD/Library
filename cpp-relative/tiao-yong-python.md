# 调用Python

## 简述

c++ 调用 python ，本质上是在 c++ 中启动了一个 python 解释器，由解释器对 python 相关的代码进行执行，执行完毕后释放资源，达到调用目的。

从操作步骤上看，C++调用Python低层接口可以分为几个阶段：

* 初始化Python解释器
* 从C++到Python转换数据
* 用转换后的数据做参数调用Python函数
* 把函数返回值转换为C++数据结构

即写一个C文件，执行【Python解释器初始化、导入模块，导入函数，构造输入参数，调用函数，解析返回值，终止Python解释器】。



## 官方文档

{% embed url="https://docs.python.org/3/extending/index.html" %}

## 前置操作

* include目录\
  $PYTHON\_PATH\include
* 库目录\
  \
  $PYTHON\_PATH\libs
* DLL目录\
  \
  $PYTHON\_PATH\DLLs
* Python库目录\
  \
  $PYTHON\_PATH\Lib\


项目运行需要依赖 $PYTHON\_PATH 下的python312.dll,因此需要把相关的dll，DLL目录，python库文件夹复制到生成的可执行文件目录下。

## 基础操作API

1. 初始化和结束解释器环境

```cpp
void Py_Initialize()：       
    初始化python解释器.C/C++中调用Python之前必须先初始化解释器
int Py_IsInitialized()：
    返回python解析器的是否已经初始化完成，如果已完成，返回大于0，否则返回0
void Py_Finalize() ：
    撤销Py_Initialize()和随后使用Python/C API函数进行的所有初始化，
    并销毁自上次调用Py_Initialize()以来创建并为被销毁的所有子解释器。
```

2. 运行python脚本，把python代码当作一个字符串传给解释器来执行

```cpp
int PyRun_SimpleString(const char*) :
    执行一个简单的执行python脚本命令的函数

int PyRun_SimpleFile(FILE *fp, const char *filename)：
    从fp中把python脚本的内容读取到内容中并执行，filename应该为fp对应的文件名
```

3. 动态加载python

```cpp
//加载模块
PyObject* PyImport_ImportModule(char *name)

PyObject* PyImport_Import(PyObject *name)
PyObject* PyString_FromString(const char*)
    上面两个api都是用来动态加载python模块的。区别在于前者一个使用的是C的字符串，而后者的name是一个python对象，
这个python对象需要通过PyString_FromString(const char*)来生成，其值为要导入的模块名


//导入函数相关
PyObject* PyModule_GetDict( PyObject *module)
    PyModule_GetDict()函数可以获得Python模块中的函数列表。PyModule_GetDict()函数返回一个字典。字典中的关键字为函数名，值为函数的调用地址。
字典里面的值可以通过PyDict_GetItemString()函数来获取，其中p是PyModule_GetDict()的字典，而key则是对应的函数名

PyObject* PyObject_GetAttrString(PyObject *o, char *attr_name)
     PyObject_GetAttrString()返回模块对象中的attr_name属性或函数，相当于Python中表达式语句：o.attr_name

//调用函数相关
PyObject* PyObject_CallObject( PyObject *callable_object, PyObject *args)
PyObject* PyObject_CallFunction( PyObject *callable_object, char *format, ...)
    使用上面两个函数可以在C程序中调用Python中的函数。callable_object为要调用的函数对象，也就是通过上述导入函数得到的函数对象，
而区别在于前者使用python的tuple来传参，后者则使用类似c语言printf的风格进行传参。
如果不需要参数，那么args可能为NULL。返回成功时调用的结果，或失败时返回NULL。
这相当于Python表达式 apply(callable_object, args) 或 callable_object(*args)
```

4. 参数构造

```cpp
PyObject* Py_BuildValue( const char *format, ...)
    Py_BuildValue()提供了类似c语言printf的参数构造方法，format是要构造的参数的类型列表，函数中剩余的参数即要转换的C语言中的整型、浮点型或者字符串等。
其返回值为PyObject型的指针。
```

5. 获取函数返回值

```
int PyArg_Parse( PyObject *args, char *format, ...)
     根据format把args的值转换成c类型的值，format接受的类型和上述Py_BuildValue()的是一样的
```

6. 内存对象管理\
   Python使用引用计数机制对内存进行管理，实现自动垃圾回收。在C/C++中使用Python对象时，应正确地处理引用计数，否则容易导致内存泄漏。在Python/C API中提供了Py\_CLEAR()、Py\_DECREF()等宏来对引用计数进行操作。 每个PyObject对象都有一个引用计数，用于垃圾回收

```cpp
void Py_CLEAR(PyObject *o)
void Py_DECREF(PyObject *o)
其中，o的含义是要进行操作的对象。
对于Py_CLEAR()其参数可以为NULL指针，此时，Py_CLEAR()不进行任何操作。而对于Py_DECREF()其参数不能为NULL指针，否则将导致错误。
```
