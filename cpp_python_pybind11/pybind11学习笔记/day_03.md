## pybind11学习--day03
**内容概述**
- pybind11的函数相关
****
重点知识
- 函数的返回值
- 函数的调用策略
- 函数的参数
****

### 返回值策略
    因为涉及到变量的回收问题，因此对返回值处理的不同策略会导致不同的结果
    常见的返回值策略如下
       策略                                   C++负责               Python负责
    1.automatic(两者均自动判断)                 自动判断                自动判断
    2.reference(python引用一份C++现有的对象)      是                     否
    3.copy(调用C++的拷贝构造给python一份)          是                     是(负责copy的那一份)
    4.move(将返回值移动到python下管理)            是(负责自身的内存)       是(负责移动得到的那一份)
```c++
//代码端效果
 .def("getName", &Pet::getName, py::return_value_policy::reference)  /*在函数定义部分，增加一个策略*/
```
**C++中尽量采用智能指针，可以规避很多不必要的错误**

### 调用策略
[官方参考文档](https://github.com/pybind/pybind11/blob/b6ec0e950c4a12d25cadf492f193ced94f681d6c/tests/test_call_policies.cpp)

### C++中直接调用python的数据类型
```c++
void print_dict(py::dict& dic) {
    for (auto it : dic) {
        std::cout << "key="<< std::string(py::str(it.first)) <<"  "<<"value="<< std::string(py::str(it.second)) <<std::endl;
    }
}
PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring
    
    m.def("print_dict", &print_dict);
}
```
```python
import example

example.print_dict({'a': 1, 'b': 2})
'''
key=a  value=1
key=b  value=2
'''
```
[更多的接口参考](https://pybind11.readthedocs.io/en/stable/advanced/pycpp/index.html)