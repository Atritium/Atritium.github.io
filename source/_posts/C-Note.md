---
title: C#程序员不得不品的一些C++笔记
date: 2026-01-08 20:38:14
tags: 
- C++
categories: 
- 计算机基础
---

# 0 前言

C++属于我一直想学但总是“望而生畏”的那种类型，毕竟业界传闻C++是最难的编程语言……加上平时工作上也用不着C++（平时工作主Unity开发，一般都用C#），所以就更给了自己理由拖延学习计划。

但离职之后我开始思考之后的发展方向，也浏览了网上一些游戏开发岗的JD，感觉从长远发展来说，会C++一定是成为一个优秀游戏程序员的必要条件之一（你家Unity底层都是用C++写的你还有什么可说的），于是痛下决心开始学习。

这篇文章记录一些学习C++过程中学到的新知识（学习过程中我也发现其实从一个语言的设计目的出发就很容易理解一个东西为什么要这么设计）。

# 1 数据类型

 C++ 的某些类型大小依赖实现/平台，而C#（.NET 规范）的数据类型大小是固定不变的（跨平台一致），这里列出两个语言数据类型的关键区别。

## 1.1 long/long long

- C++：long大小依赖于平台（通常4字节，可能8字节），long long固定8字节
  - **Windows (LLP64 模型)**：long是4字节，即使在64-bit系统上
  - **Unix/Linux/Mac (LP64 模型)**：long是8字节

- C#：long=8 字节，没有 long long

现代推荐：用固定大小类型，保证跨平台一致。

```c++
#include <cstdint>
std::int32_t i32;    // 固定 32-bit（代替 int/long 在 Windows）
std::int64_t i64;    // 固定 64-bit（代替 long long，或 Unix 的 long）
```

## 1.2 char

- C++ char永远1字节，支持ASCII（英文+基本符号），如果是一个中文的话，占3~4char
  - C++还有宽字符（wchar_t，大小2字节或4字节，平台差异）

- C# char始终2字节，专为存储一个Unicode字符设计，中文只占1char

## 1.3 C#：decimal

C# 独有，高精度小数（金融/货币计算必备），C++ 无

## 1.4 C++：指针

指针是C++的核心特性，可以通过它间接访问内存，所有类型的指针在32bit下是四字节，64bit下是8字节。

### 1.4.1 基础使用

```c++
int a = 10;
int* p;  
p = &a;          //$是取址符号
cout<<*p<<endl;  //再加一个*是解引用
```

### 1.4.2 空指针/野指针

空指针

```c++
int* p = NULL;   //指针变量p指向内存地址编号为0的空间
cout<<*p<<endl;  //内存编号0~255为系统占用内存，不允许用户访问
```

野指针：指针变量指向非法的内存空间

```C++
//指向内存编号为0x1100的空间
int* p = (int*)0x1100;
cout<<*p<<endl;  //访问野指针报错
```

### 1.4.3 const三种修饰指针的方式

常量指针

- `const int* p`
- 不允许修改指针指向的值

指针常量

- `int* const p`
- 不允许修改指针的指向

const既修饰指针又修饰常量

- `const int* const p`
- 既不能修改指针指向的值，又不能修改指针的指向

# 2 数组

C++ 和 C# 的数组在设计哲学上差异很大：C++ 数组更底层、灵活、高性能；C# 数组更安全、现代、托管。

## 2.1 原生数组

C++

- `int arr[10];`

- C++原生数组可以存放在栈上，大小在编译时确定-栈分配需要编译器在生成函数栈帧时就知道确切大小

- C++原生数组**不允许赋值操作**，如`arr2 = arr1;`

  - C++原生数组名在表达式中几乎总是退化为指向首元素的指针，而赋值运算符需要左值对象完整语义
  - 编译器生成代码时，数组是连续内存块，如果允许赋值，编译器需要生成隐式逐元素拷贝代码，如果数组很大，开销不可控，而C++追求“零开销对象”

- C++原生数组的经典特性：**数组退化**

  - 定义：在大多数表达式上下文中，数组名会自动退化为指向数组首元素的指针，从而丢失大小信息（这样做是为了效率：传整个数组内容开销大，传指针，64bit下只需要8字节）

  - 函数传参中的退化（最经典）：

  - ```c++
    // 此时arr是指向数组首元素的指针
    void func(int arr[]) {
    	//sizeof(arr)，返回指针大小（64bit-8字节）
    } 
    ```

