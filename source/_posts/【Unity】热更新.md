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

## 1 CSharp调用Lua

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

## 2 Lua调用CSharp

Lua没有办法直接访问C#，所以一定是先从C#调用Lua后，才把核心逻辑交给Lua来编写。

```c#
public class Main : MonoBehaviour{
	private void Start(){
		//这里调用了一个已经写好的Lua管理器
		LuaManager.GetInstance().Init();
		LuaManager.GetInstance().DoLuaFile("Main");
	}
}
```

### 2.1 Lua调用C#类

- Lua调用C#类
  - Unity自带类&C#自定义类
    - CS.命名空间.类
      - `CS.UnityEngine.GameObject`
      - 实例化类对象：`local obj = CS.UnityEngine.GameObject("名字")`
      - 为了节约性能&简便，可以在Lua里使用全局变量对一些C#类进行声明，如`GameObject = CS.UnityEngine.GameObject`
  - Mono类
    - 不能用new实例化，需要用组件挂载的方式，但xLua不支持AddComponent<泛型>()这种无参泛型模式，因此使用另一个重载：AddComponent(Type t)
    - `obj.AddComponent(typeof(Mono类名))`
- Lua调用类对象
  - 静态方法和变量
    - 使用 . 进行调用
    - `local obj = GameObject.Find("名字")`
  - 成员变量
    - 实例化的对象.变量名
    - `obj.transform.location`
  - **成员方法**
    - 实例化的对象:方法名

### 2.2 Lua调用C#枚举

和类的调用规则一样：`CS.命名空间.枚举名.枚举成员`

其他需要注意的：

```lua
--枚举的转换
--假设C#里我们自定义了一个枚举E_MyEnum
E_MyEnum = CS.E_MyEnum
local c = E_MyEnum.Idle

--数值转枚举
local a = E_MyEnum.__CastFrom(1)
--字符串转枚举
local b = E_MyEnum.__CastFrom("Idle"
```

### 2.3 Lua调用C# Array/list/Dictionary

**C#**

```c#
public class Test{
	public int[] array = new int[5]{1,2,3,4,5};
	public List<int> list = new List<int>();
	public Dictionary<int,string> dict = new Dictionary<int,string>();
}
```

**Lua**

**Array**

```lua
local obj = CS.Test()

--Lua使用数组，C#里怎么用Lua里就怎么用
print(obj.array.Length)
print(obj.array[0])

--遍历也要按C#的规则来：索引从0开始
for i=0,obj.array.Length-1 do
	print(obj.array[i])
end

--在lua中创建一个C#的数组，使用Array类提供的静态方法即可
local array = CS.System.Array.CreateInstance(typeof(CS.System.Int32),10)
```

**List**

```lua
local obj = CS.Test()

--添加元素
obj.list:Add(1)
obj.list:Add(2)
obj.list:Add(3)

--遍历
for i=0,obj.list.Count-1 do
	print(i)
end

--在lua中创建一个C#的list
local List_String = CS.System.Collections.Generic.List(CS.System.String)
local list = List_String()
list.Add(111)
```

**Dictionary**

```lua
local obj = CS.Test()

--添加元素
obj.dict:Add(1,"123")

--遍历
for k,v in pairs(obj.dict) do
	print(k,v)
end

--在lua中创建一个C#的Dictionary<string,Vector3>
local Dic_String_Vector3 = CS.System.Collections.Generic.Dictionary(CS.System.String,CS.UnityEngine.Vector3)
local dict = Dic_String_Vector3()
dict:Add("123",CS.UnityEngine.Vector3.right)

--**在lua中创建的dict获取值
print(dict:get_Item("123"))
dict.set_Item("123",nil)
```

### 2.4 Lua使用C#拓展方法

C#拓展方法是一种允许你为现有的类型添加新方法，而无需修改原类型代码或创建子类的功能。扩展方法看起来就像是目标类型的实例方法一样调用，但实际上它是一个静态方法。它的核心目的是为已定义的类（包括.NET框架的内置类型或第三方库中的类）增加功能，而无需修改或继承该类。

特点：

- **静态方法**：扩展方法实际上是定义在静态类中的静态方法。
- **`this` 关键字**：扩展方法的第一个参数前加上 `this` 关键字，表示该方法是扩展到这个类型的。

定义：

```c#
public static class StringExtensions
{
    // 扩展方法，添加到 string 类型
    public static string Reverse(this string str)
    {
        char[] charArray = str.ToCharArray();
        Array.Reverse(charArray);
        return new string(charArray);
    }
}
```

使用：

```c#
string example = "Hello";
string reversed = example.Reverse();  // 调用扩展方法
Console.WriteLine(reversed);          // 输出 "olleH"
```

要在lua中调用C#的拓展方法，要在声明拓展方法的静态类前面加上：`[LuaCallCSharp]`，调用方法则和普通调用实例方法一致。

另外，该特性可以提高lua访问C#的性能。
