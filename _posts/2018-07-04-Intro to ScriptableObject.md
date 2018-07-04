---
title: Intro to Scriptable Object
description: A simple tutorial for using ScritableObject in Unity.
categories:
 - tutorial
 - Unity
tags:
---

## Intro to Scriptable Object
#### ScriptObject介绍

在做本地化的时候需要在Editor模式下测试不同的语言，现在项目中的本地化控制是通过读取系统语言（游戏体量小，并没有在设置中设置语言的选项），在游戏中通过Editor编辑器扩展直接修改也比较简单，但是每次都需要设置一下，有点麻烦。所以想起了Unity官方曾经推荐过的ScriptableObject：

>A class you can derive from if you want to create objects that don't need to be attached to game objects. This is most useful for assets which are only meant to store data.
>如果你想创建不需要附着在游戏物体上的项目，你可以使用这个类。常常用来构造只为存储数据的资产。

概括一下，有下面几个优点：
-不需要附加到GameObject上
-可以轻松序列化存储数据
-在inspect中可以直观的检查和修改
	
像我这个本地化的例子，把文件放到Editor文件夹中也不会影响到游戏打包。完美。

不过实际用起来还是和MonoBehavior有一点区别。通常MonoBehavior脚本可以直接在GameObject的Inspector脚本组件界面中修改，而ScriptableObject是在脚本生成的.asset实例文件上修改。
.asset实例文件
![asset实例文件]({{site.baseurl}}/_posts/asset实例文件.png)

下面就是使用ScriptableObject的代码：
```
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

注意，和Monobehavior类似，ScriptableObject也有生命周期方法，不过只有四个：

Awake	This function is called when the ScriptableObject script is started.
OnDestroy	This function is called when the scriptable object will be destroyed.
OnDisable	This function is called when the scriptable object goes out of scope.
OnEnable	This function is called when the object is loaded.

同样，因为和MonoBehaviour有些不同（或者相同），这里的几个方法的调用时机会有一些概念上的问题。MonoBehaviour的Awake是在脚本（组件）创建的时候调用的，同样ScriptableObject的Awake也是在脚本实例创建出来的时候调用的。（一开始的时候我还以为是在类声明的时候调用的。。。ORZ）其他的几个回调函数也类似。

还有ScriptableObject类最好是单独写在一个文件中，文件名和类名相同，不然会常常出现下面的报错（毕竟和MonoBehavior有些关系）。
PS：这个问题还挺奇怪的，我试了很多次，即使创建一个相同类名的文件，即使文件是空的（记得删掉里面的MonoBehavior类），也不会出现这个Missing错误。
missing_warning
![missing_warning]({{site.baseurl}}/_posts/missing warning.png)