C#

- `int[] arr = new int[10];`
- C#原生数组始终是引用类型，在托管堆上分配，大小在运行时确定 
- C#原生数组允许赋值操作，本质上是引用拷贝（浅拷贝），`arr2`和`arr1`指向同一个数组对象（内容共享）

## 2.2 C++：array

C++11引入的模板类，是C++原生数组的现代包装。其设计目的是保留原生数组的高性能和栈分配优势，同时提供容器般的安全和便利性。

```c++
std::array<int,5> arr = {5,4,3,2,1};
```

**解决数组退化**

array在函数传参时可以按值/引用传递，不会退化为指针，丢失大小信息（array类自带数量参数）。

如何按引用传递：

```C++
//值传递（拷贝一份数据）
void func(std::array<int, 10> arr) {}

//引用传递（不用拷贝一份数据，性能较好）
void func_mod(std::array<int, 10>& arr) {}
```

**允许赋值操作**

支持`arr2=arr1;`本质上是深拷贝。

## 2.3 动态数组

严格来说，这部分不算数组，因为数组的大小是固定，不可变的。

### 2.3.1 C++：vector

#### 1 push_back()和emplace_back()

两者都是std::vector用于在容器末尾添加元素的成员函数，但它们在构造方式和效率上有很多区别，emplace_back() 是 C++11 引入的，更现代、更高效的版本。

**push_back**

- 接受一个已经构造好的对象，通过拷贝或移动整个对象到vector末尾

**emplace_back**

- 接受构造函数的参数，在vecor末尾就地构造一个新T对象，直接调用T的构造函数，避免了临时对象的拷贝/移动


#### 2 size()和capacity()

size()表示的是当前数组内元素的数量，而capacity()表示的是数组的容量，因此capacity()总是>=size()。

为什么 capacity >= size()，且通常 > size()？

**核心原因：性能优化**

- vector 内部用连续内存块存储元素（像动态数组）
- 添加元素时，如果 capacity 不足，会：
  - 分配新更大内存块
  - 拷贝/移动旧元素到新块
  - 释放旧块

- 解决方案：扩容时**故意多分配**（capacity > 新 size），留余量
  - 下几次添加只增加 size（O(1)），无需扩容

#### 3 下标操作符[]和at()

均用于元素访问，但[]没有边界检查，越界时会触发未定义行为，而at()有边界检查（C#里[]自带边界检查）。

#### 4 vector的两种创建方式

**栈上声明**

```
std::vector<int> vec;
vec.push_back(10);
```

内存位置

- vector对象本身（控制块：_data 指针、size、capacity...）：栈上
- 元素数据（实际int数组）：堆上

释放：自动

- 函数结束，vec析构-自动delete[]内部数据

**用new创建（不推荐）**

```C++
std::vector<int>* vec = new std::vector<int>();  //用new返回的是指针
vec->push_back(10);

delete vec;   //必须手动delete！
```

内存位置

- vector对象本身：堆上（由new分配）
- 元素数据：堆上
- vec指针变量：如果为局部变量，栈上

### 2.3.2 C#：List

略

# 3 内存分区

C++程序在执行时，将内存大方向划分为**4个区域**

- 代码区
  - 存储可执行机器代码（编译后的函数指令）
  - 只读：防止修改
  - 共享：对于频繁被执行的程序，只需要在内存中有一份代码即可
  - 大小固定，程序加载时分配
- 常量区/数据段
  - 常量区存放常量，数据段存放全局/静态变量
  - 常量区只读，数据段可读写
  - 大小固定
- 栈区
  - 由编译器自动分配释放, 存放函数的参数值，局部变量等
  - 从高地址向低地址增长（快速）
  - 大小固定，通常几MB，容易栈溢出
- 堆区
  - 用于动态分配内存，从低地址往高地址增长（与栈相反）
  - 大小不固定，运行时动态扩展
  - C++中可使用new/malloc分配内存

# 4 引用

- 引用的作用是给变量起个别名，由于设计目的如此，因此必须在声明时进行初始化

  - ```c++
    int a = 10;	
    int &b = a;
    ```

