---
title: UGUI渲染全流程和性能优化
date: 2025-06
tags: 
- Unity
- UGUI
categories: 
- Unity
---

# 1 渲染全流程

UGUI渲染流程可以拆分为四个阶段：Layout（布局阶段）、Graphic Rebuild（图形重建）、Batching（合批）、Draw（渲染提交）。

## 1.1 Layout（布局）

在这个阶段，CPU会重新计算部分UI的尺寸、锚点和位置。部分UI：当UI的某些属性发生变化时，UI会调用`SetLayoutDirty()`把自己注册到 `CanvasUpdateRegistry` 的 Layout Rebuild 队列。等到布局阶段，会依次对这个队列里的UI以及它们影响到的UI进行布局重算。

会导致UI进入布局阶段的原因主要有以下这些：

- **UI尺寸和形状发生了变化**
  - **修改 `sizeDelta`**：直接改动 UI 的宽度或高度
  - **修改 `Pivot`（中心点）**：改变中心点会影响旋转和缩放的基准，进而可能触发父级对位置的重新微调
  - **修改 `Anchors`（锚点）**：锚点改变意味着 UI 与父容器的相对关系变了，必须重新计算布局
- **显隐状态的改变**
  - **`gameObject.SetActive(true/false)`**：当你隐藏一个列表项，由于它不再占位，父级布局必须立即重新收缩（如果是 `ContentSizeFitter`）并排列剩余的孩子
- **UI结构和层级发生变化**
  - 只要 UI 树的“骨架”动了，布局阶段就必须介入
  - **添加/删除子物体**：在一个带有 `LayoutGroup` 的物体下 `Instantiate` 或 `Destroy` 元素
  - **修改层级顺序 (Sibling Index)**：改变子物体在列表中的先后顺序（例如拖拽排序）
  - **切换父物体 (`SetParent`)**：将 UI 从一个容器移到另一个容器

简单来说，一个UI的某种变化如果可能对自己或其他UI的布局造成影响，那就会导致它进入布局阶段，在这个阶段里，会对它自己或者它造成影响的UI的布局进行重新计算。

举例：比如当你修改一个UI的`sizeDelta`时，如果父物体挂载了`ContentSizeFitter`，那么就必须在布局阶段根据子物体的尺寸，重新计算自己的尺寸。

那如果父物体没有挂载`ContentSizeFitter`呢？那也会进入布局阶段。因为我们在修改数值时并不能够排除它对其他UI是否不会产生影响，只是在进入布局阶段后，没有进行布局相关的计算。

## 1.2 Graphic Rebuild（图形重建）

**Graphic Rebuild（图形重建）** 的本质是 CPU 调用了 `Graphic.RebuildMesh`，也就是重新执行了 **`OnPopulateMesh`** 函数。这个函数非常消耗 CPU，因为它涉及大量的顶点坐标、UV、颜色数组的内存申请和数值计算。

**会导致UI进入图形重建阶段的有这些**

- **修改 Image 的 Sprite 或 Texture**：切换贴图（更换“颜料”）
- **修改 Image 的 Color 或 Alpha**：
  - *注意*：直接修改 `image.color = ...` 会导致该组件重绘顶点颜色缓冲区
- **修改 Image 的填充方式（Fill Amount）**：比如技能冷却的 CD 进度条滑动，每一帧都在改网格形状
- **修改 Text 的内容（text 属性）**：这是最重的，改一个字符，整段文字的顶点都要重算

**核心工作过程**

当 `CanvasUpdateRegistry` 轮到处理图形队列时，它会调用每一个脏组件的 `Rebuild` 方法，最终指向核心函数：`OnPopulateMesh`。

