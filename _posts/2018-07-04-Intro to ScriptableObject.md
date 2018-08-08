---
layout:     post
title:      "Intro to Scriptable Object"
subtitle:   "简易ScriptableObject使用入门"
date:       2018-07-04 21:00:00
author:     "Xjoshua"
header-img: "img/post-bg-default.jpg"
categories:
  - Tuturial
tags:
  - Unity
  - Tuturial
  - 序列化
---

## Intro to Scriptable Object

在做本地化的时候需要在Editor模式下测试不同的语言，现在项目中的本地化控制是通过读取系统语言（游戏体量小，并没有在设置中设置语言的选项），在游戏中通过Editor编辑器扩展直接修改也比较简单，但是每次都需要设置一下，有点麻烦。所以想起了Unity官方曾经推荐过的ScriptableObject：

>A class you can derive from if you want to create objects that don't need to be attached to game objects. This is most useful for assets which are only meant to store data.

>如果你想创建不需要附着在游戏物体上的项目，你可以使用这个类。常常用来构造只为存储数据的资产。

概括一下，有下面几个优点：
* 不需要附加到GameObject上
* 可以轻松序列化存储数据
* 在inspect中可以直观的检查和修改
	
像我这个编辑器调试语言的例子，把文件放到Editor文件夹中也不会影响到游戏打包。完美。

不过实际用起来还是和MonoBehavior有一点区别。通常MonoBehavior脚本可以直接在GameObject的Inspector脚本组件界面中修改，而ScriptableObject是在脚本生成的.asset实例文件上修改。

![asset实例文件](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/_posts/Image/2018-07-04-asset实例文件.png)

#### 使用实例

下面就是使用ScriptableObject的代码：
```csharp
/// <summary>
/// ScriptableObject类
/// </summary>
public class ConfigModel : ScriptableObject
{
	public int IntData;
	public bool BoolData;
	public string StringData;
}
	
public class EditorHelper
{
	// 创建ScriptableObject实例
	[MenuItem("EnhanceFunc/Create Data Asset")]
	static void CreateDataAsset()
	{
		// 创建实例的路径
		string holderAssetPath = "Assets/Editor/";
		if (!Directory.Exists(holderAssetPath))
			Directory.CreateDirectory(holderAssetPath);
		// 建立实例
		ConfigModel holder = ScriptableObject.CreateInstance<ConfigModel>();
		// 在路径下写入 .asset 实例文件
		AssetDatabase.CreateAsset(holder, holderAssetPath + "ConfigModel.asset");
	}
		
	// 读取存储数据
	static int GetAssetData()
	{
		ConfigModel config = AssetDatabase.LoadAssetAtPath<ConfigModel>("Assets/Editor/ConfigModel.asset");
		Debug.Log(config.IntData);
		Debug.Log(config.BoolData);
		Debug.Log(config.StringData);
		return config.IntData;
	}
}
```
#### 生命周期函数

注意，和Monobehavior类似，ScriptableObject也有生命周期方法，不过只有四个：

函数名 | 功能
-------|--------------
Awake |	This function is called when the ScriptableObject script is started.
OnDestroy | This function is called when the scriptable object will be destroyed.
OnDisable | This function is called when the scriptable object goes out of scope.
OnEnable | This function is called when the object is loaded.

同样，因为和MonoBehaviour有些不同（或者相同），这里的几个方法的调用时机会有一些概念上的问题。MonoBehaviour的Awake是在脚本（组件）创建的时候调用的，同样ScriptableObject的Awake也是在脚本实例创建出来的时候调用的。（一开始的时候我还以为是在类声明的时候调用的。。。ORZ，虽然看上去有点理应如此的感觉，不过在查这个问题的时候网上也有很多人抱怨过这个问题，为我的愚蠢开个脱，哈哈。）

其他的几个回调函数也类似。

#### 小问题

还有ScriptableObject类最好是单独写在一个文件中，文件名和类名相同，不然会常常出现下面的报错（毕竟和MonoBehavior有些关系）。

<font color=red>PS：这个问题还挺奇怪的，我试了很多次，即使创建一个相同类名的文件，即使文件是空的（记得删掉里面的MonoBehavior类），也不会出现这个Missing错误。</font>

![missing_warning](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/_posts/Image/2018-07-04-missing_warning.png)

好的，ScriptableObject的简单使用介绍就到这里，更深入的使用可能还要等我在生产环境实验之后再做打算。

如果文中有错误或者遗漏，欢迎联系指正。
