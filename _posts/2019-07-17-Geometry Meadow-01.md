---
layout:     post
title:      "Geometry Meadow 01: Draw Flowers"
subtitle:   "几何着色器花海 01：绘制花"
date:       2019-07-17 21:46:00
author:     "Xjoshua"
header-img: "img/in-post/1907/Meadow01_Title.jpg"
catalog: 	true
tags:
  - Shader
  - Unity
  - Geometry Shader
---

最近在ArtStation看到这样一张图，感觉很不错：

![https://www.artstation.com/artwork/aRdBw9](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/BaseImage.jpg)

正好看到了一些关于几何着色器生成草地的文章，所以就想能不能用几何着色器来做个花海（Meadow：野花草甸）。

想了一下，仅仅还原这张静态图肯定不够的，生成花海主要有以下内容：
- 绘制草
- 绘制花
- 随机
- 地形
- 密度控制
- 风
- 与物体交互
- 阴影
- LOD（划掉）

## 绘制草
>「要有草」

仔细观察参考图，花主要有三个部分：花朵，茎，草。茎可以和草同种做法，控制叶片的宽度就好。需要绘制的两部分，花朵和草的叶片，生成花的时候把花朵放草的顶上。虽然最终要做的是花海，还是先从草开始。

绘制草的方法也有很多种，考虑到参考图的花也是低多边形的，参考塞尔达中的草，在风格化的场景中，用四个点/两个三角面已经效果不错了，（而且这样就不用在单根草上通过减少顶点来做LOD）：

![荒野之息中的近景草，烧过变黑很明显看出是两个三角形的Quad](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/BOTW01.png)

当然还要通过摄像机射线来得到 billboard 效果：

```csharp
...
#pragma geometry geom
...

geomOut GeneratePos(float3 pos, float2 uv)
{
    geomOut o;
    o.pos = UnityObjectToClipPos(pos);
    o.uv = uv;
    return o;
}

// maxvertexcount 括号中的参数控制了几何着色器中添加的最大顶点数
[maxvertexcount(6)]
void geom(point vertIn p[1], inout TriangleStream<geomOut> triStream)
{
    // 顶点
    float3 Pt = p[0].vertex + height;

    // 中间点
    float3 mid = (Pt - p[0].vertex) * 0.4 + p[0].vertex;

    // 拿到摄像机的观察向量
    float3 look = _WorldSpaceCameraPos - mul(unity_ObjectToWorld, p[0].vertex);

    // 求两边的点
    float3 crossDir = normalize(cross(Pt - p[0].vertex, look));
    float3 pos1 = mid + crossDir * _GrassSize;// + windVec * 0.35;
    float3 pos2 = mid - crossDir * _GrassSize;// + windVec * 0.35;

    // UV
    float2 grassBottom = float2(0, 0);
    float2 grassMidUv = float2(0, 0.4);
    float2 grassTopUv = float2(0, 0.79);

    // 添加三角形
    // 草 下部三角形
    triStream.Append(GeneratePos(p[0].vertex, grassBottom));
    triStream.Append(GeneratePos(pos1, grassMidUv));
    triStream.Append(GeneratePos(pos2, grassMidUv));
    triStream.RestartStrip();

    // 草 上部三角形
    triStream.Append(GeneratePos(pos1, grassMidUv));
    triStream.Append(GeneratePos(pos2, grassMidUv));
    triStream.Append(GeneratePos(Pt, grassTopUv));
    triStream.RestartStrip();
}
```

![基础的草绘制](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-01.jpg)

## 绘制花
>「要花，不要战争」

参考图看起来是截面是正方形的纺锤形，本着能节省一点算一点的思想（主要是懒），我在3D软件中尝试了一下，中间三角形，上下各一个顶点，效果也凑合（省了十个面，四舍五入就是一个亿啊...）。

```csharp

// 生成随机数的函数
float rand(float seed)
{
    return frac(sin(seed)*10000.0);
}

// 需要修改最大顶点数量
[maxvertexcount(24)]
void geom(point vertIn p[1], inout TriangleStream<geomOut> triStream)
{
    ... // 计算草的点
    // 花的点 4 个 + 草的顶点
    float3 Pf0 = Pt + _FlowerSize * up; // 三角面的中间点
    float3 Pft = Pt + 1.5 * _FlowerSize * up; // 花的顶点
    // 计算一个随机方向
    float3 flowerRandDir = normalize(float3(1, 0, 0));
    float3 Pf1 = Pf0 + flowerRandDir * _FlowerSize;
    float3 Pf01 = Pf0 - 0.5 * flowerRandDir * _FlowerSize;
    float3 Pf2 = Pf01 + flowerRandDir * 0.865 * _FlowerSize; // 0.865 近似 sqrt(3) * 0.5
    float3 Pf3 = Pf01 - flowerRandDir * 0.865 * _FlowerSize;

    ...

    float2 flowerBotUv = float2(0, 0.81);
    float2 flowerMidUpUv = float2(0, 0.91);
    float2 flowerMidDownUv = float2(0, 0.89);
    float2 flowerTopUv = float2(0, 0.99);

    // 添加三角形
    ...
    // 几何花 下部
    triStream.Append(GeneratePos(Pt, flowerBotUv));
    triStream.Append(GeneratePos(Pf1, flowerMidDownUv));
    triStream.Append(GeneratePos(Pf2, flowerMidDownUv));
    triStream.RestartStrip();

    triStream.Append(GeneratePos(Pt, flowerBotUv));
    triStream.Append(GeneratePos(Pf2, flowerMidDownUv));
    triStream.Append(GeneratePos(Pf3, flowerMidDownUv));
    triStream.RestartStrip();

    triStream.Append(GeneratePos(Pt, flowerBotUv));
    triStream.Append(GeneratePos(Pf3, flowerMidDownUv));
    triStream.Append(GeneratePos(Pf1, flowerMidDownUv));
    triStream.RestartStrip();

    // 花 上部
    triStream.Append(GeneratePos(Pf1, flowerMidUpUv));
    triStream.Append(GeneratePos(Pf2, flowerMidUpUv));
    triStream.Append(GeneratePos(Pft, flowerTopUv));
    triStream.RestartStrip();

    triStream.Append(GeneratePos(Pf2, flowerMidUpUv));
    triStream.Append(GeneratePos(Pf3, flowerMidUpUv));
    triStream.Append(GeneratePos(Pft, flowerTopUv));
    triStream.RestartStrip();

    triStream.Append(GeneratePos(Pf3, flowerMidUpUv));
    triStream.Append(GeneratePos(Pf1, flowerMidUpUv));
    triStream.Append(GeneratePos(Pft, flowerTopUv));
    triStream.RestartStrip();
}

```

当然花的颜色和草不同，准备一张采样颜色的贴图，在片元着色器中根据顶点UV采样颜色。

![颜色采样贴图](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-MainTex.png)

结果：

![几何花绘制](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-02.jpg)

## 随机
>「世上没有两片完全相同的树叶」 —— 莱布尼茨

草和花都有了，但是这也太呆了，排列整齐，高度一致。需要加点扰动，着色器中并没有现成的随机函数，看了一些资料，有下面的函数：

``` Csharp

float rand(float seed)
{
    return frac(sin(seed)*10000.0);
}

// 见参考资料一
float rand(float3 seed)
{
    return frac(sin(dot(seed.xyz, float3(12.9898, 78.233, 53.539))) * 43758.5453);
}

```

加入一张噪音图，随机顶点位置，高度，生长方向，花的大小，方向：

``` Csharp

...
vertIn vert (vertIn v)
{
    v.vertex = v.vertex + float4(rand(v.vertex.xyz) - 0.5, 0, rand(v.vertex.xyz * 2) - 0.5, 0);
    return v;
}

[maxvertexcount(24)]
void geom(point vertIn p[1], inout TriangleStream<geomOut> triStream)
{
    // 采样噪声图备用
    float4 sampleNoise = tex2Dlod(_NoiseTex, float4(p[0].uv.x, p[0].uv.y, 0, 0));

    // 随机生长方向
    float3 randomDir = float3(rand(p[0].vertex.x + sampleNoise.x) - 0.5, 0, rand(p[0].vertex.z + sampleNoise.y) - 0.5);

    // 采样噪音图 随机顶点高度 
    float height = _BaseHeight + 5 * sampleNoise.y + (rand(p[0].vertex.xyz) - 0.5) * 5 * _RandomHeightScale;

    // 顶点
    float3 Pt = p[0].vertex + height * normalize(height * up);

    ... // 草顶点计算

    // 花大小随机
    float flowerSize = _FlowerSize + rand(p[0].vertex.x * 3 + p[0].vertex.z) * 0.1;

    // 花的点 4个 + 草的顶点
    float3 Pf0 = Pt + flowerSize * dir;
    float3 Pft = Pt + 1.5 * flowerSize * dir;

    float3 flowerRandDir = normalize(float3(rand(p[0].vertex.x * 2 + sampleNoise.x), 0, rand(p[0].vertex.y * 2 + sampleNoise.y)));
    float3 dir2 = normalize(cross(dir, flowerRandDir));
    float3 Pf1 = Pf0 + dir2 * flowerSize;
    float3 Pf01 = Pf0 - 0.5 * dir2 * flowerSize;
    float3 Pf2 = Pf01 + flowerRandDir * 0.865 * flowerSize; // 0.865 近似 sqrt(3) * 0.5
    float3 Pf3 = Pf01 - flowerRandDir * 0.865 * flowerSize;

    // UV
    float2 grassBottom = float2(0, 0);
    float2 grassMidUv = float2(randomDir.x, 0.4);
    float2 grassTopUv = float2(randomDir.x, 0.79);
    // 随机颜色
    float2 flowerBotUv = float2(randomDir.x, 0.81);
    float2 flowerMidUpUv = float2(randomDir.x, 0.91);
    float2 flowerMidDownUv = float2(randomDir.x, 0.89);
    float2 flowerTopUv = float2(randomDir.x, 0.99);

    ... // 加入点
}

...

```

![随机化](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1907/Grass-03.jpg)

## 小结
这篇就到这里，主要是顶点和三角面的生成，下篇讲地形生成和风影响的顶点动画。

#### 关联笔记
1. [几何着色器花海 02：地形和风](https://xjoshua.github.io/2019/07/23/Geometry-Meadow-02/) 
2. [几何着色器花海 03：交互和阴影](https://xjoshua.github.io/2019/07/27/Geometry-Meadow-03/)

##### 参考资料：

> 1. ["Grass Shader by roystan"](https://roystan.net/articles/grass-shader.html)
> 2. ["Catlike Coding Rendering-7: Shadow"](https://catlikecoding.com/unity/tutorials/rendering/part-7/)
> 3. ["利用GPU实现无尽草地的实时渲染 by 陈嘉栋"](https://www.zhihu.com/search?type=content&q=%E8%8D%89%E5%9C%B0%E6%B8%B2%E6%9F%93)
> 4. ["基于几何图元着色器的花海 by 破晓"](https://zhuanlan.zhihu.com/p/38580338) 
> 5. ["塞尔达草地的知乎回答 by 凯丁"](https://www.zhihu.com/question/271474165/answer/361856323)
> 6. ["移动端草生成小结 by MaYidong"](http://ma-yidong.com/2017/10/15/realtime-grass-rendering-on-mobile-platform-%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AE%9E%E6%97%B6%E8%8D%89%E5%9C%B0%E6%B8%B2%E6%9F%93/) 

