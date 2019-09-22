---
layout:     post
title:      "Ray Tracing in One Weekend - Study Note 04"
subtitle:   "周末掌握光线追踪 - 学习笔记四：漫反射和金属"
date:       2018-05-28 20:00:00
author:     "Xjoshua"
header-img: "img/post-bg-default.jpg"
catalog: 	true
tags:
  - StudyNotes
  - Ray Tracing
  - Render
  - Computer Graphics
---

## Chapter 7：漫反射材料

现在我们有物体和针对每个像素的多种光线，我们可以做一些看起来更真实的材料。先从漫反射材质材质（无光泽的）入手。一个问题是我们是要混合和适配材质和形状（所以我们会给球一个材质）还是把他们放在一起把他们联系起来。（这对几何形态和材料链接起来的程序生产的物体是很有用的）（PS：这说的是啥？）我们会分开处理——大多数渲染器都是这样处理的，但是要知道它的限制。  

漫反射的物体不会发光，仅仅是接受周围环境的颜色，但是这些环境色会混合自己的颜色。从漫反射表面反射的光线会有随机的方向。那么，如果我们把三道光线发送到两个漫反射表面的缝隙中，他们会有不同的随机表现：  
![041-preview.png](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1805/024-preview02.png)

他们也许还会被吸收而不是反射。表面越暗，吸收程度越高（这就是暗的原因）。事实上所有方向随机的算法都会产生看起来无光泽的表面。一个简单的方法结果却正确的产生了漫反射的表面。（我常常就懒惰用取巧的方式来做这个，但我的博客有一个评论展示了事实上数学上理想的Lambertian的模型）  

在击中点上相切的单位球上随机选个点s，从击中的点p到随机点s发射光线。球心坐标在（p+N）：  
（PS：产生随机方向射线的方法）  
![042-preview.png](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1805/024-preview02.png)

我们也需要一个方法，能够在原点的单位圆上生成一个随机点。我们用通常来说最简单的算法：拒绝法（rejection method）。首先，选一个x,y,z都在（-1，1）范围的方块内的随机点。如果这个点在球的外面我们就重选一个。（汗）do/while循环结构就很适合来做这个。  

```Cpp
// 生成随机
vec3 random_in_unit_sphere()
{
    vec3 p;
    // 产生(-1,1)的矢量，丢弃长度大于1的矢量，剩下的就是球内的矢量
    do {
        p = 2.0 * vec3(drand48(), drand48(), drand48()) - vec3(1, 1, 1);
    } while (dot(p, p) >= 1.0);
    return p;
}

vec3 color(const ray& r, hitable *world)
{
    hit_record rec;
    if (world->hit(r, 0.0, FLT_MAX, rec)) {
        // 击中点处相切单位球上的随机点
        vec3 target = rec.p + rec.normal + random_in_unit_sphere();
        // 从点p射出到随机点的光线（漫反射光线），递归直到没有击中物体
        return 0.5*color(ray(rec.p, target - rec.p), world);
        //return 0.5*vec3(rec.normal.x() + 1, rec.normal.y() + 1, rec.normal.z() + 1);
    }
    else {
        vec3 unit_direction = unit_vector(r.direction());
        float t = 0.5*(unit_direction.y() + 1.0);
        return (1.0 - t)*vec3(1.0, 1.0, 1.0) + t*vec3(0.5, 0.7, 1.0);
    }
}
```

输出：  
![043-preview.png](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1805/043-preview.png)

注意球体下方的阴影。这张图相当的暗，但是我们的球在每次弹射中也只是吸收了一半的能量，所以只有50%的反射（PS：这是什么意思）。如果你看不清阴影，没关系，我们现在修复它。这些球应该看起来更亮一些（现实生活中，亮灰色）原因在于所有的图片浏览器都会假定图片做过“gamma 校正”，意味着0-1的值在保存之前做过一些变换。有足够好的理由这么做，但对我们来说只需要注意这一点即可。对于第一次近似来说，我们使用“gamma 2”，意思是把颜色变化原来的 1/gamma 次幂，对我们的简单案例来说就是 1/2，也就是开平方：
增加下面的代码（Main函数中）：  
```Cpp
col = vec3(sqrt(col[0]), sqrt(col[1]), sqrt(col[2]));
```

