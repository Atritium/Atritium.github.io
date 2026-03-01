---
title: 记录部分Unity性能优化知识
date: 2026-02
tags: 
- Unity
- 性能优化
categories: 
- Unity
description: 记录平时学到的部分性能优化知识
---

# 1 GC优化

`GC (Garbage Collector) `负责自动管理内存。当堆上的某个对象从任何`GC Roots`（包括栈、静态变量等）出发都不可达时，该对象会被标记为垃圾。`GC`会在特定时机（如内存分配预算达到阈值）执行回收，并视情况重新整理（压缩）堆内存以释放空间。

然`GC`是一个极其消耗性能的工作，每次都需要遍历整个堆内存，因此我们要尽量避免`GC`。

**如何避免频繁触发GC**

**——减少频繁分配内存**

```c#
void Update(){
	List<int> list = new List<int>();
	Fun(list);
}
```

像上面这种写法，每帧都在使用`new`进行内存分配，我们完全可以改成下面这种写法：

```c#
private List<int> list = new List<int>();
void Update(){
	list.Clear();
	Fun(list);
}
```

这种写法只有在容器被创建或扩容时才会有堆分配，从而减少了垃圾的产生。

**——运用对象池**

在运行时大量对象的创建和销毁依然会引起`GC`问题，用对象池技术可以让对象复用而不是重复的创建和销毁。

**——字符串**

在`C#`中，`String`是引用类型，它的值是不可变的，一旦被初始化后就不能改变其内容，频繁的修改字符串建议使用`StringBuilder`。

**——装箱**

值类型转换为引用类型的过程称为装箱，装箱会产生`GC`。

**——协程**

避免`yield return 0`，因为会产生`GC`，因为`int`类型的0被装箱，而使用`yield return null`替代则不会产生装箱操作，还比如在协程中避免多次`new`同一个`WaitForSeconds`对象。

```c#
while(!isComplete){
	yield return new WaitForSeconds(1f);
}

// 替代为↓

WaitForSeconds delay = new WaitForSeconds(1f);
while(!isComplete){
    yield return delay;
}
```

**——Linq表达式**

`LINQ`和正则表达式由于在后台会有装箱操作而产生垃圾，在有性能要求的时候最好不使用。

# 2 Draw Call优化

在Unity里，Draw Call指的是CPU发出的绘制请求，其中包含了绘制所需的所有信息，如纹理信息、着色器等。

**为什么我们要优化Draw Call？**

在现代硬件中，GPU的处理能力通常很强，假设一个场景有2000个Draw Call，CPU可能需要花20ms才能把这些指令发完，而GPU画完它们只需要5ms。也就是说，Draw Call太多的后果是GPU大部分时间都在等CPU发指令，这时游戏帧率就会卡在CPU提交这一步。

**Draw Call的优化手段：批处理技术。**

[批处理](https://blog.csdn.net/qq_45745755/article/details/154197319)
