---
layout:     post
title:      "Jigsaw Piece Generate"
subtitle:   "随机拼图生成"
date:       2019-09-22 20:00:00
author:     "Xjoshua"
header-img: "img/in-post/1909/step03.jpg"
catalog: 	true
tags:
  - Shader
  - Unity
---


最近做了一个拼图游戏，配合简化的代码，把拼图生成的思路整理一下。

## 准备

![参考图](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/jigsawSample.jpg)

上面是三个参考项目的拼图界面，前两个是ios上的，后一个是PC上的微软拼图。可以看到前两个拼图块边界样式上变化比较小，后一个边界类型更丰富。立体感上前两个也差的很远。游戏性上来说，丰富的边界对拼图是很有帮助的，而光影塑造的立体感则让人觉得和实体拼图更接近一些。

当然不可能让美术把拼图每一小块都切出来，于是用shader来完成拆分图片，随机边界，添加光影效果的任务。

（下面用的图片来自 [https://unsplash.com/t/travel](https://unsplash.com/t/travel) ，一个免费的（do whatever you want）图片分享网站。）

## 拆分图片 

拆分图片是相当简单的一部分，稍微麻烦的一点的是需要把根据块数得到的块UV传到到shader代码中（其实也很简单）。

根据块的ID，传入块左下角的坐标，和块的长宽（为了方便计算，块和整个拼图都是正方形的，长宽的数量一样）。

```csharp
	Vector4 uv = new Vector4(i % ColCount * uvx, i / ColCount * uvx, 1f / ColCount, 1f / ColCount);
	PieceItems[i].image.rectTransform.sizeDelta = new Vector2(imgSize, imgSize);
	PieceItems[i].image.rectTransform.anchoredPosition = 
	    new Vector2(- oriPos + i % ColCount * (imgSize + 20f), - oriPos + i / ColCount * (imgSize + 20f));
	PieceItems[i].Init(uv);
```

在块初始化的时候，把对应的长宽传入的网格顶点中：

```csharp
    public void Init(Vector4 uv)
    {
        pieceUV = uv;
    }

    /// <summary>
    /// 把需要使用的参数传入网格顶点中
    /// </summary>
    public override void ModifyMesh(VertexHelper vh)
    {
        UIVertex vert = new UIVertex();
        for (int i = 0; i < vh.currentVertCount; i++)
        {
            vh.PopulateUIVertex(ref vert, i);
            vert.uv1 = new Vector2(pieceUV.x, pieceUV.y);
            vert.uv2 = new Vector2(pieceUV.z, pieceUV.w);
            vh.SetUIVertex(vert, i);
        }
    }
```

要使用网格顶点的额外数据（这里是第一套uv和第二套uv），需要在根 Canvas 的 Additional Shader Channels 对应的数据打开，才能正确传入 Shader 中（这里是 TexCoord1 和 TexCoord2）.

![设置canvas](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/unity01.png)

Shader：

接受传入的数据：
```csharp
	struct appdata
	{
	    float4 vertex: POSITION;
	    float2 uv: TEXCOORD0;
	    float2 uv1: TEXCOORD1;
	    float2 uv2: TEXCOORD2;
	    float4 color: COLOR;
	};
```

根据传入的数据计算在底图上实际的uv：
```csharp
	v2f vert(appdata v)
	{
	    v2f o;
	    o.vertex = UnityObjectToClipPos(v.vertex);
	    o.uv = float2(v.uv1.x + v.uv.x * v.uv2.x, v.uv1.y + v.uv.y * v.uv2.y);
	    return o;
	}
	
	fixed4 frag(v2f i) : COLOR
	{
	    fixed4 c = tex2D(_MainTex, i.uv);
	}
```

然后我们就能得到切分成了9份的原始图片。

![step01](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/step01.jpg)

## 随机边界
边界一开始的想法是用曲线来做，比如说样条曲线，随机四个或者六个控制点的位置，来控制整个曲线。研究了一会 NURBS 曲线（非均匀有理B样条，说出来就感觉很厉害）之后，还是放弃了这个方案。并不是说这个方案不行（这个链接中有使用曲线来生成拼图的边界：[link](http://bl.ocks.org/nevernormal1/f808cffb897c63a8dd4e)）。考虑到还需要判断点是否在曲线内部，以及后续的光影，还是把这个工作交给了美术来做。

三种类型的边界：边缘的平边，凸出的和凹陷的。这里只做了向上的方向，其他方向在shader中采样的时候乘一个旋转矩阵就好了。
	PS1：后面两张图的上下是一样的，两张图看下效果就行了，需要的话也可以扩充成5*5或者更多
	PS2：因为通道的原因是红色，具体后面会说

![res01](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/res01.jpg)

然后在shader中采样边界，然后剔除对应的黑色区域，就可以得到我们需要的拼图。同样，需要把四个边界的类型和样式都传入到shader中，因为已经用了两个uv了，这里把数据传到 color 中，类型只有3种，所以可以把类型和样式结合在一样。

```csharp
	Vector4 temp = (piecedgeType * 100f + pieceThemeType) / 255;
	vert.color = new Color(temp.x, temp.y, temp.z, temp.w);
```

注意这里要考虑到颜色的粒度是 1/255，之前用 piecedgeType * 0.1f + pieceThemeType * 0.001f 在shader中数据就丢失了。（PS：所以也许传到 color 中并不是一个好方法，限制了边界类型的数量，但是现阶段是够用的）

传递数据到片元着色器中：
```csharp
    o.edgeType = v.color;
```

对边界蒙版采样：
```csharp
fixed4 GetPieceShape(float edgeType, float edgeAlpha, float2 direction)
{
    int type = (edgeType + 0.01) * 2;
    switch (type)
    {
        case 0:
            return tex2D(_MaskA, direction);
        case 1:
            return tex2D(_MaskB, direction);
        case 2:
            return tex2D(_MaskC, direction);
    }
    return fixed4(0, 0, 0, 0);
}
```

在片元着色器中处理数据：
```csharp
    fixed4 c = tex2D(_MainTex, i.uv);

    float2 uv_Top = i.uv0;
    
    float alpha = c.a;
    fixed4 col = fixed4(0, 0, 0, 0);
    float shadow;

    col = GetPieceShape(i.edgeType.r, alpha, uv_Top);
    alpha *= col.r;
    …

    c.a = alpha;
    return c;
```

如果是左边，旋转矩阵变换之后再进行同样的操作：
```csharp
	float2x2 rotationMatrix = float2x2(0, -1, 1, 0);
	float2 uv_Left = mul(uv_Top.xy, rotationMatrix);
	…
```

由于边缘蒙版的原因，切分后的图片会被挡住一部分，于是原来的切分需要做一些修改，每个小块都需要显示更大的区域：
```csharp
	Vector4 uv = new Vector4(i % ColCount * uvx - uvx / 3f,
	    i / ColCount * uvx - uvx / 3f, uvx * 5f / 3, uvx * 5f / 3);
	PieceItems[i].image.rectTransform.sizeDelta = new Vector2(imgSize * 5f / 3, imgSize * 5f / 3);
```

随机得到边的类型和样式：
```csharp
	Vector4Int edge = GetRandomEdge(col, row);
	Vector4Int edgeTheme = GetEdgeTheme(col, row);
	PiecesEdges[i] = edge;
	PiecesEdgeThemes[i] = edgeTheme;
	
	PieceItems[i].Init(uv, edge, edgeTheme, i);
```

得到生成的拼图：
![step02](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/step02.jpg)

## 光影
现在已经得到基本的形状了，如果加上简单的描边，就和前两个参考的效果类似。加上光影其实也比较简单，主要是在美术上需要有一样工作量，把斜面的光照效果画到边缘图的gb通道上，然后把光照添加到边缘即可。

边缘图的G通道，观察边缘处的光照效果：
![res02](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/res02-r.jpg)

在获取蒙版颜色之后再加上光影的颜色（0.8可以根据需要调整大小）：
```csharp
	shadow = (col.b - 0.5) * 0.8;
	c += fixed4(shadow,shadow,shadow,0);
```

最终的效果：
![step03](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1909/step03.jpg)

当然这样做的问题是无论块如果旋转，光照都是一样的。。。

另一种方法是把法线画到边缘图的gb通道上（z方向其实不用？），然后对光照方向做一个计算，这样在块旋转之后也能得到正确的光影效果。这里就不展示具体的写法了（主要是懒得画法线图。。。）。

## 结束
还有一些比如块的阴影，拼接完成后的闪光效果，都比较简单，这里就不一一说明了。

##### 参考资料：

> 1. ["使用的图片来源"](https://unsplash.com/photos/DGOI_BlPdC4)
> 2. ["DYNAMIC JIGSAW PUZZLE GENERATION：似乎修改点有点问题"](http://stephenhadadream.com/?p=865)
> 3. ["随机点然后用的SVG的库，不太有用"](http://bl.ocks.org/nevernormal1/f808cffb897c63a8dd4e)


