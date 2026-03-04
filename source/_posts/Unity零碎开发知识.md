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

