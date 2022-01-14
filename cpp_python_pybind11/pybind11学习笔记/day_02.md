## pybind11学习--day02
**内容概述**
- pybind11的Object-oriented code
****
重点知识
- struct/class的传递(封装)
- 成员函数&变量的开放
- 动态参数的设计
- 继承&多态
- overload的设计
- 内部类的设计
****

### 1.单一struct/class的接口设计
**常规class的接口设计案例**
```c++
#include <string>
#include <pybind11/pybind11.h>
namespace py = pybind11;
using namespace std;

/*全局函数，其接口设计为m.def()*/
int add(int i, int j) {
    return i + j;
}

/*
struct/class的接口设计为 
    py::class_<classname>(m, "classname")
        .def(普通成员函数)
        .def_static(静态成员函数)
        .def("函数名", lambda表达式)
        .def_readwrite("funcNumber", &Pet::funcNumber)
*/
struct Pet {
    Pet(const string& str) : name(str) {}

    void setName(const string& str) {
        name = str;
    }

    const string& getName()const {
        return name;
    }

    static int getNum() {
        return 1;
    }
    int funcNumber = 3;
private:
        string name;
};


PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    m.def("add", &add, "A function that adds two numbers",
        py::arg("i")= 1, py::arg("j")= 2);
    m.attr("funcNum") = 1;
    py::object name = py::cast("add");
    m.attr("funcName") = name;

    py::class_<Pet>(m, "Pet")
        .def(py::init<const std::string&>())     /*构造函数的传递方式，需要更加注意*/
        .def("setName", &Pet::setName)
        .def("getName", &Pet::getName)
        .def_static("getNum", &Pet::getNum)     /* 静态成员函数的传递方式*/
        .def("__repr__", [](const Pet& a) {
                            return a.getName(); 
            }
            )                                   /*lambda表达式的传递*/
        .def_readwrite("funcNumber", &Pet::funcNumber);  /*开放变量*/
}
```
**动态参数的设计**
在原生的python中，类的设计如下:
```python
class ClaPy:
    name = 'Bob'


cp = ClaPy()
cp.age = 18
print(cp.age)             # 是可以动态增加参数的
```
而在C++中是不支持这种机制的，为了配合这种机制，pybind11设计了一种参数声明
```c++
py::class_<Pet>(m, "Pet", py::dynamic_attr())   /*此处声明py::dynamic_attr()*//
        .def(py::init<const std::string&>())     /*构造函数的传递方式，需要更加注意*/
        .def("setName", &Pet::setName)           /*此处也可以按照key/default arguments方式写*/
        .def_readwrite("funcNumber", &Pet::funcNumber); /*开放变量给外界*/
```
```python
# 调用效果
p = example.Pet('Allen')
p.age = 18
print(p.age)     # 正常输出18
```

### 2.继承&多态
**2.1.继承关系的描述**
```c++
struct Animal {
    Animal(const string& name):mName(name) {}
    Animal() {}
    string mName;
};

struct Dog : Animal {
    Dog(const string& name):Animal(name) {}
    string bark() { return "woof!"; }
};

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    py::class_<Animal>(m, "Animal")
        .def(py::init<const std::string&>())
        .def_readwrite("name", &Animal::mName);

    //method 1: 直接在<>中定义继承关系    推荐方法1
    
    py::class_<Dog, Animal>(m, "Dog")
        .def(py::init<const std::string&>())
        .def("bark", &Dog::bark);

    /*
    py::class_<Animal>animal(m, "Animal");
    animal.def(py::init<const std::string&>())
          .def_readwrite("name", &Animal::mName);

    //method 2: 事先定义具体的父类对象
    
    py::class_<Dog>(m, "Dog",animal)
        .def(py::init<const std::string&>())
        .def("bark", &Dog::bark);
    */
}
```
```python
import example
dog = example.Dog('Ruby')
print(dog.name)
print(dog.bark())
'''
Ruby
woof!
'''
```
**2.2.基类指针指向子类对象的设计完善**

