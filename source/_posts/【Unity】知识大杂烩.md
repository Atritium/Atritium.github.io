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
Rx(θ) = \begin{bmatrix} 1&0&0 \\ 0&cosθ&sinθ \\ 0&-sinθ&cosθ \end{bmatrix}

Ry(θ) = \begin{bmatrix} cosθ&0&-sinθ \\ 0&1&0 \\sinθ&0&cosθ \end{bmatrix}

Rz(θ) = \begin{bmatrix} cosθ&sinθ&0 \\ -sinθ&cosθ&0\\0&0&1 \end{bmatrix}
$$
当y轴旋转90°，y轴的旋转矩阵则变为：
$$
Ry(θ) = \begin{bmatrix} 0&0&-1 \\ 0&1&0 \\1&0&0 \end{bmatrix}
$$
那么总的变换矩阵可以计算：
$$
Rx(θ)Ry(90°)Rz(θ) = \begin{bmatrix} 0&0&-1 \\ sin(θx-θz)&cos(θx-θz)&0 \\cos(θx-θz)&-sin(θx-θz)&0 \end{bmatrix}
$$
从最终的变换矩阵可以看出来，结果只受x和z两个维度控制。

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

