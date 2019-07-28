---
layout:     post
title:      "Geometry Meadow 02: Terrain And Wind"
subtitle:   "几何着色器花海 02：地形和风"
date:       2019-07-23 20:13:00
author:     "Xjoshua"
header-img: "img/in-post/1907/Meadow02_Title.jpg"
catalog: 	true
tags:
  - Shader
  - Unity
  - Geometry Shader
---

书接上文，之前绘制好了草和花的几何体，基于预设的`Plane`不能控制点的位置和密度，所以我们需要通过代码生成草的顶点。

## 地形绘制 & 密度控制

地形绘制主要参考了陈嘉栋的文章（参考资料3），通过读取高度图，创建Mesh。密度控制也类似，读取另一张NoiseMap，根据值来控制生成顶点的数量，少于一定数量则不生成。

```csharp

public void CreateGrass()
{
    GameObject grassField = new GameObject("GrassMeadow");
    MeshFilter mf = grassField.AddComponent<MeshFilter>();
    Mesh mesh = new Mesh();
    MeshRenderer mr = grassField.AddComponent<MeshRenderer>();
    mr.sharedMaterial = GrassMat;
    List<int> indices = new List<int>();
    List<Vector3> verts = new List<Vector3>();
    int p = 0;
    for (int i = 0; i < this.TerrainSize; i++)
    {
        for (int j = 0; j < this.TerrainSize; j++)
        {
            float density = HeightMap.GetPixel(i * 10, j * 10).r;
            if (density <= 0.3f) continue;

            int mount = (int)(UnityEngine.Random.Range(0, density) * 8) - 1;
            if (mount <= 0) continue;
            for (int z = 0; z < Mount; z++)
            {
                verts.Add(new Vector3(i + Random.Range(-1f, 1f), 
                    HeightMap.GetPixel(i * 5, j * 5).grayscale * HeightScale * baseHeight, j + Random.Range(-1f, 1f)));
                indices.Add(p);
                p++;
            }
        }
    }

    Vector2[] uvs = new Vector2[verts.Count];

    for (var i = 0; i < uvs.Length; i++)
    {
        uvs[i] = new Vector2(verts[i].x / TerrainSize, verts[i].z / TerrainSize);
    }

    mesh.vertices = verts.ToArray();
    mesh.uv = uvs;
    mesh.SetIndices(indices.GetRange(0, verts.Count).ToArray(), MeshTopology.Points, 0);
    mf.mesh = mesh;
}
```

![地形和密度控制](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-04.jpg)

![加入花](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-05.jpg)

## 风
>「狂风，听我号令」 —— 驭风者奥拉基尔

当然静态的草甸缺乏生机，加上风做些顶点动画就好很多。在几何着色器中对顶点进行操作也很方便，稍微麻烦的是几何着色器中不能使用tex2D函数，要用tex2DLod函数。

为了方便对风进行控制，我们需要一个风场图，需要不同的风的时候对不同的风场图进行采样就好了。

![不同的风场图](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-WindMap.jpg)

``` Csharp
[maxvertexcount(24)]
void geom(point vertIn p[1], inout TriangleStream<geomOut> triStream)
{
    // 采样风场图
    float2 wind = (-tex2Dlod(_WindTex, float4(p[0].uv.x + _Time.x * 2 , p[0].uv.y, 0, 0)) + fixed2(0.5, 0.5)) * 8;

    float3 windVec =  float3(wind.x, 0, wind.y);// * sinW;

    ...

    // 校正后的方向
    float3 dir = normalize((height * up + windVec * _WindScale + randomDir * _RandomDirScale);

    // 随机+风影响后的顶点
    float3 Pt = p[0].vertex + height * dir;

    ... 
}
```

不同的风场图效果：

![风场图01](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-Wind01.gif)

![风场图02](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-Wind02.gif)

调下颜色伪装下麦浪也是可以的：

![麦浪](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-Wind02-1.gif)

## 小结
这篇就到这里，加上地形和风之后草甸的基本形态已经有了，下篇讲交互和阴影。

#### 关联笔记
1. [几何着色器花海 01：绘制花](https://xjoshua.github.io/2019/07/17/Geometry-Meadow-01/) 
2. [几何着色器花海 03：交互和阴影](https://xjoshua.github.io/2019/07/27/Geometry-Meadow-03/)

##### 参考资料：

> 1. ["Grass Shader by roystan"](https://roystan.net/articles/grass-shader.html)
> 2. ["Catlike Coding Rendering-7: Shadow"](https://catlikecoding.com/unity/tutorials/rendering/part-7/)
> 3. ["利用GPU实现无尽草地的实时渲染 by 陈嘉栋"](https://www.zhihu.com/search?type=content&q=%E8%8D%89%E5%9C%B0%E6%B8%B2%E6%9F%93)
> 4. ["基于几何图元着色器的花海 by 破晓"](https://zhuanlan.zhihu.com/p/38580338) 
> 5. ["塞尔达草地的知乎回答 by 凯丁"](https://www.zhihu.com/question/271474165/answer/361856323)
> 6. ["移动端草生成小结 by MaYidong"](http://ma-yidong.com/2017/10/15/realtime-grass-rendering-on-mobile-platform-%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AE%9E%E6%97%B6%E8%8D%89%E5%9C%B0%E6%B8%B2%E6%9F%93/) 