- 引用本质上是一个指针常量，因此不允许重新赋值（修改指针常量指向）

- 引用做函数参数，可以简化使用指针修改实参

  - ```c++
    //1.值传递
    void mySwap01(int a, int b) {
    	int temp = a;
    	a = b;
    	b = temp;
    }
    
    //2.地址传递
    void mySwap02(int* a, int* b) {
    	int temp = *a;
    	*a = *b;
    	*b = temp;
    }
    
    //3.引用传递
    void mySwap03(int& a, int& b) {
    	int temp = a;
    	a = b;
    	b = temp;
    }
    
    //通过引用参数产生的效果同按地址传递是一样的。引用的语法更清楚简单
    ```

# 5 类和对象

## 5.1 对象的初始化

可以用初始化列表的方式初始化

```c++
class Person {
public:
	Person(int a, int b, int c) :m_A(a), m_B(b), m_C(c) {}
private:
	int m_A;
	int m_B;
	int m_C;
};
```

## 5.2 友元

设计目的是允许部分外部函数或类访问类内的私有属性，关键字为friend。

```c++
class Building
{
	friend void method();
    //...
};
```

# 6 模板

函数模板作用：建立一个通用函数，其函数返回值类型和形参类型可以不具体制定，用一个**虚拟的类型**来代表。

## 6.1 函数模板

### 6.1.1 语法

```c++
template<typename T>
void mySwap(T& a, T& b)
{
	T temp = a;
	a = b;
	b = temp;
}
```

### 6.1.2 普通函数和函数模板的调用规则

规则如下

- 如果函数模板和普通函数都可以实现，优先调用普通函数
- 可以通过空模板参数列表来强制调用函数模板

```c++
//普通函数与函数模板调用规则
void myPrint(int a, int b)
{
	cout << "调用的普通函数" << endl;
}

template<typename T>
void myPrint(T a, T b) 
{ 
	cout << "调用的模板" << endl;
}

void test01()
{
	//1、如果函数模板和普通函数都可以实现，优先调用普通函数
	int a = 10;
	int b = 20;
	myPrint(a, b); 

	//2、可以通过空模板参数列表来强制调用函数模板
	myPrint<>(a, b); 
}
```

### 6.1.3 模板的特化

例如像下面代码，如果传入的a和b是一个数组，就无法实现了。

```c++
template<class T>
void f(T a, T b)
{ 
    a = b;
}
```

C++为了解决这种问题，提供模板的特化，可以为这些特定的类型提供具体化的模板。

**注意**：函数模板只支持**全特化**（template<>），不支持偏特化（partial specialization）。

C++

```c++
#include<iostream>
using namespace std;

#include <string>

class Person
{
public:
	Person(string name, int age)
	{
		this->m_Name = name;
		this->m_Age = age;
	}
	string m_Name;
	int m_Age;
};

//普通函数模板
template<class T>
bool myCompare(T& a, T& b)
{
	if (a == b)
	{
		return true;
	}
	else
	{
		return false;
	}
}


//具体化函数模板
template<> bool myCompare(Person &p1, Person &p2)
{
	if ( p1.m_Name  == p2.m_Name && p1.m_Age == p2.m_Age)
	{
		return true;
	}
	else
	{
		return false;
	}
}
```

## 6.2 类模板

很多通用容器如`std::vector<T>`都是基于类模板实现的。

### 6.2.1 语法

```c++
template <typename T> 
class MyClass {
public:
    T data;  

    MyClass(T value) : data(value) {} 

    void print() {
        std::cout << data << std::endl;
    }
};

void method(){
	MyClass<int> intObj(42);      
	MyClass<double> doubleObj(3.14);
	MyClass<std::string> strObj("hello");
}
```

**默认模板参数：int**

```c++
MyClass<> obj;  //等价 MyClass<int>
```

### 6.2.2 模板的特化

支持为某个类型特化模板（类模板支持全特化和偏特化）。

```c++
#include <iostream>

//通用模板：处理普通类型
template <typename T>
class Test {
    //...
};
```

**全特化**

完全指定所有模板参数。

```c++
//为 bool 全特化
template <>
class Test<bool> {
    //...
};
```

**偏特化**

```c++
// 偏特化：匹配 T*（任何类型的指针）
template <typename T>
class Test<T*> {
    //...
};
```