- **顶点计算**
  - 系统会根据 `RectTransform` 算好的宽高，计算出网格的顶点
  - **如果是 Simple 模式**：很简单，只有 4 个顶点，2 个三角形
  - **如果是 Sliced (九宫格) 模式**：这就是 `sizeDelta` 变动最累的地方。它需要计算 **36 个顶点**，并把它们拆解成 **9 个矩形区域**，确保边角不拉伸
  - **如果是 Text**：每个字符都是一个四边形
- **填充UV和颜色**
  - **UV 映射**：告诉 GPU 图片的哪一部分该贴在哪个顶点上
  - **颜色填充**：将`Graphic.color`的数值写进每一个顶点的属性里
- **数据提交**
  - 所有的顶点、UV、颜色都会被丢进一个叫做`VertexHelper`的临时容器里面，最后它会将数据一次性推给`CanvasRenderer`

| **操作**                   | **影响范围**                  | **优化方案**                                                 |
| -------------------------- | ----------------------------- | ------------------------------------------------------------ |
| **修改 Text 内容**         | 极高 (涉及字库采样、顶点生成) | 减少频繁变动的 Text，或用 `TextMeshPro`                      |
| **修改 Sliced Image 尺寸** | 中 (九宫格顶点计算)           | 尽量使用 Simple 模式                                         |
| **修改颜色/透明度**        | 中                            | **必杀技**：用 `CanvasGroup.alpha` 或 Shader 修改颜色，这样可以**彻底绕过** Graphic Rebuild |

**原生 Text**：当你修改 `text = "123"` 为 `text = "1234"` 时，它通常会销毁旧的顶点数组，然后重新分配内存来创建新的数组。这种频繁的 **内存申请与垃圾回收 (GC)** 是性能杀手。

**TMP**：它内部维护了一个**持久化的顶点缓存**。

- 如果你把“123”改为“1234”，它会直接在原有的 Buffer 里追加数据
- 如果字数变少了，它只是简单地把多余的顶点索引设为 0，而**不释放内存**
- **结果**：修改内容时，TMP 造成的 **CPU 耗时和内存抖动（GC Alloc）** 显著低于原生 Text

## 1.3 Batching（合批）

这是Canvas最繁重的工作，目的是为了减少**Draw Call**。它的本质是，将多个UI元素的网格合并成一个巨大的网格。

**合批的三个硬指标**

- **相同的材质（Material）**：使用同一个 Shader 和同样的参数
- **相同的贴图（Texture/Atlas）**：这就是为什么我们要用 **Sprite Atlas（图集）**。如果两个 Image 引用的贴图不在一张图集里，合批必然断开
- **相同的深度（Depth）**
  - 系统会给每个 `Graphic` 元素分配一个整数 `Depth` 值
  - 初始值：如果一个 UI 元素不遮挡任何东西，它的 `Depth = 0`
  - 系统会检查这个 UI 元素在屏幕空间（Rect）内，**重叠（Overlap）** 了哪些已经计算过深度的 UI，如果它覆盖了多个 UI，它会取这些 UI 中 **最大的 Depth 值**
    - 如果它能和下面那个最大 Depth 的 UI **合批**（材质、贴图一致），它的 `Depth = MaxDepth`
    - 如果它不能和下面那个最大 Depth 的 UI 合批，它的 `Depth = MaxDepth + 1`

## 1.4 Draw

所谓的**Draw阶段**，其实是**CPU将合批后的数据正式提交给GPU渲染管线** 的过程。

- **提交渲染队列**
  - UGUI 属于 **Transparent（半透明）** 渲染队列。
    - **排序准则**：GPU 在画 3D 物体时通常由近到远（为了 Early-Z 优化），但 UI 必须**由后到前（Back-to-Front）** 绘制
    - **Draw 顺序**：Canvas 会严格按照我们在 **Batch 阶段** 算好的 `Depth` 顺序，把一个一个的 Batch（合批包）扔进渲染队列
