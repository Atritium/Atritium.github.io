---
title: Unity学习杂记
date: 2024-08-19
tags: 
- Unity 
categories: 
- Unity
---









速记，后面会归类到对应模块的博客。

# Addressable

**减少资源重复的正确打包方式**

[链接](https://www.bilibili.com/video/BV1xK4y1M7JN/?spm_id_from=333.788&vd_source=d1cd7437519192c36dc659c247e8160e)

假设有两个场景都含有一个相同的prefab，这个时候我们将这两个场景打成bundle包，分别是700M和750M。

我们希望减小bundle包体大小，于是我们决定从重复的prefab资源下手。我们的操作方式是：将prefab也做成Addressable的，列入bundle打包行列。

按我们的思路，重新打包后，prefab会单独打为一个bundle包，而场景的bundle包里，prefab资源会被去除。情况变为：当加载场景时，prefab的bundle也会被一起下载。

但实际上我们发现场景的bundle包包体大小在重新打包后并没有变化。

这是因为**prefab本质上一堆引用关系，而不是资源本身**，场景也类似。因此打包时，它们会把引用到的资源都打进来。

如果我们要真正实现减少资源重复，正确的方式是把重复的资源文件打成bundle包，而不是把prefab打成bundle包。



# UI ToolKit
