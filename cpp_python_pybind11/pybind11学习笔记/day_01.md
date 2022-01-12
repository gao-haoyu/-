## pybind11学习--day01
**内容概述**
- pybind11的基础用法
****
重点知识
- pybind11的环境配置
- 基础测试用例
- keyword argument
- default argument
- exporting variables
****

### 1.pybind11的环境配置
#### step1：安装pybind11的安装包
```text
虚拟环境的创建
    1.检查工具链 pip list   查看是否有virtualenv
    2.若没有，直接pip install virtualenv进行安装
    3.拥有之后，virtualenv 虚拟环境名
    4.cd 进入虚拟环境文件夹，运行Scripts下的activate文件，进入虚拟环境
    5.虚拟环境下，pip install pybind11
虚拟环境的好处：
    1.将不同的项目之间进行分离，每个环境下都有其单独的运行依赖
```
```text
vs中的路径依赖
    1.将解决方案平台设置为x64
    2.检查项目的属性，配置属性——>常规——>配置类型，此处设置为.dll动态库
    3.配置属性——>高级——>目标扩展名，此处设置为.pyd
    4.配置属性——>VC++目录——>包含目录，此处增加pybind11的头文件和python的头文件
      (pybind11的头文件为pybind11的安装目录下pybind11/include将该路径包含即可)
      (python的头文件为python安装目录下的python3.9/include)
    5.配置属性——>VC++目录——>库目录，此处增加python的libs目录
      (此处的目录，是python的库，为python安装目录下的python3.9/libs)
    6.配置属性——>链接器——>输入——>附加依赖项，增加两个环境
      (这两个均是python安装目录下libs下的环境python3.lib   python39.lib)
```

### 2.基础测试
在完成上面的配置之后，在项目中新建代码文件
```c++
#include <pybind11/pybind11.h>
namespace py = pybind11;

int add(int i, int j) {
    return i + j;
}

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    m.def("add", &add, "A function that adds two numbers");
}
```
```text
后续操作：
  1.分析上面的代码
    PYBIND11_MODULE(example, m) {   
    /*此处是生成一个example的库来供python调用
      因此，项目生成的.pyd文件名称也要为example，便于后面的import example*/
    m.doc() = "pybind11 example plugin"; // optional module docstring
    /*文件声明，需要有*/
    m.def("add", &add, "A function that adds two numbers");
    /*库中定义了一个函数'add'，其调用地址在&add,也就是C++定义的add函数*/
  
  2.具体的应用
    在example.pyd所在的目录下进入shell环境，运行下列代码调用
    PS G:\github\pybind\vsproject\Project1\x64\Debug> python
    Python 3.9.0 (tags/v3.9.0:9cf6752, Oct  5 2020, 15:34:40) [MSC v.1927 64 bit (AMD64)] on win32
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import example
    >>> example.add(1,2)
    3                     # 可以看到正常调用example库且函数成功运行
}
```

### 3.keyword argument
**增加keyword argument**
```c++
//new code
#include <pybind11/pybind11.h>
namespace py = pybind11;

int add(int i, int j) {
    return i + j;
}

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    m.def("add", &add, "A function that adds two numbers",
        py::arg("i"), py::arg("j"));
        /*此处增加 py::arg("keyword name")来设定关键字参数*/
    
    /*keyword arguments another notation  更简便
      using namespace pybind11::literals;
      m.def("add2", &add, "i"_a, "j"_a); 
    */
}
```
```shell
# 执行指令如下
>>> import example
>>> example.add(i=1, j=2)
3
```

### 4.default argument
**增加default argument**
```c++
//new code
/*在c++中函数的形参部分直接定义默认值并不会正常传递到python中去，需要额外定义*/
PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    m.def("add", &add, "A function that adds two numbers",
        py::arg("i")= 1, py::arg("j")= 2);
}
```
```shell
# 采用这种方式之后，原本的add函数中不写默认值也可以，python中依然按照有默认值调用
>>> import example
>>> example.add()
3
```

### 5.exporting variables
**变量的传递**
```c++
/*采用attr关键字来设计传递的变量*/
PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    m.def("add", &add, "A function that adds two numbers",
        py::arg("i")= 1, py::arg("j")= 2);
    
    m.attr("funcNum") = 1;
    
    py::object name = py::cast("add");    /*做一次格式转换*/
    m.attr("funcName") = name;
}
```
```shell
>>> import example
>>> example.funcNum
1
>>> example.funcName
'add'
>>> example.add()
3
```