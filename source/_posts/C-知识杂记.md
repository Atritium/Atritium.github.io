---
title: 【C#】知识杂记
date: 2024-08-20 23:08:27
tags: 
- 计算机基础
- C#
categories: 
- 计算机基础
---



参考：《C#7.0本质论》

# 1 C#概述

## 1.1 .Net和Mono的区别

### 1.1.1 什么是.Net？

.Net是微软的一种技术平台/一种规范，它支持多种语言开发：C#、F#、Visual Basic等。

目前.Net有三种主流实现：

- .Net Framework：主要是基于Windows上开发
- .Net Core：支持跨平台开发
- Mono：支持跨平台开发

### 1.1.2 .Net的编译过程

首先我们写的代码通过特定语言的编译器编译成CIL（Common Intermediate Language：中间语言，字节码，也可以称为IL或CIL），它是一种托管代码，会存储在.DLL或.EXE的程序集中，它是一种伪代码因此不能被计算机直接识别。它与平台操作系统无关与CPU无关是一种中间语言，这也为跨平台奠定了基础。

之后在程序运行时再通过CLR（Common Language Runtime：公共语言运行时）内部的编译器将CIL编译成计算机可以识别的CPU指令(机器码：01010101)，CIL语言是在CLR中运行的，而CLR并不知道CIL是由哪种语言编译而来。

<img src="C-知识杂记/6ffb7cc2dbd2cb69eea47919b4a99589.png" alt="img" style="zoom:50%;" />

### 1.1.3 什么是Mono？

因为.Net Framework本身只能在Windows平台上运行，对于跨平台的需求Mono就产生了。
Mono是基于CLI和C#的ECMA标准提供的.Net的另一种实现，与.Net不同的是它将CLR在所有支持的平台上重新实现了一遍(安卓、Switch，PS4)并且还将.Net Framework提供的基础类库也重新实现了一遍。

Mono的组成：

- C#编译器：C#编译器称为mcs，可以完成C#的编译工作，作用就是将C#源码编译成中间语言CIL，在非Windows平台上需要Mono运行时来运行，而在Windows平台上既可以用.Net运行时，也可以使用Mono运行时。
- Mono运行时(Mono VM)： 实现了ECMA公共语言架构，提供了一个即时编译器（JIT）、预编译器（AOT）、类库加载器、垃圾回收器、线程系统和互操作性功能。
- 基础类库(.Net类库)：提供一组全面的类，这些类兼容.Net并保持一致，是构建程序的结实基础。
- Mono类库：提供了很多超越基础类库的类，提供了额外的功能，例如一些处理Gtk+，Zip文件，LDAP，OpenGL、Cairo、POSIX等等。

## 1.2 GC

