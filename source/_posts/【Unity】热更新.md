---
title: 【Unity】热更新
date: 2024-08-30 15:24:51
tags: 
- Unity
- AssetBundle
- Lua
categories: 
- Unity
---







# 资源管理

## AssetBundle

### 1 打包面板信息

#### 1.1 AB包的压缩格式

- LZMA
  - 默认压缩方式，能使压缩文件达到最小。但由于其压缩方式是基于整个AssetBundle的数据流实现的，因此在对其解压时需将整个流进行解压。
- LZ4
  - 一种基于块的压缩算法，压缩文件比LZMA大，但在解压时只需要解压包含所请求资源的字节的块。
- No Compression
  - 不压缩。

![image-20240830161446971](【Unity】热更新/image-20240830161446971.png)

#### 1.2 AB包生成的文件

- 对应资源的文件
  - 资源AB包
  - manifest文件
    - 记录了AB包的文件信息
    - 加载时可以用来提供资源信息、资源依赖关系等等
- 关键文件（和定义的output文件同名）
  - 主包
  - manifest文件：记录AB包之间的依赖信息

### 2 加载AB包

```c#
using System.Collections;
using System.Collections.Generic;
using TMPro.EditorUtilities;
using Unity.VisualScripting;
using UnityEngine;
using UnityEngine.Events;

public class ABLoadManager : SingletonMono<ABLoadManager>
{
    private string PathUrl
    {
        get
        {
            return Application.streamingAssetsPath + "/";
        }
    }

    private string MainABName
    {
        get
        {
#if UNITY_IOS
            return "IOS";
#elif UNITY_ANDROID
            return "Android";
#else
            return "PC";
#endif
        }
    }

    private AssetBundle mainAB = null;
    private AssetBundleManifest mainABManifest = null;
    private Dictionary<string, AssetBundle> abDict = new Dictionary<string, AssetBundle>();

    
    public T LoadRes<T>(string abName,string resName) where T : Object
    {
        if (!abDict.ContainsKey(abName))
        {
            AddToABDict(abName);
        }
        return abDict[abName].LoadAsset<T>(resName);
    }

    public void LoadResAsync<T>(string abName,string resName,UnityAction<T> action) where T : Object {
        if (!abDict.ContainsKey(abName))
        {
            AddToABDict(abName);
        }
        StartCoroutine(WaitForLoadResAsync(abName, resName, action));
    }

    private IEnumerator WaitForLoadResAsync<T>(string abName, string resName, UnityAction<T> action) where T :Object{
        AssetBundleRequest abr = abDict[abName].LoadAssetAsync<T>(resName);
        yield return abr;
        action?.Invoke(abr.asset as T);
    }

    private void AddToABDict(string abName)
    {
        //加载主包
        if (mainAB == null)
        {
            mainAB = AssetBundle.LoadFromFile(PathUrl + MainABName);
            mainABManifest = mainAB.LoadAsset<AssetBundleManifest>("AssetBundleManifest");
        }
        //根据依赖关系加载依赖包
        var strs = mainABManifest.GetAllDependencies(abName);
        foreach (var str in strs)
        {
            if (!abDict.ContainsKey(str))
            {
                abDict[str] = AssetBundle.LoadFromFile(PathUrl + str);
            }
        }
        //加载目标包
        abDict[abName] = AssetBundle.LoadFromFile(PathUrl+abName);
    }

    /// <summary>
    /// 卸载单个AB包
    /// </summary>
    public void UnLoadAB(string abName) {
        if (abDict.ContainsKey(abName))
        {
            abDict[abName].Unload(false);
            abDict.Remove(abName);
        }
    }

    /// <summary>
    /// 卸载所有的AB包
    /// </summary>
    public void UnLoadAllAB()
    {
        AssetBundle.UnloadAllAssetBundles(false);
        abDict.Clear();
        mainAB = null;
        mainABManifest = null;
    }
}
```



## Addressable

Addressable是在AssetBundle基础上实现的，详细用法：[参考](https://zhuanlan.zhihu.com/p/635796583)

# XLua

