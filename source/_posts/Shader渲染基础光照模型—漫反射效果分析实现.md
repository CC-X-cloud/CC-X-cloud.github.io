---
title: Shader渲染基础光照模型—漫反射效果分析实现
date: 2026-05-19 19:16:19
tags:
  - Shader
categories: 渲染
cover:
---
## 什么是漫反射和辐照度（朗伯余弦定律）？
漫反射描述了这样一个物理现象：一束笔直的光照沿着直线打在粗糙的物体上，物体表面表面会把光线向着四面八方反射，所以入射线虽然互相平行，由于各点的法线方向不一致，造成反射光线向不同的方向无规则地反射。
![](MFS2.webp)
辐照度是描述光照强度的一个物理量，是指是受照面单位面积上的辐射通量。
辐照度的计算公式有很多，但是我们只需要知道朗伯余弦定律公式就可以了
当光线以入射角 θ照射到表面时：
$$E = E_n \cdot \cos\theta$$

其中：
- ​ $E_0$为法向入射时的辐照度
- θ为入射角（光线与法线的夹角）
PS：为什么要用 $\cos\theta$呢？光L从侧面射入分解到垂直方向的为$\cos\theta \cdot L$
![](MFS2.webp)
### 朗伯余弦定律
漫反射强度 ∝ cos(θ)（ ∝正相关）

- θ = 0°（垂直入射）：强度最大（1）
    
- θ = 90°（平行入射）：强度为0
    
- θ > 90°（背面入射）：光线无法到达表面

## 漫反射效果分析
首先一个物体反射出来的颜色是它本身的颜色（有点废话），而且物体表现的颜色还能受到光线本身的颜色影响。
一个物体有受光面和背光面，而且有明暗的渐变，这是因为辐照度的原因，而我们要得到辐照度就要知道入射角θ，在Unity里你可以得到normal法线向量和光线向量，通过点乘运算的公式。$$\vec{n} \cdot \vec{l} = |\vec{n}| \, |\vec{l}| \cos\theta$$
当这个法线向量和光线向量为单位向量时就可以得到入射角θ了。

换到Shader编写下的公式其中C代表颜色，m代表材质的漫反射颜色，`max(0, n·l)` 是为了**剔除背光面**，避免光线从物体背面照亮正面，产生错误的物理效果，符合朗伯余弦定律。
$c_{diffuse} = (c_{light} \cdot m_{diffuse}) \cdot \max(0, \hat{n} \cdot \hat{l})​$

## Shader代码编写
漫反射有两种编写模式分别为逐顶点（Grouraud）和逐像素(Phong)，和一种优化写法半兰伯特（half Lambert）
| 对比项 | 逐顶点光照 | 逐像素光照 |
| --- | --- | --- |
| **计算位置** | 顶点着色器 | 片元着色器 |
| **颜色变化** | 三角形内线性插值 | 每个像素独立计算 |
| **高光效果** | 容易产生棱角/锯齿 | **平滑、细腻** |
| **性能** | 较好 | 稍差（但现代GPU可接受） |
| **别名** | Gouraud着色 | Phong着色（注意：不是Phong光照模型） |
其实就是在顶点着色器计算还是在片元着色器计算
这里代码用逐像素处理，因为逐顶点用的不多。
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
Shader "Unlit/DIffuse reflact Pixel"
{
    Properties
    {
        //物体的本身的颜色
        _Diffuse("Diffuse", Color) = (1,1,1,1)
    }
    SubShader
    {


        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            //包含的库
            #include "UnityCG.cginc"
            #include "Lighting.cginc"
            float4 _Diffuse;
           struct a2v{
               //添加输入语义获得模型空间的顶点位置和模型空间中的法线位置
               float4 vertex:POSITION;
                float3 normal:NORMAL;
           };
           struct v2f{
               //添加输出语义输出裁切空间下的顶点位置和自定义纹理坐标数据
               float4 pos:SV_POSITION;
               float3 worldNormal : TEXCOORD0;
               };
               v2f vert(a2v v){
                   v2f o;
                   o.pos = UnityObjectToClipPos(v.vertex);
                   //得到空间坐标下的法线向量
                   o.worldNormal = mul((float3x3)unity_ObjectToWorld, v.normal);
                   return o;
               }
               fixed4 frag(v2f i): SV_TARGET{
                i.worldNormal = normalize(i.worldNormal);
                //获得环境光
                fixed3 ambinet = UNITY_LIGHTMODEL_AMBIENT.xyz;
                //得到空间坐标下光照向量
                fixed3 WorldRayDirection = normalize(_WorldSpaceLightPos0.xyz);
                //计算反射效果（颜色相乘是叠加颜色）
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(i.worldNormal, WorldRayDirection));
                fixed3 color = ambinet + diffuse;
                return fixed4(color, 1);
               }

            ENDCG
        }
    }
}
{% endcodeblock %}
### 半兰伯特优化
公式
$c_{diffuse} = (c_{light} \cdot m_{diffuse}) \cdot (0.5 \cdot (\hat{n} \cdot \hat{l}) + 0.5) ​$

{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(i.worldNormal, WorldRayDirection));
//转换为
fixed halfLambert = dot(worldNormal, worldLightDir) * 0.5 + 0.5; 
fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * halfLambert;
{% endcodeblock %}

## 疑点解析

### 为什么要从模型空间转换到世界空间？
将法线、位置等转换到**世界空间**进行计算，主要是为了**统一坐标系**，**让不同物体之间的计算有相同的参考基准**
哎上面这段话说的是不能讲不同空间下的向量相乘的原因，还有一个重要的原因是节省性能，你注意看我们转世界坐标是在顶点着色器里，在片元着色器里直接通过API得到的世界空间的光线（不用算），如果我们要用模型空间的坐标就需要在片元着色器里计算世界转模型，这有什么区别呢不都需要计算？区别是计算量，你试想一个正方形有四个顶点，而他的像素量呢？是顶点的几千倍不止。
### 为什么颜色相乘是叠加颜色？
颜色相乘（`color1 * color2`）是**吸收/过滤模型**的数学表示，模拟了光线被物体吸收某些波长后剩下的颜色。
这个可以理解为rgb三个通道从1到0为三个不同灯泡（LED灯类似）的亮度表示为
（1，1，1），一开始都是全亮的通过加滤片控制亮度，你加了一个滤片只让光通过30%，变成（0.3，0.3，0.3），那么你加第二片滤片的时候又只让关透过10%，那么这时候就要在上一个滤片的基础上乘掉10%才对，你看这时候的混合颜色是不是就是这两个滤片的混合颜色。