- **设置渲染状态**
  - 这是 Draw 阶段最耗 CPU 的“前置动作”。在发出真正的绘制指令前，CPU 必须配置好环境
  - **Set Texture**：绑定当前 Batch 对应的 **Sprite Atlas（图集）**
  - **Set Shader & Pass**：指定 UI 材质使用的着色器（比如 `UI/Default`）
  - **Set Stencil/Clip**：如果你的 UI 在 `Mask` 里面，这一步会设置 **模板缓冲区（Stencil Buffer）**，告诉 GPU 哪些像素该裁掉
  - **Update Uniforms**：更新变换矩阵（Canvas 坐标转屏幕坐标）和材质参数（如 Tint Color）
- **发出Draw Call指令**
  - 当一切准备就绪，CPU 会调用底层图形 API（如 `glDrawElements` 或 `DrawIndexedPrimitive`）：
  - **内容**：告诉 GPU：“去这块内存地址，拿这一组顶点和索引，用刚才设好的材质和贴图，画出这 $N$ 个三角形”

# 2 性能优化

## 2.1 减少 UI Rebuild && UI Rebatch

### 2.1.1 UI Rebuild VS UI Rebatch

**重建 (Rebuild)** 是**“个体行为”**： 比如你改了 A 图片的颜色，只有 A 图片需要重新计算它的顶点颜色。这叫 A 发生了 Graphic Rebuild。

**合批 (Rebatch)** 是**“集体行为”**： 因为 A 变了，它所属的这个 Canvas 必须重新扫描所有人，看看 A 变了之后还能不能和邻居 B、C 打包在一起。这叫 Canvas 发生了 Rebatch。

注意：触发Rebuild一定会触发Rebatch，但是触发Rebatch不一定会触发Rebuild。

例如：修改一个UI的`anchoredPosition`，不会触发Rebuild，但因为位置变化可能会导致UI的遮挡关系发生变化，Canvas需要重新计算Depth，因此会触发Rebatch。

### 2.1.2 减少UI Rebuild

-  **用 `CanvasGroup.alpha` 替代物理显隐**

  - **错误做法**：`gameObject.SetActive(true/false)` 或修改 `Image.color.a`
  - 代价：前者触发整个层级的 Layout 和 Graphic 双重重建；后者触发该元素的 Graphic 重建
  - **原理**：`CanvasGroup` 的透明度变化是在渲染层（GPU 或 Canvas 顶点属性统一乘积）处理的，**完全不触发** C# 端的网格重算（`OnPopulateMesh`）

- **避免频繁改动RectTransform的尺寸**

  - 这会强制触发 **Layout Rebuild**（重新计算布局）和 **Graphic Rebuild**（因为网格顶点位置变了）

  - 如果只是想做缩放效果（比如按钮点击缩入），请修改 **`transform.localScale`**

    Scale 的改变是在矩阵变换阶段处理的，不会触发网格顶点数据的重新生成

- **抛弃自动布局组件**

  - `VerticalLayoutGroup`、`HorizontalLayoutGroup` 和 `ContentSizeFitter` 是 Rebuild 的万恶之源
  - 这些组件在 `OnEnable` 或子物体变动时，会触发极度复杂的递归计算
  - 用代码手动计算位置替代。手动计算坐标只需一次简单的赋值，而 LayoutGroup 每一帧都在通过“脏标记”监控子物体，性能消耗差了几个数量级

### 2.1.3 减少UI Rebatch

除了上面减少Rebuild的方式之外，还有：

- **动静分离**
  - **做法**：将 UI 分成两类，分别放在不同的 `Canvas` 或 `Sub-Canvas`（子画布）下。
    - **静态 Canvas**：放背景、不动的装饰、极少点击的按钮
    - **动态 Canvas**：放血条、CD 倒计时、摇杆、飘字数字
  - **原理**：**Rebatch 是以 Canvas 为边界的。** 动态 Canvas 里的数字每帧跳动，只会引起这个小 Canvas 的 Rebatch。而静态 Canvas 因为内部没有任何元素变脏，CPU 连看都不会看它一眼，直接复用上一帧的合批包
