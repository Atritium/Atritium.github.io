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





热更新原理：

AssetBundle包可以从服务器上下载然后加载进游戏，所以我们需要热更的资源只要丢在服务器上，等进入游戏时再进行下载就可以了。但AssetBundle只能打包资源，不能打包C#代码（可能出于安全和方便管理），因此我们需要考虑代码热更的方式。

现在主流的一些热更方式：

- ToLua/XLua
- ILRuntime
- HybridCLR

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

## 1 C#调用Lua

### 1.1 Lua解析器

```c#
//Lua解析器
var luaEnv = new LuaEnv();

//执行Lua代码
luaEnv.DoString("print('Hello,World!')");
//执行一个Lua脚本：运用Lua多脚本执行的方法require，脚本默认从Resources文件加载，并且由于Resources只支持某几种文件，比如txt,bytes等，因此文件名要写为LuaTest.lua.txt
luaEnv.DoString("require('LuaTest')");

//清除Lua中没有手动释放的对象
luaEnv.Tick();
//销毁解析器
luaEnv.Dispose();
```

### 1.2 文件加载重定向

解决Lua文件默认从Resources文件执行不方便的问题。

```c#
void Start() {
	//Lua解析器
	var luaEnv = new LuaEnv();
    //添加自定义文件加载规则
	luaEnv.AddLoader(MyCustomABLoader);
}

public byte[] MyCustomABLoader(ref string filepath){
	//...
    //这里可以写从AB包加载的相关逻辑
    //注意打进AB包时lua文件的后缀要改成.txt的
    var path = Application.streamingAssetsPath + "/lua";
	var ab = AssetBundle.LoadFromFile(path);
	var tx = ab.LoadAsset<TextAsset>(filePath + ".lua");
	return tx.bytes;
}
```

Lua会在调用`require("Lua文件名")`时依次访问目前所有的文件加载规则，直到找到对应的文件。

访问 自定义加载规则1->自定义加载规则2->...->默认加载规则。

### 1.3 luaEnv.Global

#### 1.3.1 获取全局变量

```c#
//得到Lua中的大G表
public LuaTable Global{
	get{
		return luaEnv.Global;
	}
}

private void Start(){
    //虽然Lua中的数值类型只有Number一种，但我们可以根据具体的数值，在拿到C#里的时候进行转换，如下
    var i = Global.Get<int>("testNumber");
    Global.Set("testNumber",55);
}
```

#### 1.3.2 获取全局函数

**lua文件**

```lua
testFun = function()
	print("Hello,World!")
end
```

**C#文件**

```c#
//得到Lua中的大G表
public LuaTable Global{
	get{
		return luaEnv.Global;
	}
}
//定义一个和方法相同形式的委托
public delegate void CustomCall();
private void Start(){
    //1.自定义委托/C#自带委托Action&Func/UnityAction
	var call = Global.Get<CustomCall>("testFun");
	call();
    
    //2.XLua提供的方法（不建议使用）
    var lf = Global.Get<LuaFunction>("testFun");
    lf.Call();
    //如果没有定期回收的话，需要手动释放，防止内存泄露，因为LuaFunction持有Lua虚拟机里的引用
    lf.Dispose();   
}
```

另外，对于自定义委托，无参无返回值的XLua已经帮助处理了，所以不需要特殊声明，但如果是有参有返回值的委托，需要在前面加上`[CSharpCallLua]`。

```c#
//得点XLua工具里的生成代码，XLua才能识别并生成对应的
[CSharpCallLua]
public delegate int CustomCall(int num);
```

#### 1.3.3 获取List/Dictionary

**lua文件**

```lua
--List
testList1 = {1,2,3,4,5}
testList2 = {1,"123",true}

--Dictionary
testDict1 = {
	["1"] = 1,
	["2"] = 2,
	["3"] = 3,
	["4"] = 4,
}
testDict2 = {
	["1"] = 1,
	[true] = 1,
	[false] = true,
	["123"] = false,
}
```

**C#文件**

```c#
//得到Lua中的大G表
public LuaTable Global{
	get{
		return luaEnv.Global;
	}
}
//值拷贝/浅拷贝：不会改变lua中的内容
var list1 = Global.Get<List<int>>("testList1");
var list2 = Global.Get<List<object>>("testList2");
var dict1 = Global.Get<Dictionary<string,int>>("testDict1");
var dict2 = Global.Get<Dictionary<object,object>>("testDict2");
```

#### 1.3.4 获取类

**lua文件**

```lua
testClass = {
	testInt = 2,
	testBool = true,
	testString = "123",
	testFun = function()
    	print("Hello,World!")
     end
}
```

**C#文件**

```c#
public class CallLuaClass{
	//在这个类中声明成员变量，名字一定要和Lua里声明的一样
	public int testInt;
	public bool testBool;
	public string testString;
	public UnityAction testFun;
	
	//这个自定义类中的变量可以比Lua中更多也可以更少
}
public class Program{
	//...
	var obj = Global.Get<CallLuaClass>("testClass");
}
```

#### 1.3.5 获取接口

**lua文件**

```lua
testInterface = {
	testInt = 2,
	testBool = true,
	testString = "123",
	testFun = function()
    	print("Hello,World!")
     end
}
```

**C#文件**

```c#
//接口中不允许写成员变量，但可以写属性
[CSharpCallLua]
public interface ICallLuaInterface{
	int testInt{
		get;
		set;
	}
	bool testBool{
		get;
		set;
	}
	string testString{
		get;
		set;
	}
	UnityAction testFun{
		get;
		set;
	}
    
    //这个自定义接口中的变量可以比Lua中更多也可以更少，和上面类的规则一样
}
public class Program{
	//...
    //这里是引用拷贝！！！
	var obj = Global.Get<ICallLuaInterface>("testInterface");
}
```

#### 1.3.5 获取LuaTable

**lua文件**

```lua
test = {
	testInt = 2,
	testBool = true,
	testString = "123",
	testFun = function()
    	print("Hello,World!")
     end
}
```

**C#文件**

```c#
//引用拷贝！！
var luaTable = Global.Get<LuaTable>("test");
Debug.Log(luaTable.Get<int>("testInt"));

//如果没有定期回收，需要手动释放以防止内存泄漏，不建议使用
luaTable.Dispose();
```

## 2 Lua调用C#

