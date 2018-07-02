---
title: C++中类的封装与重复定义的思考
date: 2018-06-15 09:13:58
tags: C++
categories: coding
---

由于我自身对C/C++的语法不熟练，在C++类的封装时，遇到过"multiple-declaration"或者"multiple definition"的问题。这种情况一般出现在对某一个类的头文件.h和实现文件.cpp封装的时候在头文件中写法不规范导致的。仔细研究了一下，并且查阅了网上一些资料，所以在这里做一个简单的总结。

## C基础数据类型的重复定义
这种问题一般有两种：同一编译单元内的重复定义和不同编译单元内的重复定义。一个.cc/.cpp/一般就是一个编译单元，会生成对应的.o文件。

### 同一编译单元内的重复定义
```c++
//student.h
int age = 23;
int student_no = 1203102
int sex = 0;
...

//school.h
#include "student.h"
int class_no;
int search_student(int stu_no);
...

//school.cpp
#include "school.h"
#include "student.h"
...
```
上述情况，在school.cpp中引用school.h和student.h，编译的时候就会报错:"multiple definition"，原因就在于，school.h引用了student.h，school.cpp中又引用了一遍student.h，相当于对student.h引用了两次，于是对于age等变量相当于在同一个编译单元内重复定义了两遍，编译就报错了。解决方法是在每一个.h文件加宏来确保在同一个编译单元内只被引用一次。比如：
```c++
//student.h

#ifndef STUDENT_H
#define STUDENT_H
int age = 23;
int student_no = 1203102;
int sex = 0;
#endif
...

```

### 不同编译单元内的重复定义
当然，上述情况并没有从根本上解决问题，比如看下面代码：

```c++
//student.h
int age = 23;
int student_no = 1203102;
int sex = 0;
int get_age();
...

//student.cpp
#include "student.h"
int get_age(){
    return age;
}
...

//test.cpp
#include "student.h"
int stu_age = get_age();

```
比如有这样三个文件，student.h是学生类的头文件，student.cpp是学生类的实现文件，test.cpp是引用这个类的测试文件。这种用法是很常规的用法。但是编译的时候会报错重复定义错误。原因就在于student.h中的变量声明的同时做了定义，include到两个不同的cpp文件，即两个编译单元后，由于编译会最终把student.o和test.o编译到一起，所以还是会报重复定义的错误。

因此，想要解决C中基础数据类型的重复定义错误，就**必须把声明和定义分开，即在.h文件中对于基础数据类型的变量只做声明不做定义**。修改后如下：

```c++
//student.h
int age;
int student_no;
int sex;
int get_age();
...

//student.cpp
#include "student.h"



int get_age(){
    age = 23;
    return age;
}
...

//test.cpp
#include "student.h"
int stu_age = get_age();

```
这样做编译器不会报错。但是这里也有一个问题，C中没有类的概念，全局变量的作用范围是单个文件，如果想在多个文件中共享一个变量的值，可以用`extern`关键字。需要明确的一点是，C的语法要求是，可以在不同的编译单元对相同的变量分别做声明，但是定义必须是唯一的，这也就是为什么一个.h文件中的声明在对应的实现文件.cpp中做了定义，其他引用这个.h文件的cpp是可以拿来直接用的原因。

```c++
//student.h
extern int age;
int student_no;
int sex;
int get_age();
...

//student.cpp
#include "student.h"

int age = 23;
...

//test.cpp
#include "student.h"
printf("age: %d\n", age);

```
这样，test.cpp中打印出的age的值就是在student.cpp中赋值的23。

## C++中类的.h写法

也许习惯了C的这种语法过后，在.h文件中声明一个类，对类的成员函数做定义的时候你可能会有这样一个疑问：“在.h中定义成员函数，被多个cpp include的时候难道不会重复定义吗？”

一般我们的写法都是借鉴C中的规范，在.h中对类只做声明，对类的定义放到相应的.cpp文件中去。这样做无疑是最好的也是最规范的做法，但是在.h中对类做定义，是不会像基础数据类型变量那样发生重复定义报错的。