[C#垃圾回收机制(GC)](https://cloud.tencent.com/developer/article/2318791)

### 1.2.1 为什么需要GC？

GC是CLR的一个组件，它控制内存的分配和释放，它的出现是为了简化程序员的内存管理工作。

在面向对象的环境中，每个类型都可以代表可供程序使用的一种资源，访问资源的步骤：

 - 调用IL指令newObj，为代表资源的类型分配内存（一般使用new操作符来完成）。
 - 初始化内存，设置资源的初始状态并使资源可使用。类型的实例构造器负责设置初始化状态。
- 访问类型的成员来使用资源。
- 摧毁资源的状态以进行清理。
- 释放内存。

上述的最后一步如果由程序员负责，可能会产生一些无法预测的问题（如：忘记释放不再使用的内存、试图使用已被释放的内存等等），因此GC被引入，单独负责这一步，简化了程序员的内存管理工作。

**new的底层逻辑**

托管堆上有一个nextObjPtr指针，指向下一个对象在堆中分配的位置。

当应用程序执行new操作符后，若内存中有足够的可用空间，就在nextObjPtr处放入对象，接着调用对象的构造方法，并为应用程序返回一个该对象的引用。

nextObjPtr会加上当前对象占用的字节数，获得下一个对象放入托管堆时的地址。

### 1.2.2 GC的工作原理

#### 工作原理

GC即垃圾回收。它是以应用程序的**root**为基础，遍历应用程序在托管堆（Heap）上动态分配的所有对象，通过识别它们是否被引用来确定哪些对象是已经死亡的，哪些是仍需要被使用的。其中，已经不再被引用的对象就是已经死亡的对象，即所谓的垃圾，需要被回收。这就是GC工作的原理。

#### 什么是root？

每个应用程序都包含一组root。每个root都是一个存储位置，其中包含指向引用类型对象的一个指针（可以理解为对象的引用）。该指针要么引用托管堆中的一个对象，要么为null。

在应用程序中，只要某对象变得不可达，也就是没有根（root）引用该对象，这个对象就会成为垃圾回收器的目标。

.NET中可以当作GC Root的对象有如下几种：

- 全局变量
- 静态变量
- 栈上的所有局部变量（JIT）
- 栈上传入的参数变量
- 寄存器中的变量

注意，只有**引用类型的变量**才被认为是root，值类型的变量永远不被认为是root。

#### GC算法：Mark-Compact 标记压缩算法

- 暂停进程中的所有线程。
- GC标记阶段
  - CLR遍历堆中所有对象，将他们的同步索引块中的某一位设为0。
  - **引用跟踪算法**：CLR基于应用程序的root进行检查，查看它们引用了哪些对象，其中空引用（null）的被CLR忽略掉。 任何根如果引用了堆上的对象，CLR都会标记那个对象，将它的同步索引块中的一位设为1 。 那些未被标记为1 的对象即垃圾，被垃圾回收。
- GC压缩阶段
  - 对象回收之后heap内存空间变得不连续，在heap中移动这些对象，使他们重新从heap基地址开始连续排列，类似于磁盘空间的碎片整理。
- 修复引用
  - 压缩过程移动了堆中的对象，使对象地址发生变化，因此需要修复所有引用，即更新它们存储的堆内地址。
  - 上一步有个类似于重定位表的东西，它记录了旧地址到新地址的映射，可以用在这一步。
- 恢复所有线程。

#### GC优化：Generational 分代算法

进行一次完整内存区域的GC（full GC）操作成本很高，因此我们采用分代算法对GC性能进行一定改善。

分代算法的思想：将对象按照生命周期分成新老对象，对新、老区域采用不同的回收策略和算法，加强对新区域的回收处理力度，争取在较短时间间隔、较小的内存区域内，以较低成本将执行路径上大量新近抛弃不再使用的局部对象及时回收掉。

分代算法的假设前提条件：

- 大量新创建的对象生命周期都比较短，而较老的对象生命周期会更长。
- 对部分内存进行回收比基于全部内存的回收操作要快。
- 新创建的对象之间关联程度通常较强。heap分配的对象是连续的，关联度较强有利于提高CPU cache的命中率。

.NET将heap分成3个代龄区域: Gen 0、Gen 1、Gen 2。

![format](C-知识杂记/format.png)

如果Gen 0 heap内存达到阀值，则触发0代GC，0代GC后Gen 0中幸存的对象进入Gen1。如果Gen 1的内存达到阀值，则进行1代GC，1代GC将Gen 0 heap和Gen 1 heap一起进行回收，幸存的对象进入Gen2。2代GC将Gen 0 heap、Gen 1 heap和Gen 2 heap一起回收。

Gen 0和Gen 1比较小，这两个代龄加起来总是保持在16M左右；Gen2的大小由应用程序确定，可能达到几G，因此0代和1代GC的成本非常低，2代GC称为fullGC，通常成本很高。

### 1.2.3 GC的触发时间

 - 最常见的触发条件：CLR在检测第0代内存超过预算时触发一次GC。

- 代码显式调用GC.Collect()。
- Windows报告低内存。
- CLR正在卸载AppDomain。
- CLR正在关闭。

### 1.2.4 如何减少垃圾回收

- 减少new产生对象的次数。
- 使用公用的对象（静态成员）。
- 将string换为stringBuilder（这部分详细看后面string的部分）。

### 1.2.5 手动回收

 - GC.Collect();
 - .Net的GC并不是实时性的，这会造成系统性能上的瓶颈和不确定性。所以有了IDisposable接口，IDisposable接口定义了Dispose方法，这个方法用来供程序员显式调用以释放非托管资源。

### 1.2.6 需要特殊清理的类型

在编写应用程序中肯定会涉及例如：操作文件 FileStream、网络资源socket、互斥锁 Mutex 等这些本机资源。

创建对象时不仅也要为它分配内存资源，还要为它分配本机资源。那么包含本机资源的类型被GC 时，GC会回收对象在托管堆中使用的内存，但这个类型的本机资源不清理的话，就会造成本机资源的泄漏。

所以，CLR提供了称为**终结**的机制，允许对象在被判定为垃圾之后，但在对象内存被回收之前执行一些代码。任何包装了本机资源的类型都支持终结。

CLR 判定一个对象不可达是，对象将终结它自己，释放它包装的本机资源。之后，GC 会从托管堆回收对象。

对于使用了本机资源的对象，在废弃它的时候我们该如何处理呢？

终极基类 System.Object 定义了受保护的虚方法 Finalize。如果你创建的对象使用了本机资源，你可以要重写Object 的虚方法。在类名前添加~ 符号来定义Finalize方法。垃圾回收器判定对象是垃圾后，会调用对象的Finalize 方法。

```c#
internal sealed class SomeType{
    ~SomeType()
    {
        //这里的代码会进入Finalize 方法
    }
}
```

拥有本机资源的对象经历垃圾回收的顺序是这样的：

- 拥有本机资源对象被标记为垃圾，等待GC清理。
- GC 将堆中其他垃圾回收完毕后才调用 Finalize方法，这些使用了本机资源的对象的内存没有被GC马上被回收，因为Finalize 方法可能要执行访问字段的代码。
- 上一步导致拥有本机资源的对象被提升到下一代，使对象活得比正常时间长。
- 当下一代对象被GC 回收时，拥有本机资源的对象的内存才会被回收。如果拥有本机资源的对象的字段引用了其他对象，那么它们也会被提升到下一代。

## 1.3 内存

### 1.3.1 内存分区

- **栈**：由编译器自动分配释放 ，存放值类型的对象本身，引用类型的引用地址（指针）等。其操作方式类似于数据结构中的栈。
- **堆**：用于存放引用类型对象本身。在c#中由.net平台的垃圾回收机制（GC）管理。栈，堆都属于动态存储区，可以实现动态分配。
- **静态区及常量区**
  - 如果一个类型是静态值类型或者常量对象，那么存储在静态区/常量区；如果一个类型是静态引用类型，那么引用存储在静态区/常量区，而对象本身存储在堆上。
  - 由于存在栈内的引用地址都在程序运行开始最先入栈，因此静态区和常量区内的对象的生命周期会持续到程序运行结束时，届时静态区内和常量区内对象才会被释放和回收（编译器自动释放）。所以应限制使用静态类，静态成员（静态变量，静态方法），常量，否则程序负荷高。
- **代码区**：存放函数体内的二进制代码。

### 1.3.2 为什么栈比堆快？
首先，栈是程序运行前就已经分配好的空间，所以运行时分配几乎不需要时间。而堆是运行时动态申请的，分配内存会有耗时。

其次，访问堆需要两次内存访问，第一次取得地址，第二次才是真正得数据，而栈只需访问一次。栈有专门的寄存器，压栈和出栈的指令效率很高，而堆需要由操作系统动态调度。
### 1.3.3 .NET&CLR
C# 程序在 .NET 上运行，而 .NET 是名为公共语言运行时 (CLR) 的虚执行系统和一组类库。CLR 是 Microsoft 对公共语言基础结构 (CLI) 国际标准的实现。 CLI 是创建执行和开发环境的基础，语言和库可以在其中无缝地协同工作。

用 C# 编写的源代码被编译成符合 CLI 规范的中间语言(IL)。 IL 代码和资源（如位图和字符串）存储在扩展名通常为 .dll 的程序集中。

执行 C# 程序时，程序集将加载到 CLR。 CLR 会直接执行实时 (JIT) 编译，将 IL 代码转换成本机指令。 CLR 可提供其他与自动垃圾回收、异常处理和资源管理相关的服务。 CLR 执行的代码有时称为“托管代码”。而“非托管代码”被编译成面向特定平台的本机语言。

### 1.3.4 C#中的内存泄漏
内存泄漏指的是程序中不再需要的内存没有被释放，从而导致内存使用不断增加，最终可能导致系统性能下降或应用程序崩溃。

C#的内存泄露情况有以下几种：
1.委托或事件没有解除注册。
2.静态引用：如果一个静态对象长时间存活且占用大量内存，并且该对象不会被释放或重置，可能导致内存泄漏。
3.长生命对象：如果对象的生命周期很长，而它又引用了大量短命对象，这些短命对象就无法被回收，从而导致内存泄漏。
4.未释放非托管资源：尽管垃圾回收器可以自动管理托管内存，但对非托管理资源（如文件句柄、数据库连接等）仍然需要手动释放。如果未正确释放这些资源，会导致内存泄漏（可以用接口IDispose进行释放）。

### 1.3.5 弱引用（Weak Reference）
弱引用（Weak Reference）是一种特殊的引用类型，它允许你引用一个对象而不阻止该对象被垃圾回收器（GC）回收。换句话说，弱引用不会延长对象的生命周期。

```csharp
var strongReference = new object(); // 创建一个强引用对象
var weakReference = new WeakReference<object>(strongReference); // 创建一个对该对象的弱引用
strongReference = null; // 删除强引用
```

# 2 数据类型

## 2.1 类型的划分

C#的类型有两种：值类型（value type）和引用类型（reference type）。值类型的变量存储数据，而引用类型的变量存储对数据的引用。

### 2.1.1 值类型

值类型直接包含值，换言之，变量引用的位置就是值在内存中实际存储的位置。将第一个变量的值赋给第二个变量，会在新变量的位置创建原始变量的值的一个内存副本。类似的，将值类型的实例传给方法，如Console.WriteLine()，也会生成一个内存副本，在方法内部对参数值进行任何修改都不会影响原始值。

对于值类型，每个变量都有它们自己的数据副本（ref和out参数除外）。
![在这里插入图片描述](C-知识杂记/53022300bd71438da7def67964f17d3d.png)

### 2.1.2 引用类型

相反，引用类型的变量存储的是对数据存储位置的引用，而不是直接存储数据。因此，为了访问数据，“运行时”要先从变量中读取内存位置，再“跳转”到包含数据的内存位置。
![在这里插入图片描述](C-知识杂记/ae2b8456cfab4757a30a32c017e382f3.png)
引用类型复制引用而不需要复制所引用的数据，表现为在栈内开辟一块新的空间，存储复制数据在堆中的地址，因此当有一个引用类型变量复制了另一个引用类型变量，该变量的改变也会引起另一个变量的改变。

### 2.1.3 值类型和引用类型的区别

 - 值类型数据存储在栈上，而引用类型数据存储在堆上。
 - 值类型的复制是按值传递的，引用类型的复制是按引用传递的，当我们对复制体进行修改时，值类型的原数据不会受到影响，而引用类型数据会随之改变。
 - 值类型存取快，引用类型存取慢。因为首先值类型存储在栈上，栈上的内存是事先分配好的。其次，访问堆需要两次内存访问，第一次取得地址，第二次才是真正得数据，而栈只需访问一次。栈有专门的寄存器，压栈和出栈的指令效率很高，而堆需要由操作系统动态调度。
 - 值类型继承自System.ValueType，引用类型继承自System.Object。
 - 值类型的内存管理由编译器自动完成，而引用类型的内存管理由垃圾回收器完成。
 - 值类型的生命周期由程序的作用域控制，而引用类型的生命周期由垃圾回收器控制。

### 2.1.4 装箱和拆箱

- 装箱
  - 值类型转换为引用类型。
  - 从托管堆中为新生成的引用对象分配内存，然后将值类型的数据拷贝到刚分配的内存中，并返回该内存的地址。这个地址存储在栈上作为对象的引用。
- 拆箱
  - 引用类型转换为值类型。
  - 首先获取托管堆中属于值类型那部分字段的地址，将引用对象中的值拷贝到位于栈上的值类型实例中。拆箱只是获取引用对象中指向值类型部分的指针，而内容拷贝是赋值语句触发。

装箱会产生较大性能损失（主要是构造新对象），拆箱的性能损失较小（主要是强制类型转换）。

```csharp
using System;
class Test{
	static void Main(){
		int i = 123;
		object o = i;     //Boxing
		int j = (int)o;   //Unboxing
	}
}
```

### 2.1.5 try&finally实验

**情况一**

```csharp
void Start(){
	int i = GetInt();
	Debug.Log("第A处 i="+i);
}
int GetInt(){
	int i=10;
	try{
		return i;
	}
	finally{
		i=11;
		Debug.Log("第B处 i="+i);
	}
}
```

结果：

```csharp
B=11
A=10
```

执行顺序：先执行try，然后缓存i，由于i是值类型，所以实际上缓存的是一个i的数据副本，值为10，然后再执行finally，最后return。

**情况二**

```csharp
class Test{
	public int i=10;
}
class CSharpLearing:MonoBehaviour{
	void Start(){
		Test t=GetObj();
		Debug.Log("第A处 i="t.i);
	}
	Test GetObj(){
		Test t = new Test();
		try{
			return t.i;
		}
		finally{
			t.i = 11;
			Debug.Log("第B处 i="+t.i);
		}
	}
}
```

结果：

```csharp
B=11
A=11
```

虽然return依旧对要返回的数据进行缓存，但由于t是引用类型，因此后面执行的finally中对数据的操控依旧影响到了t。

### 2.1.6 用ref和out来传递值类型

ref和out可以将值类型以引用的方式进行传递，从而改变原来变量中的值。

它们俩的区别在于：通过ref传入的参数必须在传入方法前对其进行初始化操作，而通过out传入的参数不需要在传入方法前对其初始化，即便初始化了，也会在传入方法时清空，然后再在方法内赋初值。

例如当我们使用Swap我们可以使用ref：

```csharp
class Program{
	static void Main(string[] args){
		Program pg = new Program();
		int x = 10;
		int y = 233;
		pg.Swap(ref x,ref y);
		Console.WriteLine(x+" "+y);
	}
	static void Swap(ref int x,ref int y){
		int temp = x;
		x = y;
		y = x;
	}
}
```

## 2.2 数组（Array）
### 一维数组

```csharp
int[] arrayA = new int[n];
int[] arrayB = new int[]{1,2,3};
```
### 二维数组
```csharp
//两行三列的二维数组
int[,] arrayA = new int[2,3];
int[,] arrayB = new int[,]{
	{1,2,3},
	{3,2,1},
};
```
### 交错数组
交错数组可以理解为**数组的数组**。
```csharp
//由于交错数组存放的数组长度可能各不相同，所以不指定第二维度
int[][] arrayA = new int[2][];
int[][] arrayB = new int[][]{
	new int[]{1,2,3},
	new int[]{3,2,1},
};
```
## 2.3 元组（Tuple）

元组提供了一种简洁的方式来组合多个值，尤其是在你需要返回多个值或将多个值作为一个整体传递时。例如，你可以使用元组来返回多个结果而无需定义一个新的类或结构体。

在C#中，你可以使用System.ValueTuple命名空间下的ValueTuple结构来创建元组。

PS：ValueTuple比起Tuple更推荐使用。

元组创建：

```csharp
// 创建一个元组，包含两个元素
var person = (Name: "Alice", Age: 30);
Console.WriteLine(person.Name); // 输出: Alice
Console.WriteLine(person.Age);  // 输出: 30
```

or

```csharp
(int,float,bool,string) tmp = (1,5.5f,true,"123");
Console.WriteLine(yz.Item1);
```

or

```csharp
(int i,float f) yz = (1,5.5f);
Console.WriteLine(yz.i);
```

元组解构：

```csharp
var (name, age) = person;
Console.WriteLine(name); // 输出: Alice
Console.WriteLine(age);  // 输出: 30
```

返回多个值：

```csharp
public (string FullName, int Age) GetPersonInfo()
{
    return ("Bob", 25);
}
```

## 2.4 字符串（string）

### 2.4.1 string的不变性和驻留性
string是一种比较特殊的引用类型。

**不变性**
字符串一经创建，值不可变。对字符串的各种修改操作都会创建新的字符串对象。

当你给一个字符串重新赋值，会在堆中重新开辟一块空间存储新值，并将栈内存储的地址修改为新开辟空间的地址，而老的值会继续存在于堆中，等到垃圾回收时再被销毁。

> [C#里其他不可变类型](https://zhuanlan.zhihu.com/p/655900143)

**驻留性**
运行时将字符串值存储在“驻留池（字符串池）”中，相同值的字符串都复用同一地址。

一般只有两种情况下字符串会被驻留：
1.字面量的字符串，这在编译阶段就能确定的“字符串常量值”。相同值的字符串只会分配一次，后面的就会复用同一引用。例如 string str = "abc" 这种。
2.过 string.Intern(string) 方法主动添加驻留池。
![在这里插入图片描述](C-知识杂记/991d18dd207d4da0b7248daf8f54e345.png)
驻留的字符串（字符串池)在托管堆上存储，大家共享，内部其实是一个哈希表，存储被驻留的字符串和其内存地址。驻留池生命周期同进程，并不受GC管理。

### 2.4.2 字符串的比较（==和Equals()）
对于值类型，这俩都一样，都是比较两个值类型的值是否相等，所以在这里我们只探讨引用类型。
**原本的**
C#的“==”用来比较引用类型的地址是否相等，即对比的两个变量是否指向同一个内存地址。而Equal()则比较的是引用类型的内容是否相等，即堆内存里存放的值是否相等。
**string的**
而string的“==”进行了重载，作用和Equals()一致。
### 2.4.3 字符串的连接

> [C#字符串连接](https://www.cnblogs.com/popzhou/p/3676691.html)
> [C# string](https://www.cnblogs.com/anding/p/18221262)

**+=**

string str = tmpStr + "abc";会被优化程string.Concat(...)。通过分析string.Concat(params string[] values)的实现可以知道：先计算目标字符串的长度，然后申请相应的空间，最后逐一复制，时间复杂度为o(n)，常数为1。

固定数量的字符串连接效率最高的是+。

但是字符串的连+不要拆成多条语句，比如：

```csharp
string sql = "update tableName set int1=";
sql += int1.ToString();
sql += ...
```

这样的代码，不会被优化为string.Concat，就变成了性能杀手，因为第i个字符串需要复制n-i次，时间复杂度就成了o(n^2)。

**StringBuilder**

StringBuilder 只分配一次内存，如果第二次连接内存不足，则修改内存大小；它每次默认分配16字节，如果内存不足，则扩展到32字节，如果仍然不足，继续成倍扩展。

如果频繁的扩展内存，效率大打折扣，因为分配内存，时间开销相对比较大。如果事先能准确估计程序执行过程中所需要的内存，从而一次分配足内存，效率大大提高。

如果字符串的数量不固定，就用StringBuilder，一般情况下它使用2n的空间来保证o(n)的整体时间复杂度，常数项接近于2。

因为这个算法的实用与高效，.net类库里面有很多动态集合都采用这种牺牲空间换取时间的方式，一般来说效果还是不错的。

**string.Format**

它的底层是StringBuilder，所以其效率与StringBuiler相似。

### 2.4.4 空字符串

```csharp
string str = null;
string str = "";
string str = string.Empty;
```

**三者的区别**

str = null 在堆中没有分配内存地址。

str = "" 和 string.Empty 一样都是在堆内存中分配了空间，里面存储的是空字符串。

string.Empty是一个静态只读变量。

# 3 基础

## 3.1 类

### 3.1.1 嵌套类

假如一个类在它的包容类外部没有太多意义，就适合设计为嵌套类。

```c#
public class Program{
	private class Program2{
		//...
	}
}
```

### 3.1.2 分部类（partial）

分部类就是将类的定义划分到多个文件（同个程序集）里，编译器可以把它们合成一个完整的类。

```c#
//File1
partial class Program{}
//File2
partial class Program{}
```

### 3.1.3 is和as

**is**

is用来验证基础类型。

```c#
public void Method(object data){
	if(data is string){
		//...
	}
}
```

```c#
public void Method(object data){
	//这里检查data是否为string，如果为true则赋值给text
	if(data is string text&&text.Length>0){
		//...
	}
}
```

**as**

as比is更进一步，它会尝试转换数据类型。

```c#
Console.WriteLine(data as string);
```

## 3.2 结构体（struct）

### 3.2.1 定义

结构体可以简单地视为值类型的类。

```csharp
struct Point{
	public int x,y;
	public Point(int x,int y){
		this.x = x;
		this.y = y;
	}
}
```

### 3.2.2 结构体VS类

**区别**

- 结构体中声明的字段无法赋予初值，类可以。
- 结构体的构造函数中，必须为结构体所有字段赋值，类的构造函数则没有这个限制。
- 类为引用类型，可继承；结构体为值类型，不可继承（只能继承接口）。

**选择**

类是引用类型，因此类的对象存储在堆中。而结构体是值类型，结构体的对象存储在栈中。堆空间大，但访问速度慢。栈空间小，但访问速度快。因此，当我们描述的对象是一个轻量级对象时，选用结构体可提高效率。但如果我们希望传值的时候传递的是对象的引用而非对象的拷贝，那么我们可以选用类。

## 3.3 字段和属性

```csharp
public class Person{
	private string name;
	public string Name{
		get{return name;}
		set{name = value;}
	}
}
public class Program{
	Person preson = new Person();
	person.Name = "manqi";
	Console.WriteLine(person.Name);
}
```
属性是一种特殊的方法，用于控制对成员变量的访问和赋值。通过使用属性，可以对成员变量进行保护，使得外部代码无法对其直接访问和修改。

优点：
- 安全性：将读、写权限分开：get和set是分开实现的，保证了数据安全。
- 灵活性：给属性赋值或取值时，可以根据自己的需求实现各种不同的赋值或取值逻辑。

## 3.4 泛型（Generic）

### 3.4.1 为什么使用泛型？

使用泛型是一种增强程序功能的技术，具体表现在以下几个方面：

 - 它有助于您最大限度地重用代码、保护类型的安全以及提高性能。
 - 您可以创建泛型集合类。.NET 框架类库在 System.Collections.Generic 命名空间中包含了一些新的泛型集合类。您可以使用这些泛型集合类来替代 System.Collections 中的集合类。
 - 您可以创建自己的泛型接口、泛型类、泛型方法、泛型事件和泛型委托。
 - 您可以对泛型类进行约束以访问特定数据类型的方法。
 - 关于泛型数据类型中使用的类型的信息可在运行时通过使用反射获取。

最常用的泛型例子：

```csharp
List<T>
```

编译时由编译器会对泛型进行类型替换。

### 3.4.2 泛型约束

泛型允许你定义具有通用类型参数的类、接口、方法等，使得代码能够适应多种数据类型。泛型的约束（Generic Constraints）是对这些通用类型参数所能接受的类型的限制。

C#里有这么几种泛型约束：

**值类型约束**

```csharp
public void Method<T>(T obj) where T:struct{}
```

**引用类型约束**

```csharp
public void Method<T>(T obj) where T:class{}
```

**无参构造约束**
限制类型参数必须有一个无参数的构造函数。

```csharp
public void Method<T>(T obj) where T:new(){
	public T CreateInstance()
    {
        return new T(); // 由于约束，确保 T 有一个无参数构造函数
    }
}
```

**类约束**
限制类型参数必须继承自某个特定的基类。

```csharp
public void Method<T>(T obj) where T:BaseClass{}
```

**接口约束**
限制类型参数必须实现某个接口。

```csharp
public void Method<T>(T obj) where T:ISomeInterface{}
```

**另一个泛型约束**
限制类型参数T必须是另一个类型参数U的子类型。

```csharp
public void Method<T, U>(T obj) where T:U{}
```

### 3.4.3 协变和逆变（out&in）

[协变和逆变](https://blog.csdn.net/weixin_41793160/article/details/135179750)

子类->父类：协变

父类->子类：逆变

协变和逆变能够实现数组类型、委托类型和泛型类型参数的隐式引用转换。

## 3.5 委托&Lambda&事件

[知乎：事件是以特殊方式声明的委托字段吗](https://zhuanlan.zhihu.com/p/452117637)

### 3.5.1 委托（delagate）的用途

为了方便理解，我们先从委托的用途讲起。

在C/C++中，“函数指针”将对方法的引用作为实参传给另一个方法，而委托在C#中承担着相似的功能。虽然这样说，但我们对委托用途的理解可能还是很抽象，这里用一个简单的例子帮助理解。

冒泡排序是最基础的排序算法，它的代码大致如下：

```csharp
public static void BubbleSort(int[] items) {
    if (items == null) return;
    for(var i = items.Length - 1; i >= 0; i--)
    {
        for(var j = 0; j+1 <= i; j++)
        {
            if (items[j] > items[j + 1])
            {
                var temp = items[j];
                items[j] = items[j + 1];
                items[j + 1] = temp;
            }
        }
    }
}
```
该方法对整数数组执行升序排序。

但为了能够选择升序和降序，我们开始拓展这段代码。
第一个方案：拷贝以上代码，然后把大于操作符换成小于操作符。
第二个方案：增加一个参数，指出我们当前希望如何排序，然后在代码里进行判断。

但以上代码只照顾到了两种可能的排序方式，如果还想要按其他方式进行排序，代码就会变得很庞大。

为了增加灵活性，减少重复代码，我们可以将比较方法作为参数传入。而此时我们需要有一个数据类型可以表示方法，这就是委托。

委托加入后，代码会变成这样：

```csharp
public static void BubbleSort(int[] items,Func<int,int,bool> compare) {
    if (items == null) return;
    if (compare == null) return;
    for(var i = items.Length - 1; i >= 0; i--)
    {
        for(var j = 0; j+1 <= i; j++)
        {
            if (compare(items[j], items[j+1]))
            {
                var temp = items[j];
                items[j] = items[j + 1];
                items[j + 1] = temp;
            }
        }
    }
}
```
显然灵活多了。

PS：写到这里我突然理解了Sort((x,y)=>x>y)的含义，之前对升序到底对应x>y还是y>x总是一知半解。x>y那么x和y就交换，所以就是升序排列（虽然Sort底层是优化过的快排，不是冒泡排序，但也有两个数比对的步骤，所以代入一下就能得出结论）。

### 3.5.2 委托的声明
#### 自定义委托

声明委托需要使用关键词delegate，并且指出委托的返回值和所需参数，只有方法的返回值和参数与委托一致，才可以将方法传递给委托。

```csharp
//先定义委托
delegate void Feedback();
//然后使用new操作符构造委托实例并传入方法
class Program{
	static void Main(string[] args){
		//向委托的构造函数传递静态方法
        Feedback fbStatic = new Feedback(Program.FeedbackToConsole);
        //传递实例方法
        Feedback fbInstance = new Feedback(new Program().FeedbackToFile);
        
        //调用委托的两种不同方式
        fbStatic.Invoke();
        fbInstance();
	}
}
```

#### System.Func和System.Action

为了减少定义自己的委托类型的必要，C#推出了一组常规用途的委托。

**Func**
代表有返回值的方法。

```csharp
delegate TResult Func<参数,参数...,out TResult>
```
 **Action**
 代表无返回值的方法。


```csharp
delegate void Action<参数,参数...>
```

PS：虽然C#开始提供Func和Action委托，减去了自定义委托类型的必要（要写声明还要自定义名字），但有时候出于可读性，还是可以考虑声明自己的委托类型。如Comparer委托就能使人对其用途一目了然。

```csharp
public delegate bool Comparer(int first,int second);
class Program{
	public static void BubbleSort(int[] items,Comparer comparer){}
}
```

### 3.5.3 委托是一种特殊的类
委托实际上是特殊的类，它派生自System.MulticastDelegate，而后者又派生自System.Delegate，其实把它理解成一种C#里的类型就可以了，但它和一般的类型又有些不同。如果说int，string等是对数据类型的定义，那么委托就类似于对“方法类型”的定义（看着下面的代码仔细琢磨）。

```csharp
string str;
delegate void Method(int num);  //理解：Method是变量名，delegate/返回值/参数共同构成了一种“方法类型”
```

### 3.5.4 Lambda表达式
上文提到的冒泡排序，如果我们想去调用它，代码大致如下：

```csharp
public static void BubbleSort(int[] items,Func<int,int,bool> comparer){}

//声明方法
public static void GreaterThan(int first,int second){
	return first>second;
}

//Main函数里调用
static void Main(string[] arg){
	int[] items = new int[5];
	//...初始化
	BubbleSort(items,GreaterThan);
}
```
你会发现整个过程下来要进行的准备有点太过复杂了，其实我们只需要主体的return first>second。于是C#提供了匿名函数（C# 3.0叫Lambda表达式），简化了这个过程。

```csharp
BubbleSort(items,(first,second)=>{return first>second});
```
### 3.5.5 事件（event）
事件就是对委托的封装，相对于委托，它只提供了“+=”和“-=”两个方法，保证了在外部操作时的安全性。

**对订阅的封装**
只提供“+=”和“-=”，避免程序员在编写代码时错误地使用“=”代替“+=”。

**对发布的封装**
不再提供Invoke方法，保证只有指定字段发生变更时，委托方法才会被调用，避免外部主动调用。

```csharp
private delegate void Method();
public event Method EventName;   //提供给外部
```

### 3.5.6 闭包

闭包允许函数访问外部作用域中的变量，用Lambda或者匿名函数实现，可以捕获不属于其作用域的值。

# 4 高级
## 4.1 面向对象三大特征
### 4.1.1 封装
封装指的是隐藏类的实现细节，对外只暴露必要的接口和方法。它将数据和操作封装在一个类中，控制数据的访问权限。例如：事件就体现了封装的概念。

在C#中，封装的体现：

 - 访问修饰符
    - public：对任何类和成员都公开，无访问限制。
	- internal：只能在包含该类的程序集中访问该类。
	- protected：对该类和该类的派生类公开。
	- protected internal：protected+internal。
	- private：仅对该类公开，如果是修饰类的话，那么该类仅对同个文件公开。
- 字段和**属性**
- 事件
- ...

### 4.1.2 继承
继承允许我们创建一个新的类，去继承已有的类的属性和方法，并添加或修改自己的属性和方法。通过继承，可以实现代码的复用，提高程序的可维护性和可扩展性。
### 4.1.3 多态
#### 重载（overload）

重载是编译时多态，指的是在同一个类中定义多个同名的方法，但参数列表不同，这样可以根据不同的参数列表调用不同的方法。

#### 重写（override）

重写是运行时多态，指的是在继承关系中，子类可以重写基类中的虚方法和抽象方法。

## 4.2 抽象
### 4.2.1 虚函数VS抽象函数
- 虚函数用virtual修饰，在基类中定义；抽象函数用abstract修饰，在抽象类中定义。
- 虚函数有实现体，抽象函数没有实现体。
- **虚函数子类可以选择性重写，而子类必须实现抽象方法，否则子类也必须声明为抽象类。**
- 虚函数可以通过基类对象的引用或子类对象的引用调用，而抽象函数只能通过子类对象的引用调用。

```csharp
public abstract class Animal{
    public abstract void Eat();
    public virtual void Run(){
        Console.WriteLine("Animal is running.");
    }
}

public class Dog : Animal{
    public override void Eat(){
        Console.WriteLine("Dog is eating.");
    }
    public override void Run(){
        Console.WriteLine("Dog is running.");
    }
}

public static void Main(string[] args){
    Animal animal1 = new Dog(); // 通过基类对象的引用调用虚方法
    animal1.Run(); // 输出：Dog is running.

    Dog dog1 = new Dog(); // 通过子类对象的引用调用虚方法和抽象方法
    dog1.Run(); // 输出：Dog is running.
    dog1.Eat(); // 输出：Dog is eating.
}
```
### 4.2.2 抽象类VS接口
 - 抽象类不能被实例化，而接口也不能被实例化，只有具体类才能被实例化。
- 抽象类可以包含抽象方法和具体方法，也就是说抽象类中可以有部分已经实现的方法，但接口不行，接口只能包含抽象方法，没有具体的实现。
- 抽象类可以被具体类继承，并且可以作为父类来提供一些通用的行为和属性，而接口可以被具体类实现，一个类可以实现多个接口，从而实现多重继承。
- 抽象类通常用于定义类的层次结构，并提供一些通用的方法和属性，而接口则通常用于定义一些规范或者协议，来保证实现该接口的类拥有一定的行为和属性。

### 4.2.3 为什么构造函数不能是虚函数

构造函数和虚函数在设计概念和实现上是存在冲突的。

构造函数的主要职责是初始化对象的状态，在一个类的对象被创建时，构造函数负责分配内存、初始化成员变量。构造函数在创建对象的过程中是按继承层次从基类到派生类依次调用的（这里其实很好理解，因为父类都还没初始化好，何来子类呢）。

而虚函数则不同。

从设计层面，虚函数是一种支持多态性的方法，在多态情况下只执行其中的一个，这点和构造函数不同。

从存储空间角度，虚函数对应一个虚函数表（vtable），可是这个vtable其实是存储在对象的内存空间的。问题出来了，如果构造函数是虚的就需要通过vtable来调用，可是对象还没有实例化，也就是内存空间还没有，无法找到vtable，所以构造函数不能为虚函数。

## 4.3 反射（Reflection）

[C# 反射（Reflection）超详细解析](https://blog.csdn.net/weixin_45136016/article/details/139095147)

C# 中的反射（Reflection）是一种在运行时动态获取类型信息并操作类型实例的技术。反射允许程序在运行时检查程序集、模块、类型、成员（如方法、属性、字段）等信息，并且可以动态调用方法、访问属性和字段等。反射是 .NET 框架的一部分，主要位于 `System.Reflection` 命名空间中。

反射的核心在于它能够**在运行时读取元数据**（metadata）。在 .NET 中，所有类型、方法、属性等信息都会在编译时嵌入到程序集的元数据中。反射利用这些元数据来实现对类型及其成员的动态操作。

```csharp
public void Method(Object obj){
	//在运行时获取类型信息
	Type t = obj.GetType();
}
```
## 4.4 多线程（Task/async/await）

### 4.4.1 概念

[Task/async/await的使用](https://www.cnblogs.com/funiyi816/p/16557254.html)

### 4.4.2 多线程与异步

异步操作指的是在程序执行过程中，可以继续进行其他任务而不需要等待当前操作的完成。

从定义来说，**多线程是异步模型的一种实现形式**，在多线程模型中，可以将耗时的操作放在一个单独的线程中执行，而不会阻塞主线程，这种思路很明显是符合异步模型的。因此，异步操作可以基于多线程实现，而多线程可以用于实现并发执行的异步操作。

有一个著名的说法：“异步不一定是多线程，也可以是单线程”，比如Unity协程的实现方式（协程可以主动申请暂停自身，并提供一个唤醒条件，Unity会每帧检查这个唤醒条件是否已经满足，满足了则继续执行。协程是在主线程上实现的，因此是单线程）。

