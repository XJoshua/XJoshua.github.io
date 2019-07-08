---
layout:     post
title:      "Paintbook and SVG"
subtitle:   "Unity填色游戏和SVG的使用"
date:       2019-06-08 21:00:00
author:     "Xjoshua"
header-img: "img/post-bg-default.jpg"
catalog: 	true
tags:
  - Unity
  - Tuturial
  - Demo
  - Python
  - SVG
---

## 源起

前些天组长喊我研究一下填色游戏，就是那种点击线稿图填充颜色的游戏。如果能把底图按照线稿区域划分成单独的区域，当然没有任何难度。但这当然不行，没有困难需要创造困难啊（主要是没有空闲的美术来做图...）。

在研究竞品的时候，发现他们的图可以放很大也几乎没有模糊，猜测可能是矢量图。搜索一下，发现有一个付费的SVG插件：SVG Importer和preview中的官方插件：Vector Graphics，反正是研究，就先用了官方插件。["官方介绍"](https://forum.unity.com/threads/vector-graphics-preview-package.529845/)

如果是简单的UI，使用SVG还是很不错的，体积小，放大不会模糊，缺点是顶点和面数更多。

![Vector Graphics-preview27：在我用的时候竟然从26升级到27了...](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1906/190608-vectorGraphics.jpg)

既然解决了在Unity中使用SVG的问题，然后路线就是如何把图转化成区域，用来点击和填充颜色。最好能够方便的导入Unity中直接使用，生成颜色配置表之类的。于是想到了老本行软件：Adobe Illustrator（后文称为AI），如果能在AI中分好区域，存储成SVG，然后读取到Unity中，写一个后处理，应该就能解决了这个问题吧。

## 矢量图

在网上找了一张颜色简单，适合填色的图：
![侵删](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1906/190608-baseImage.jpg)

导入进AI，选择「图像描摹-低保真度图片」，然后选择「扩展」，就会发现图片已经变成了矢量图块，可以单独选中，也可以方便的加上描边。发现有些区域有重叠的部分，可以在「路径查找器」中使用「修边」功能。当然还有一些地方有点瑕疵，比如区域太小，之后可能很难在游戏中选中之类的，还是需要稍微在Ai中处理一下。
![处理之后可以看出树的渐变颜色变成了几种颜色](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1906/190608-baseImage-ai.jpg)

## 导入Unity

导入Unity后发现一个问题，这图是一整块啊，并没有把SVG中的图层，编组或者形状信息保留下来...研究了一下，发现也有人问过了这个问题，官网说之后可能会加这个功能，反正现在还没有。找了一下其他的方案，本来想在资源导入的后处理中把SVG按照形状拆开的，但是怎么都不能按我想要的存成「预制体」-「形状A-形状B-...」的结构，还能保存下VectorSprite。
nickfourtimes说解决了这个问题，也提供了代码，然而运行报错（也可能是哪里没有调整好...）：[nickfourtimes:https://forum.unity.com/threads/vector-graphics-preview-package.529845/page-5#post-3621448](https://forum.unity.com/threads/vector-graphics-preview-package.529845/page-5#post-3621448)

另一个看似可行的方案是直接保存成整块，然后运行时加载成单独的形状，其实这个方案比前面一个更好些，也确实可以在运行时生成并分割成小块，但是后面的按形状生成点击区域却完成不了了...
[rab:https://forum.unity.com/threads/vector-graphics-preview-package.529845/page-5#post-3621448](https://forum.unity.com/threads/vector-graphics-preview-package.529845/page-5#post-3621448)

看起来在Unity中处理的尝试失败了，难道就这样放弃了吗？当然不，既然不能在Unity中处理，那就在外面直接处理生成的SVG文件好了。用VSCode打开看了下SVG文件，发现是用XML存储的信息，那就简单了，直接写一个Python脚本来处理好了。

```python
#!/usr/bin/env python
import copy
import sys
import xml.etree.ElementTree as ET
import json

ET.register_namespace('', "http://www.w3.org/2000/svg")
ET.register_namespace('xlink', "http://www.w3.org/1999/xlink")

tree = ET.parse(sys.argv[1])
root = tree.getroot()
# print(root)
listofgroup = []
listofcolor = []
count = 0
for g in root.findall('{http://www.w3.org/2000/svg}path'):
    # name = g.get('id')
    color = g.get('fill')
    g.set('fill', "#FFFFFF")
    count += 1
    listofgroup.append(count)
    listofcolor.append(color)
print(listofgroup)

# write config json file
json = json.dumps(listofcolor)
with open(sys.argv[1].replace('.svg', '_config'),'w') as f:
    f.write(json)

for counter in range(len(listofgroup)):
    temp = listofgroup[:]
    temp_tree = copy.deepcopy(tree)
    del temp[counter]
    print(temp)
    temp_root = temp_tree.getroot()
    temp_count = 0
    for g in temp_root.findall('{http://www.w3.org/2000/svg}path'):
        # name = g.get('id')
        temp_count += 1
        if temp_count in temp:
            temp_root.remove(g)
    temp_tree.write(sys.argv[1].replace('.svg', '_' + str(listofgroup[counter]) + '.svg'))
```

主要就是按 `path` 生成单独的SVG文件，并把颜色信息单独存成config文件，之后读取配置文件填充颜色。每种不足的是拆分成小文件后体积大了很多，毕竟文件头都是一样的。

## 生成点击区域

拆分成小图导入后，为了保证放入场景的相对位置是一致的（避免还要手动调整位置），把导入设置中的 `Pivot` 设置为 `SVG Origin`。

为了交互，需要生成碰撞区域，勾选上导入设置中的 `Generate Physics Shape`，在 `SVGImporter.cs` 中 `OnImportAsset` 方法的最后加上下面的代码：

![设置导入选项](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1906/190608-importSetting.jpg)

```csharp
if (true)
{
    GeneratePhysicsShape = true;
    Alignment = VectorUtils.Alignment.SVGOrigin;

    var physicsShapes = VectorUtils.TraceNodeHierarchyShapes(sceneInfo.Scene.Root, tessOptions);

    foreach (var vertices in physicsShapes)
    {
        if (rect == Rect.zero)
        {
            rect = VectorUtils.Bounds(vertices);
            VectorUtils.RealignVerticesInBounds(vertices, rect, flip: true);
        }
        else
        {
            VectorUtils.FlipVerticesInBounds(vertices, rect);

            // The provided rect should normally contain the whole geometry, but since VectorUtils.SceneNodeBounds doesn't
            // take the strokes into account, some triangles may appear outside the rect. We clamp the vertices as a workaround for now.
            VectorUtils.ClampVerticesInBounds(vertices, rect);
        }
    }

    sprite.OverridePhysicsShape(physicsShapes);
}
```

[代码来自论坛的 Seb-1814：https://forum.unity.com/threads/vector-graphics-preview-package.529845/page-9](https://forum.unity.com/threads/vector-graphics-preview-package.529845/page-9)   

下面的图可以看出通过代码在后处理中生成的collider更加接近我们的需求（因为都是简单的闭合形状）。
![论坛的图](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1906/190608-generateCollider.png)

## Demo

剩下的就基本没有什么问题了，基本上写两个交互脚本就是一个简单的Demo。["Github地址"](https://github.com/XJoshua/PaintbookDemo-Unity) 

![Demo](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1906/190608-Paintbook.gif)


## 小结

总结一下几点：
* SVG有内存占用小、清晰度高的优点，在顶点和面影响不大的情况下还是可以使用的，就看什么时候官方把插件完善了。
* 使用Python脚本批处理美术资产，可以节约大量美术师的时间。
* 资源导入的后处理还是有很多可能的，有空可以再研究一下。

虽然感觉流程优化的还可以（基本上美术那边只需要稍稍处理下图就行，当然SVG插件本身还是preview，如果做的话出现什么其他的问题也说不定），然后组长后来想了一下还是觉得很麻烦（...)，美术抽不开身处理图，项目就搁置了。

本身研究的过程还是蛮有趣的，所以就整理出来了。

如果文中有错误或者遗漏，欢迎联系指正。