- **巧用 Sub-Canvas（局部隔离）**
  - 可以直接在某个频繁变动的 UI 节点上挂载一个 **`Canvas` 组件**
- **优化 Overlap（重叠）减少深度计算压力**
  - Rebatch 过程中最耗时的部分是 **深度排序（Depth Sort）**
  - 尽量让 UI 元素在空间上**互不重叠**（比如背包格子之间留 1 像素缝隙）

## 2.2 渲染提交阶段：减少Draw Call

**Draw Call**

在Unity里，Draw Call指的是CPU发出的绘制请求，其中包含的数据有：顶点、纹理、shader等。

在现代硬件中，GPU的处理能力通常很强，假设一个场景有2000个Draw Call，CPU可能需要花20ms才能把这些指令发完，而GPU画完它们只需要5ms。也就是说，Draw Call太多的后果是GPU大部分时间都在等CPU发指令，这时游戏帧率就会卡在CPU提交这一步。

减少Draw Call就是合并请求（合批），以减少CPU在提交命令上花费太多时间。

**Frame Debugger**

分析工具我主要使用的是Unity的Frame Debugger（Window-Analysis-Frame Debugger），这个工具可以在游戏运行时冻结特定帧，并查看用于渲染该帧的各个渲染调用的具体情况。在Frame Debugger里，一个渲染调用称为Draw Mesh，通常就是一个Draw Call。

例外情况：当启用GPU Instancing时，多个相同网格的绘制可能合并为一个Draw Call，但在Frame Debugger中可能仍显示为多个"Draw Mesh"。

<img src="UGUI性能优化/image-20250624164104269.png" alt="image-20250624164104269" style="zoom:50%;" />

**减少Draw Call**

减少Draw Call即尽可能地进行合批，UI合批要求UI在同一个Canvas下，并拥有相同的材质（Material）、纹理（Texture）和深度（Depth）。

**相同材质**就是使用相同的材质球。

这里我测试了一下，在场景里放置两个Img，给第一个Img放一个新建的默认材质球，然后复制一份这个材质球，赋给第二个Img，然后用Frame Debugger进行抓帧，发现它们不能合批。这说明了即便材质球的参数一模一样，只要是两个材质球实例，就无法合批。

**相同纹理**则是保证Image.sprite.texture一致。

一般情况下不同Sprite的Texture是不一致的，但Unity提供了一个东西叫做图集（Sprite Atlas），可以把多张小图打包成一张大图，当Sprite被打入同一个图集，那么使用这些Sprite的UI组件最终引用的都是同一张纹理，即可合批。

**相同深度**是保证Depth一致。

尽量让使用同一图集的物体在 Hierarchy 层级面板里排在一起。并且减少不同图集物体之间的重叠（Overlap）。

**另外少用Mask**

Mask实现的具体原理是一个Drawcall来创建Stencil mask(来做像素剔除)，然后画所有子UI，再在最后一个Drawcall移掉Stencil mask。这头尾两个Drawcall无法跟其他UI操作进行Batch，所以表面上看加个Mask就会多2个Drawcall，而且Mask中的UI元素无法与其他batch，所以很多原本可以合并的UI就无法合并了，从而增加DrawCall。

可以使用RectMask2D进行替代，它是基于Shader裁切的，不产生额外的Draw Call。

**一个坑**

在使用Frame Debugger的时候我发现Draw Mesh的合并有时候成功有时候失效，后面控制变量找到了原因：我在Prefab编辑页面里运行游戏进行分析，Draw Mesh合并就会失效，要退到Scene里才会恢复正常。

## 2.3 GPU绘制阶段：overdraw优化

overdraw指的是一个像素被多次绘制。

对于边框类的UI，我们一般会切九宫，然后把full center这个选项取消勾选，这样中间的像素就不会被绘制。

Image修改透明度