在C++中存在大量的父类指针指向子类对象，调用子类的方法。但是pybind11并不直接支持这种功能，需要进行声明
```c++
//常规C++代码
#include <iostream>
#include <string>
#include <Memory>
using namespace std;

class Animal{	
public:
	Animal(const string& name):m_name(name){}
	virtual ~Animal(){}
	virtual void bark()= 0;
string m_name;

};

class Dog: public Animal{
public:
	Dog(const string& name):Animal(name){}

	void bark(){
		cout<<"woof!"<<endl;
	}
};

int main(){
	std::unique_ptr<Animal> ptr(new Dog("Gigi"));
	ptr->bark();                                  # 正常输出woof
	return 0;
}
```
```text
所谓的声明，即是让基类中定义虚函数，让指针可以自动的下行转换。
```
```c++
struct Animal {
    Animal(const string& name):mName(name) {}
    virtual ~Animal() {}    //定义一个虚函数
    string mName;
};

struct Dog : Animal {
    Dog(const string& name):Animal(name) {}
    string bark() { return "woof!"; }
};

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring
    m.def("get_animal_point", []() {
        return unique_ptr<Animal> (new Dog("GiGi"));
        });
    py::class_<Animal>(m, "Animal")
        .def(py::init<const std::string&>())
        .def_readwrite("name", &Animal::mName);

    py::class_<Dog, Animal>(m, "Dog")
        .def(py::init<const std::string&>())
        .def("bark", &Dog::bark);
}
```
```python
point = example.get_animal_point()
print(type(point))
print(point.bark())
'''
<class 'example.Dog'>
woof!
'''
```

### 3.函数重载之overload
**采用static_cast来进行声明**
```c++
class Person {
public:
    Person(const string& name, int age):m_name(name),m_age(age) {}
    void set(int age) { m_age = age; }
    void set(const string& name) { m_name = name; }
    string m_name;
    int m_age;
};

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    py::class_<Person>(m, "Person")
        .def(py::init<const std::string&, int>())
        .def("set", static_cast<void (Person::*)(int)>(&Person::set), "Set the pet's age")
        .def("set", static_cast<void (Person::*)(const std::string&)>(&Person::set), "Set the pet's name")
        .def_readwrite("name", &Person::m_name)
        .def_readwrite("age", &Person::m_age);
}
```
```python
person = example.Person("Andy", 23)
print(person.name, ":", person.age)
person.set(25)
person.set("Bob")
print(person.name, ":", person.age)
'''
Andy : 23
Bob : 25
'''
```

### 4.内部类的传递
```c++
class Person {
public:
    Person(const string& name, int age):m_name(name),m_age(age){}

    struct Salary
    {
        Salary() {}
        Salary(int month, int money) :salary_month(month), salary_money(money) {}
        int salary_month;
        int salary_money;
    };

    void set(int age) { m_age = age; }
    void set(const string& name) { m_name = name; }
 
    void set_salary(struct Salary sal) {
        m_salary.salary_month = sal.salary_month;
        m_salary.salary_money = sal.salary_money;
    }

    Salary m_salary;
    string m_name;
    int m_age;
};

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    py::class_<Person>person(m, "Person");
    person.def(py::init<const std::string&, int>())
        .def("set", static_cast<void (Person::*)(int)>(&Person::set), "Set the pet's age")
        .def("set", static_cast<void (Person::*)(const std::string&)>(&Person::set), "Set the pet's name")
        .def_readwrite("name", &Person::m_name)
        .def_readwrite("age", &Person::m_age)
        .def("set_salary", &Person::set_salary)
        .def_readwrite("salary", &Person::m_salary);

  py::class_<Person::Salary>(person, "Salary")
      .def(py::init<int, int>())
      .def_readwrite("sarlary_month", &Person::Salary::salary_month)
      .def_readwrite("sarlary_money", &Person::Salary::salary_money);
}
```

```python
import example
s = example.Person.Salary(15, 25)
p = example.Person("Andy", 23)
p.set_salary(s)
month = p.salary.sarlary_month
money = p.salary.sarlary_money
print(money, month)  # 25  15
```
