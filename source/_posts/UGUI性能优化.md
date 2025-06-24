---
title: UGUI性能优化
date: 2025-06-24 17:34:25
tags: 
- Unity
- UGUI
categories: 
- Unity
description: 在公司项目里接触最多的就是UGUI，因此也负责了一部分UGUI优化相关的工作，在此进行记录。
---

# 减少Draw Call

**Draw Call**

在Unity里，Draw Call指的是CPU发出的绘制请求，其中包含了绘制所需的所有信息，如纹理信息、着色器等，Draw Call太多会增加CPU的消耗，会产生更多的发热和耗电，减少Draw Call就是合并请求（合批），以减少CPU在提交命令上花费太多时间。

**Frame Debugger**

分析工具我主要使用的是Unity的Frame Debugger（Window-Analysis-Frame Debugger），这个工具可以在游戏运行时冻结特定帧，并查看用于渲染该帧的各个渲染调用的具体情况。在Frame Debugger里，一个渲染调用称为Draw Mesh，通常就是一个Draw Call。

例外情况：当启用GPU Instancing时，多个相同网格的绘制可能合并为一个Draw Call，但在Frame Debugger中可能仍显示为多个"Draw Mesh"。

<img src="UGUI性能优化/image-20250624164104269.png" alt="image-20250624164104269" style="zoom:50%;" />

**减少Draw Call**

减少Draw Call即尽可能地进行合批，UI合批要求UI在同一个Canvas下，并拥有相同的材质（Material）和纹理（Texture），在Hierarchy里尽量相邻（如果两个满足合批条件的UI之间夹了一个不满足合批条件的UI，它们俩无法合批）。

**相同材质**就是使用相同的材质球。

这里我测试了一下，在场景里放置两个Img，给第一个Img放一个新建的默认材质球，然后复制一份这个材质球，赋给第二个Img，然后用Frame Debugger进行抓帧，发现它们不能合批。这说明了即便材质球的参数一模一样，只要是两个材质球实例，就无法合批。

**相同纹理**则是保证Image.sprite.texture一致。

一般情况下不同Sprite的Texture是不一致的，但Unity提供了一个东西叫做图集（Sprite Atlas），可以把多张小图打包成一张大图，当Sprite被打入同一个图集，那么使用这些Sprite的UI组件最终引用的都是同一张纹理，即可合批。

**一个坑**

在使用Frame Debugger的时候我发现Draw Mesh的合并有时候成功有时候失效，后面控制变量找到了原因：我在Prefab编辑页面里运行游戏进行分析，Draw Mesh合并就会失效，要退到Scene里才会恢复正常。



