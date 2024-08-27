---
title: 【Unity】知识大杂烩
date: 2024-08-27 17:24:36
tags: 
- Unity 
categories: 
- Unity
---



# 基础

## 1 四元数（Quaternion）

### 1.1 为什么使用四元数：欧拉角的缺陷

**概念**

- (x,y,z)
- Unity中的欧拉角
  - Inspector窗口中调节的Rotation就是欧拉角（注意，是Inspector窗口中的Rotation，transform中的Rotation是四元数）
  - this.transform.eulerAngles得到的就是欧拉角角度

**优点**

- 直观、易理解
- 存储空间小（三个数表示）
- 可以从一个方向到另一个方向旋转大于180度的角度

**缺点**

- 同一旋转的表示不唯一（旋转90度和旋转450度是一个意思，但是表示不同）
- **万向死锁**
  - [现象分析](https://www.bilibili.com/video/BV1Nr4y1j7kn/?spm_id_from=333.337.search-card.all.click&vd_source=d1cd7437519192c36dc659c247e8160e)
  - 当位于中间调整顺序的轴（比如x->y->z的y）旋转为90°，其他两个轴就位于重合位置，就会发生万向死锁，所以我们可以把最不可能为90°的轴放在中间，最大限度地避免万向死锁的发生。
