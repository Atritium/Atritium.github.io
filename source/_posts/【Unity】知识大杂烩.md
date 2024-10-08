---
title: 【Unity】知识大杂烩
date: 2024-08-27
tags: 
- Unity 
categories: 
- Unity
---



# 基础

## 1 四元数（Quaternion）

### 1.1 欧拉角的缺陷

#### 1.1.1 概念

- (x,y,z)
- Unity中的欧拉角
  - Inspector窗口中调节的Rotation就是欧拉角（注意，是Inspector窗口中的Rotation，transform中的Rotation是四元数）
  - this.transform.eulerAngles得到的就是欧拉角角度

#### 1.1.2 优点

- 直观、易理解
- 存储空间小（三个数表示）
- 可以从一个方向到另一个方向旋转大于180度的角度

#### 1.1.3 缺点

- 同一旋转的表示不唯一（旋转90度和旋转450度是一个意思，但是表示不同）
- **万向节死锁**
  - 插值时可能会导致抖动、路径错误等。

#### 1.1.4 万向节死锁

[参考](https://blog.csdn.net/qq_43439214/article/details/134109715)

万向节死锁简单来说就是欧拉角的三次旋转过程中产生了一次旋转轴的重合，导致了原来三个旋转自由度实际退化成为两个有效的自由度。

本质上是因为物体的角度状态和欧拉角的坐标并非一一对应关系，某些位置状态并不唯一确定一组欧拉角坐标。

其实从数学上就能马上推算出来：

绕三个轴旋转的旋转矩阵：
$$
Rx(θ) = \begin{bmatrix} 1&0&0 \\ 0&cos(θ)&-sin(θ)\\0&sin(θ)&cos(θ) \end{bmatrix} Ry(θ) = \begin{bmatrix} cos(θ)&0&sin(θ) \\ 0&1&0\\-sin(θ)&0&cos(θ) \end{bmatrix} Rz(θ) = \begin{bmatrix} cos(θ)&-sin(θ)&0 \\ sin(θ)&cos(θ)&0\\0&0&1 \end{bmatrix}
$$
然后，我们计算绕x轴旋转α度，绕y轴旋转90°，绕z轴旋转β度的总变换矩阵：

![image-20240903102729324](【Unity】知识大杂烩/image-20240903102729324.png)

该矩阵可以转换为最后一行，即绕x轴旋转（α-β）度，绕y轴旋转90度的旋转矩阵，z轴自由度丢失。

### 1.2 四元数是什么？

#### 1.2.1 概念

四元数用三个分量 x， y， z 和一个标量 w 来表示旋转：(x,y,z,w)。

#### 1.2.2 Unity中的四元数（Quaternion）

**基本使用**

```c#
// 获取四元数
Quaternion qt = this.transform.rotation;

// 使用四元数做旋转
this.transform.rotation = Quaternion.Euler(0, 50, 0);

//通过Quaternion.AngleAxis()这个API来达到旋转的效果
Quaternion q2 = Quaternion.AngleAxis(60, Vector3.up);
this.transform.rotation = q2;
```

PS：四元数旋转角度相加需要用 * 乘法，而不是加法。

**四元数和欧拉角互转**

```c#
//欧拉角转四元数
Quaternion nQ3 = Quaternion.Euler(60, 0, 0);
//四元数转欧拉角
Vector3 eAngle = nQ2.eulerAngles;
```

**四元数插值**

```c#
A.transform.rotation = Quaternion.Slerp(A.rotation, target.rotation, Time.deltaTime);
```



## 2 Normalize和ClampMagnitude

两者都是Unity里处理向量常用的方法，主要起到规范化向量的功能。

### 2.1 Normalize

Normalize 是一种操作，将向量调整为单位向量，即向量的长度变为 1，同时保持其方向不变。

```
Vector3 direction = new Vector3(3, 4, 0);
direction.Normalize(); // direction becomes (0.6, 0.8, 0)
```

### 2.2 ClampMagnitude

ClampMagnitude是一种操作，用于将一个向量的长度限制在指定的最大值。如果向量的长度超过了这个最大值，它将被缩小到该长度，同时保持方向不变。

ClampMagnitude常用于限制速度、力量或其他物理量，防止它们超过某个阈值。

```c#
Vector3 velocity = new Vector3(10, 15, 0);
velocity = Vector3.ClampMagnitude(velocity, 5); // velocity becomes (3.33, 5, 0)
```