> PS：解释一下Gamma Correction
>（由于进化，人类的视觉系统）对于暗色的分辨能力远超过亮色。那么在有限的计算机颜色（民用显示器和操作系统中黑色到白色256个色阶）中，亮色和暗色均匀分布的话，那亮色部分就会精度过剩而暗色部分就会精度不足。如何解决这个问题？进行 Gamma 矫正。
> 参考Avater Ye和韩世麟的答案
> 来自 <https://www.zhihu.com/question/27467127/answer/37602200> 


这样就生成了亮灰色，正如我们的期望：
![044-preview.png](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1805/044-preview.png)

## Chapter 8：金属材料

如果我们想在不同物体有不同的材质，我们有个设计决策。我们能有一个通用的材质，这个材质有着各种大量参数，而不同的材质只是取消一些参数。这不能说是一种不好的方式。或者我们用一个抽象的材质类来封装表象（？例如漫反射，反射之类的？）。我是后一种方式的支持者。对于我们的程序来说这个材质需要做两件事：  
- （是否）产生散射光线（或者说会不会吸收入射光）
- 如果散射了，光线如何衰减

抽象类：  
```Cpp
class material {
	public:
		virtual bool scatter(const ray& r_in, const hit_record& rec, vec3& atteuation, ray& scattered) const = 0;
};
```

`hit_record`节约了一大堆的参数，所以我们可以放很多想要的东西进去。你也可以直接用大量的参数，这是品味的问题。`Hitables`和`meterials`需要互相参照，这是个依赖的怪圈。在C++中你只需要警告编译器把指针指向一个类（ps：不懂），就是hitable中的`material`类那样：

```Cpp
// 代码
```

对于我们之前的`Lambertian`案例来说，它的反射可以被反射率R减弱，也可以看作是有（1-R）被吸收而不是减弱了，又或者是这两种策略的混合（ps：？这不是一个意思吗？不懂）关于`lambertian`材料我们有一个简单的类：

```Cpp
class lambertian :public material {
public:
    lambertian(const vec3& a) : albedo(a) {}
    virtual bool scatter(const ray& r_in, const hit_record& rec, vec3& attenuation, ray& scattered) const {
        vec3 target = rec.p + rec.normal + random_in_unit_sphere();
        scattered = ray(rec.p, target - rec.p);
        attenuation = albedo;
        return true;
    }
};
```

我们可以用p来散射光线，也可以用`albedo/p`。你自己可以做选择。  

对于光滑的金属来说，光线不会是随机的散射。关键的计算在于：从金属镜面反射的光线是怎么表现的？向量是我们的朋友：  
![045-diagram.png](https://raw.githubusercontent.com/XJoshua/XJoshua.github.io/master/img/in-post/1805/045-diagram.png)

红色的反射光线是（v+2B）。在我们的设计中，N是一个单位向量，但是v可能不是。B的长度应该是dot(v,N)。因为...

> WARNING：施工未完成

未完待续。

#### 其他笔记

1. [读书笔记一：输出图像和向量类](https://xjoshua.github.io/2018/05/22/SimpleMaze-0x01/) 
2. [读书笔记二：光线和球体](https://xjoshua.github.io/2018/05/24/RTIOW-StudyNote02/)
3. [读书笔记三：法线，多物体和抗锯齿](https://xjoshua.github.io/2018/05/22/SimpleMaze-0x01/)
4. [读书笔记五：施工中](https://xjoshua.github.io/2018/05/22/SimpleMaze-0x01/)

##### 参考资料：

> 1. ["In One Weekend：原书博客"](http://in1weekend.blogspot.com/) 
> 2. ["作者发布在GoogleDrive上的PDF，请尽量在Amazon购买支持"](https://drive.google.com/drive/folders/14yayBb9XiL16lmuhbYhhvea8mKUUK77W) 
> 3. ["总结《Ray Tracing in One Weekend》 by 图形跟班"](https://blog.csdn.net/libing_zeng/article/details/72598060?locationNum=7&fps=1)
