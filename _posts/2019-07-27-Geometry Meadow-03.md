---
layout:     post
title:      "Geometry Meadow 03: Interaction And Shadow"
subtitle:   "几何着色器花海 03：交互和阴影"
date:       2019-07-27 20:26:00
author:     "Xjoshua"
header-img: "img/1907/Meadow03_Title.jpg"
catalog: 	true
tags:
  - Shader
  - Unity
  - Geometry Shader
---

## 交互

观察荒野之息可以看到，在林克走在草地上的时候，几何草向四周倒下：

![荒野之息中的草交互](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/BOTW02.gif)

那么只需要向shader传入物体当前的位置，和影响范围内的点，计算出倒伏的方向：

```csharp

// 传入交互的物体的位置
public void SetTargetPos()
{
    Vector3 pos = TargetBall.transform.position;

   // Vector2 uvPos = new Vector2(pos.x / 200f, pos.y / 200f);

    GrassMat.SetVector("_TargetPos", pos);
    FlowerMat.SetVector("_TargetPos", pos);
}

```

```csharp

[maxvertexcount(24)]
void geom(point vertIn p[1], inout TriangleStream<geomOut> triStream)
{
    ...

    // 和移动物体交互
    // 计算和物体的距离
    float dis = distance(_TargetPos, p[0].vertex);
    //float fall = step(dis, _FallRange);
    float fall = smoothstep(_FallRange * 0.5, _FallRange, dis);

    // 修改校正后的方向
    float3 dir = normalize((height * up + windVec * _WindScale + randomDir * _RandomDirScale) * fall + (p[0].vertex - _TargetPos) * float3(1,0,1) * (1 - fall));
		
    ...
}
```

![草交互](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-Move1.gif)

## 阴影

阴影有两部分，接受阴影和投影。需要在Shader中分成两个Pass来做。

第一个Pass设置 `LigntMode` 为 `ForwardBase` ，在计算顶点的时候加入`o._ShadowCoord = ComputeScreenPos(o.pos);`，计算点在屏幕空间的位置，这样可以在片元着色器中使用 `SHADOW_ATTENUATION` 来计算阴影强度。

另一个Pass设置 `LightMode` 为 `ShadowCaster` ，在片元着色器中只执行 `SHADOW_CASTER_FRAGMENT(i)` 。

原理可以在 Catlike Coding 关于渲染阴影的章节找到详细内容（参考资料2）。

中间有一个情况是绘制的物体接受阴影的时候可能会接受到自己的阴影，导致物体表面上出现条纹阴影，这个问题可以通过对阴影做一个偏移来规避。具体是在计算顶点时，在 `ForwardBase Pass` 中计算 `ComputeScreenPos` ，而在 `ShadowCaster Pass` 中对顶点执行 `UnityApplyLinearShadowBias` ，然后在场景光中设置动态阴影的 `Bias` 设置为合适的值（在我的项目中设置到最大值2才消除）。

![阴影错误](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-Shadow01.jpg)

设置偏移会带来阴影的位置不正确（放置在平面上的物体会比较明显），所以这个值的设置还是要具体情况来看。

具体代码可以看下Github中的项目，就不在这里贴出来了。

![阴影效果](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-Shadow02.jpg)

## 小结
至此几何着色器草甸篇章就结束了，也没有什么新的内容，主要是对学习几何着色器过程的一个记录。当然据说几何着色器的性能并不好，不能在移动端使用，不过这就是另外的问题了。

![阴影效果](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-05.gif)

![最后效果](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-06.jpg)

["Github地址"](https://roystan.net/articles/grass-shader.html)

#### 关联笔记
1. [几何着色器花海 01：绘制花](https://xjoshua.github.io/2019/07/17/Geometry_Meadow_01/) 
2. [几何着色器花海 02：地形和风](https://xjoshua.github.io/2019/07/23/Geometry_Meadow_02/) 

##### 参考资料：

> 1. ["Grass Shader by roystan"](https://roystan.net/articles/grass-shader.html)
> 2. ["Catlike Coding Rendering-7: Shadow"](https://catlikecoding.com/unity/tutorials/rendering/part-7/)
> 3. ["利用GPU实现无尽草地的实时渲染 by 陈嘉栋"](https://www.zhihu.com/search?type=content&q=%E8%8D%89%E5%9C%B0%E6%B8%B2%E6%9F%93)
> 4. ["基于几何图元着色器的花海 by 破晓"](https://zhuanlan.zhihu.com/p/38580338) 
> 5. ["塞尔达草地的知乎回答 by 凯丁"](https://www.zhihu.com/question/271474165/answer/361856323)
> 6. ["移动端草生成小结 by MaYidong"](http://ma-yidong.com/2017/10/15/realtime-grass-rendering-on-mobile-platform-%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AE%9E%E6%97%B6%E8%8D%89%E5%9C%B0%E6%B8%B2%E6%9F%93/) 

