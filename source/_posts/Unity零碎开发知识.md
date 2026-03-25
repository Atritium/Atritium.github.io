---
title: Unity零碎开发知识
date: 2026-03-01 21:19:42
tags:
- Unity
categories: 
- Unity
---

# UGUI

## AVG游戏中打字机效果的实现方式

**————方案1：字符串截取**

这是最直觉的方案：在 `Update` 或 `Coroutine` 中，每隔一段时间增加显示的字符数量。

**实现逻辑**：`currentText = fullText.Substring(0, visibleCount);`

**缺点**

- **重建开销极高**：每多出一个字，都会触发 `Layout Dirty` 和 `Vertices Dirty`。如果是原生 `Text`，整个文本框的网格会每帧重构
- **不支持富文本**：截取到 `<color=#ff0000>` 中间时，标签会失效并直接显示源码

**————方案2：`TextMeshPro`的 `maxVisibleCharacters` 属性**

 **实现逻辑**：预先将**全量文本**赋给 TMP 组件，然后通过代码动态改变 `tmp.maxVisibleCharacters` 的数值。

**底层原理**：

- **一次性布局**：TMP 在第一帧就计算好了所有文字的位置（排版已定死，不会跳动）
- **网格级隐藏**：当你修改 `maxVisibleCharacters` 时，TMP 只是在生成网格时，将超出索引的字符顶点的透明度设为 0（或者不填入顶点索引）

**优点**

- **性能极佳**：不会触发 Layout 重建，仅微量修改顶点数据
- **完美支持富文本**：标签会被预先解析，不会出现标签源码外露

## Mask VS RectMask2D

### 1 Mask (模板遮罩)

它是 UGUI 最早的遮罩方案，利用的是显卡的 **Stencil Buffer (模板缓冲)（GPU渲染一个像素到屏幕之前会经历一系列测试，其中一个就是模板测试）**。

- **原理**：
  - **写入阶段**：渲染 Mask 自身的 Image，并在对应的屏幕像素位置把 Stencil Buffer 的值改写为 **1**
  - **测试阶段**：渲染 Mask 下方的子物体（如子图片的 Image）。这些子物体的 Shader 会配置为：**“只渲染 Stencil 值为 1 的像素”**
  - **清理阶段**：渲染完所有子物体后，再跑一个 Pass 把这块区域的 Stencil 值改回 **0**（为了不影响屏幕上其他的 UI）
- **优点**：
  - **支持任意形状**：只要你的 Image 是圆的、星形的，遮住的就是对应的形状
- **缺点**：
  - **多出 Draw Call**：Mask 组件本身会产生 **2 个额外的 Draw Call**（一个开启模板，一个关闭）
  - **打断合批**：Mask 内部的元素无法与 Mask 外部的元素合批。如果有多个 Mask，它们之间也无法合批
  - **GPU 消耗**：操作模板缓冲对 GPU 有一定压力

### 2 RectMask2D（矩形遮罩）

它是后来推出的优化版本，直接在**Shader 层级**通过计算坐标来决定像素是否显示。

**原理**：它将遮罩矩形的四个边界坐标（左下角、右上角）传给子物体的 Shader。Shader 在渲染每个像素时，判断该点是否在矩形内，不在就丢弃（Discard）

**优点**：

- **性能极高**：**零额外 Draw Call**。它不需要模板缓冲，不增加 SetPass Call
- **裁剪剔除 (Culling)**：它会自动计算子物体是否完全在矩形外，如果在外面，直接不渲染该物体，节省 CPU/GPU

**缺点**：

- **只能是矩形**：顾名思义，它不支持圆形或复杂形状
- **不支持旋转**：如果 UI 旋转了，它的裁剪范围依然是轴向的矩形

## 红点系统——基于前缀树

