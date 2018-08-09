---
layout:     post
title:      "Shader102: Stream Lighting Effect"
subtitle:   "流光效果"
date:       2018-05-22 20:00:00
author:     "Xjoshua"
header-img: "img/post-bg-default.jpg"
catalog: 	true
tags:
  - Unity
  - Shader
---


## Shader102: 流光效果

有空的时候会学着写一些Shader，正好有个朋友问到Logo的流光，就想试着写了一个。一开始的时候立即想到用数学方法写，写了一半发现效果一直调不好。。。只好上网找找资料，结果发现大部分都是用另外做的一张流光图叠加达到效果，嗯。。。虽然有点偷懒，不过确实简单很多。所以就开心的放弃了写了一半的数学方法，就地25.

首先你要有一张流光图。这个简单，用PS处理一下就好。
![流光图](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/_posts/Image/2018-07-13-StreamLight.png)

#### Shader Code

```csharp
Shader "UI/FlashLight" 
{
	Properties 
	{
		[NoScaleOffset] _MainTex ("Main Texture", 2D) = "white" {}
                _LightTex ("Light Texture", 2D) = "white" {}
                _FlashColor("Light Color",Color) = (1,1,1,1)
                _ScrollSpeed("ScrollValue",Range(-10, 10)) = 2
                _TimeInterval("TimeInterval(not second)", Range(1, 10)) = 5
                _Slope("Slope", Range(0, 5)) = 0.5
	}
	SubShader 
	{
		Tags
                {
                "Queue" = "Transparent"
                "IgnoreProjector" = "True"
                "RenderType" = "Transparent"
                }

		Cull Off
                Lighting Off
                ZWrite Off
                Fog { Mode Off }
                Offset -1, -1
                Blend SrcAlpha OneMinusSrcAlpha 
                AlphaTest Greater 0.1

		Pass
                {
                    CGPROGRAM
                    #pragma vertex vert
                    #pragma fragment frag
                    
                    #include "UnityCG.cginc"

                    struct appdata
                    {
                        float4 vertex : POSITION;
                        float2 uv : TEXCOORD0;
                    };

                    struct v2f
                    {
                        float2 uv : TEXCOORD0;
                        float4 vertex : SV_POSITION;
                        float2 lightuv : TEXCOORD1;
                    };

                    sampler2D _MainTex;
                    float4 _MainTex_ST;

                    sampler2D _LightTex ;
                    float4  _LightTex_ST;

                    half4 _FlashColor ;
                    float _ScrollSpeed;
                    float _TimeInterval;
                    float _Slope;

                    v2f vert (appdata v)
                    {
                        v2f o;
                        o.vertex = UnityObjectToClipPos(v.vertex);
                        o.uv = v.uv;
                        return o;
                    }

                    fixed4 frag (v2f i) : SV_Target
                    {
                        // 采样颜色
                        fixed4 col = tex2D(_MainTex, i.uv);
                        // 缓存贴图的透明度
                        float a = col.a;
                        // 计算流光的移动值
                        fixed ScrollValue = _ScrollSpeed * _Time.y - 2;
                        float2 scrolledUV = i.uv + fixed2(ScrollValue, 0.0f);
                        // 根据需要的 斜率，流光间隔 调整移动值
                        scrolledUV = float2(fmod(scrolledUV.x * _Slope, _TimeInterval), scrolledUV.y);
                        // 采样流光贴图
                        float4 lightCol = tex2D(_LightTex, scrolledUV);
                        // 和主贴图颜色混合
                        float3 finalColor = col.rgb + _FlashColor * lightCol;
                        // 读取缓存的透明度
                        fixed4 finalRGBA = fixed4(finalColor, a);

                        return finalRGBA;
                    }
		ENDCG
		}
	}
	FallBack "Diffuse"
}
```

#### 效果

![Stream Light Effect](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/_posts/Image/2018-07-13-StreamLightEffect.gif)

新学了一个 `fmod` 函数，很有趣。有点类似于「取余」运算，不过可以用浮点数来做(看函数名就是浮点数模运算)，用这个函数可以省很多事。

这个Shader也确实很简单，就不多解释了，基本看看注释就好。

话说回来，写到后来又发现用数学方法做的好处了。用贴图做法多张贴图也还好，但是调整斜率却是用压缩uv来实现的，这就意味着同时光柱的宽度也在变化，压缩过大的时候光柱就变成了一条线，效果很糟。不过应该也是可以解决的（修改贴图就算了），下次有时间再试吧。

如果文中有错误或者遗漏，欢迎联系指正。

唯一指定联系邮箱：xysjoshua@hotmail.com
