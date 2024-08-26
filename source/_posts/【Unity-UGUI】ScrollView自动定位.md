---
title: 【Unity-UGUI】ScrollView自动定位
date: 2024-08-26
tags: 
- Unity 
- UGUI
categories: 
- Unity
---



最近遇到了这样的需求：滑动列表自动定位到第一个未领取的奖励，翻译一下其实就是ScrollView的自动定位问题。

ScrollView的滑动、定位问题一般都与**ScrollRect.normalizedPosition**这个字段有关，这个字段描述了ScrollView组件下Content（内容）和Viewport（视窗）的相对位置，滑动、定位问题本质上就是寻找新的normalizedPosition的问题。

> Unity官方对这个字段的描述：
> The scroll position as a Vector2 between (0,0) and (1,1) with (0,0) being the lower left corner.

新的normalizedPosition的计算方法：当前的ScrollRect.normalizedPosition+偏差量（根据目标Item位置和当前viewport位置进行计算，详细见代码）
```csharp
using UnityEngine;
using UnityEngine.UI;

public class ScrollEx : MonoBehaviour
{
    private ScrollRect scrollRect;
    private RectTransform viewport;
    private RectTransform content;

    [SerializeField][Header("要定位到的目标物体")]
    private RectTransform targetItem;

    void Start() {
        if (scrollRect == null) scrollRect = this.GetComponent<ScrollRect>();
        if (viewport == null) viewport = this.transform.Find("Viewport").GetComponent<RectTransform>();
        if (content == null) content = this.transform.Find("Viewport/Content").GetComponent<RectTransform>();

        //如果content上面加了Layout和content_fitter组件就需要加上这一句，让content自适应在计算之前进行
        LayoutRebuilder.ForceRebuildLayoutImmediate(content.GetComponent<RectTransform>());
        Nevigate(targetItem);
    }

    public void Nevigate(RectTransform item) {
        //算出content需要偏移的量
        Vector3 itemCurrentLocalPos = scrollRect.GetComponent<RectTransform>().InverseTransformVector(ConvertLocalPosToWorldPos(viewport));
        Vector3 itemTargetLocalPos = scrollRect.GetComponent<RectTransform>().InverseTransformVector(ConvertLocalPosToWorldPos(item));
        Vector3 diff = itemTargetLocalPos - itemCurrentLocalPos;
        diff.z = 0.0f;
        var difPos = new Vector2(
            diff.x / (content.GetComponent<RectTransform>().rect.width - viewport.rect.width),
            diff.y / (content.GetComponent<RectTransform>().rect.height - viewport.rect.height)
        );
        var newNormalizedPos = scrollRect.GetComponent<ScrollRect>().normalizedPosition + difPos;
        newNormalizedPos = new Vector2(Mathf.Clamp01(newNormalizedPos.x), Mathf.Clamp01(newNormalizedPos.y));
        scrollRect.GetComponent<ScrollRect>().normalizedPosition = newNormalizedPos;
    }

    private Vector3 ConvertLocalPosToWorldPos(RectTransform target) {
    	//由于RectTransform的localPos受到pivot的影响，所以需要先计算偏差进行还原
        var pivotOffset = new Vector3(
            (0.5f - target.pivot.x) * target.rect.size.x,
            (0.5f - target.pivot.y) * target.rect.size.y,
            0f);

        var localPosition = target.localPosition + pivotOffset;

        return target.parent.TransformPoint(localPosition);
    }
}
```

**Q：为什么在这里，我们要使用ConvertLocalPosToWorldPos计算对应UI元素的世界坐标，而不是通过RectTransform.position或者Transform.position来获取世界坐标？**

A：我们使用方法获取到的世界坐标，是UI**真正轴心点**的坐标，而后面两个其实指的都是UI **pivot**的世界坐标（是的，对于UI来说，Transform.pos获取到的也是pivot的坐标）。