我们知道，类的声明和普通变量声明一样都是不产生目标代码的。但是**类的定义也不产生目标代码。因此它和普通变量的声明唯一的区别是不能在同一编译单元内出现多次**。比如普通变量在同一编译单元内的不同作用域中可以重复声明定义，但是类不行。这是因为类的定义，只是告诉编译器，类的数据格式是如何的，实例话后对象该占多大空间。**写在类定义里面的函数是inline的。**


```c++
  //source1.cc

   class A;

   class A;  //类重复声明，OK

   class A{

   };

   class A{

   };       

   class A{

       int x;

  };    //同一编译单元内，类重复定义，会编译时报错,因为编译器不知道在该编译单元，A a；的话要生产怎样的a.

         //如果class A{};定义在head.h ，而head.h 没有 

         //#ifndef  #endif 就很可能在同一编译单元出现类重复定义的编译错误情况。
```

**但是在不同编译单元内，类可以重复定义,因为类的定义未产生实际代码。**

```c++
  //source1.cc

  class A{

  }

  //source2.cc

  class A{

  }                      //不同编译单元，类重复定义，OK。所以类的定义可以写在头文件中！
```
### 关于类成员函数的inline

> 在类体中定义的成员函数的规模一般都很小，而系统调用函数的过程所花费的时间开销相对是比较大的。调用一个函数的时间开销远远大于小规模函数体中全部语句的执行时间。为了减少时间开销，如果在类体中定义的成员函数中不包括循环等控制结构，C++系统会自动将它们作为内置(inline)函数来处理。

也就是说，在程序调用这些成员函数时，并不是真正地执行函数的调用过程(如保留返回地址等处理)，而是把函数代码嵌入程序的调用点。这样可以大大减少调用成员函数的时间开销。C++要求对一般的内置函数要用关键字inline声明，但对类内定义的成员函数，可以省略inline，因为这些成员函数已被隐含地指定为内置函数。
```c++
class Student
{
public :
    void display( )
    {
    cout<<"num:"<<num<<endl;cout<<"name:"
    <<name<<endl;cout<<"sex:"<<sex<<endl;
    }
private :
    int num;
    string name;
    char sex;
};
```

其中第3行`void display( )`也可以写成`inline void display( )`将display函数显式地声明为内置函数。

以上两种写法是等效的。对在类体内定义的函数，一般都省写inline。

应该注意的是，**如果成员函数不在类体内定义，而在类体外定义，系统并不把它默认为内置(inline )函数，调用这些成员函数的过程和调用一般函数的过程是相同的。如果想将这些成员函数指定为内置函数，应当用inline作显式声明**。如：

```c++
class Student
{
public : inline void display( );//声明此成员函数为内置函数
private :
    int num;
    string name;
    char sex;
};
inline void Student::display( ) // 在类外定义display函数为内置函数
{
    cout<<"num:"<<num<<endl;cout<<"name:"<<name<<endl;cout<<"sex:"<<sex<<endl;
}
```
在前面曾提到过，在函数的声明或函数的定义两者之一作inline声明即可。值得注意的是，如果在类体外定义inline函数，则必须将类定义和成员函数的定义都放在同一个头文件中(或者写在同一个源文件中)，否则编译时无法进行置换(将函数代码的拷贝嵌入到函数调用点)。但是这样做，不利于类的接口与类的实现分离，不利于信息隐蔽。虽然程序的执行效率提高了，但从软件工程质量的角度来看，这样做并不是好的办法。只有在类外定义的成员函数规模很小而调用频率较高时，才将此成员函数指定为内置函数。

## 总结

1. C中的基础数据类型变量在.h中只能声明不能定义。（全局变量作用域为整个文件）
2. C中多文件共享全局变量要在.h中声明为extern类型，并且只能在一处编译单元做定义。
3. C++中.h文件可以同时做类的声明和定义
4. C++中非inline函数（如static 成员函数）不可以在头文件中定义，必须到类的外部去定义。

## 参考资料

> [C++ 关于声明，定义，类的定义，头文件作用，防止头文件在同一个编译单元重复引用，不具名空间——游园惊梦](http://www.cnblogs.com/rocketfan/archive/2009/10/02/1577361.html)
> [C++类的成员函数(在类外定义成员函数、inline成员函数)](http://www.cnblogs.com/wuchanming/p/4061654.html)