[放牛的星星：红点系统实现](https://zhuanlan.zhihu.com/p/85978429)

## 怎么把一个3D物体渲染在UI上面

### 1 方案一：Render Texture

这种方案是创建一个专门的相机来拍摄 3D 物体，然后将画面“投射”到 UI 的一张图片上。

- **步骤**：
  1. **创建 Render Texture (RT)**：在 Project 窗口右键创建
  2. **创建第二相机 (Camera)**：专门负责拍模型。将其 **Target Texture** 设置为刚才创建的 RT
  3. **UI 表现**：在 Canvas 下创建一个 **RawImage**，将它的 **Texture** 拖入这个 RT
- **优点**：
  - 3D 物体可以放在场景的任何角落（甚至离地八百米），完全不影响 UI 布局。
  - 可以使用不同的光照和 Shader，效果最稳定
- **缺点**：
  - 内存开销稍大（RT 是一张贴图）
  - 如果 UI 分辨率很高，RT 设置太小会模糊，设置太大费显存

### 2. 方案二：直接放在 Canvas 下（Screen Space - Camera）

如果你把 Canvas 的模式改为 **Screen Space - Camera**，UI 就不再是贴在屏幕上的“纸”，而是在相机前方的一块区域。

- **步骤**：
  1. 将 Canvas 模式设为 **Screen Space - Camera**，并拖入主相机
  2. 将 3D 物体的 **Layer** 设为 UI
  3. 直接将 3D 物体拖入 Canvas 层级下
  4. 通过调整物体的 **Z 轴深度**，让它位于 UI 背景和前景之间
- **优点**：
  - 零内存额外开销，直接渲染
  - 模型可以和 UI 产生完美的遮挡关系
- **缺点**：
  - 模型会受到主相机参数（如 FOV）的影响
  - 如果界面有很多层级，管理 Z 轴位置会非常痛苦

如果模型比 UI 背景大，超出的部分会显得很突兀，怎么裁剪超出的3D部分？

- **回答**：
  - 如果用 **方案一 **：直接调整 RawImage 的大小，或者给 RawImage 加一个 `Mask` / `RectMask2D` 即可
  - 如果用 **方案二**：需要给 3D 物体定制一个 Shader，利用 `Stencil Buffer`（模板缓冲）进行裁剪

## 怎么把粒子渲染在UI上？—— UIParticle插件

除了和3D物体采用相同的方式外，还可以使用成熟的插件UIParticle。

原理：这类组件会劫持粒子系统的渲染数据，动态地修改粒子的 `Vertex Data`（顶点数据），将其注入到 UGUI 的渲染流水线中。

优点：

- 粒子会像 Image 一样遵循 `RectTransform` 布局

- **支持合批**：可以和 UI 元素一起进行 Canvas 重绘

- **支持 Mask**：粒子可以被 UGUI 的 `Mask` 正常裁剪

# 其他

## sharedMaterial VS material

在 Unity 中，每个 `Renderer` 都有 `material` 和 `sharedMaterial` 两个属性。

- `sharedMaterial`
  - 访问`sharedMaterial`时，获取的是磁盘上那个材质球文件的直接引用
  - 如果你有 100 个小兵都使用同一个 `Soldier_Mat`，那么这 100 个小兵的 `sharedMaterial` 指向的都是同一个内存地址
  - 如果你通过代码修改了其中一个人的 `sharedMaterial.color = Color.red`，那么**所有**使用这个材质的小兵都会瞬间变红，甚至连你工程目录里的材质球文件也会被修改
- `material`
  - 访问`material`时，会先检查是否已经实例化过材质，如果没有，则`new`一个材质副本，并赋值给该物体
  - 内存泄漏：即使你删除了这个物体，那个被`new`出来的材质依然留在内存中
    - 解决方案：在物体的`OnDestory`中进行清理

使用`MaterialPropertyBlock`解决又想使用同一材质促使合批，又想展示个体的差异化问题：

```c#
// 1. 建议定义为静态或全局重用，减少 new 的开销
private static MaterialPropertyBlock _propBlock;

void Start() {
    if (_propBlock == null) _propBlock = new MaterialPropertyBlock();
    
    Renderer rend = GetComponent<Renderer>();

    // 2. 先从 Renderer 中获取当前的属性块（防止覆盖掉其他已有的属性）
    rend.GetPropertyBlock(_propBlock);

    // 3. 设置你想要修改的 Shader 变量名（建议使用 Shader.PropertyToID 优化性能）
    _propBlock.SetColor("_Color", Color.red);

    // 4. 将修改后的属性块重新应用给 Renderer
    rend.SetPropertyBlock(_propBlock);
}
```

