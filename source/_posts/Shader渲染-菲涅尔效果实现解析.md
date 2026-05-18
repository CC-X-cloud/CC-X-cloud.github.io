---
title: Shader渲染-菲涅尔效果实现解析
date: 2026-05-18 10:16:27
cover: https://example.com/cover.webp
tags:
  - Shader
categories: 渲染
---
## 菲涅尔效应是什么？
理解菲涅尔效应，你首先得知道在自然界中会发生反射和折射，而菲涅尔就是描述反射/折射与视点角度之间的关系。
举一个例子，假设有一个十分平静的水面，当你垂直看向水面的时候，反射是最强的，折射是最弱的，当年倾斜45度的时候，你能既能看到水下面的东西和天空的颜色，因为这时候反射和折射同时存在。
![](FNE1.webp|434)
## 如何计算菲涅尔效应？

Schlick菲涅尔近似等式：
$F(\theta) = F_0 + (1 - F_0)(1 - \cos\theta)^5$
其中$F_0$是为法线入射时的反射率（通常取 0.04），$\cos\theta = \hat{v} \cdot \hat{n}​$
$\hat{v}$ $\hat{n}​$分别为法向向量和观察角view向量
得出来的是反射率。

Empricial菲涅而近似等式
$F_Empirical(v, n) = max(0, min(1, bias + scale × (1 - v·n)^{power}))$
其中bias和scale，power都是控制项代表偏移，缩放，强度。
$\hat{v}$ $\hat{n}​$同上

## Shader代码实现（Unity）

这里我们用简单一点的Schlick菲涅尔等式实现。
我们要知道光线发生菲涅尔现象之后如何传播，很显然一部分发生反射沿着一定角度弹回去，一部分发生折射现象。
那么折射的光去哪里了？首先你要知道折射现象能发生在任何物体上，包括不透明物体。
非金属不透明物体：折射光被吸收，通过物体内部粒子散射最后形成漫反射的效果。
金属不透明物体：折射光被物体内部粒子几乎全部吸收，没有漫反射，只有表面反射。
半透明材质：和非金属不透明物体差不多最终都是呈现漫反射效果
透明材质：折射光全部透过物体，发生透视。
这里我们只实现最基本的不透明漫反射。
这时候我们可以知道我们实现这些效果就需要计算慢反射，菲涅尔的反射部分，和环境光的影响。


`Shader "Unlit/Fresnel"`
`{`
    `Properties`
    `{`
        `_Color("Color",Color) = (1,1,1,1)`
        `_FresnelScale("FresnelScale",Range(0,1)) = 0.5`
        `_Cubemap("ReflectionCubemap",Cube) = "_Skybox"{}`
        `//这些是可控制项，分别是物体本身的颜色，物体的反射率，反射的天空盒`
    `}`
    `SubShader`
    `{`
        
        `Pass`
        `{`
            `CGPROGRAM`
            `#pragma vertex vert`
            `#pragma fragment frag`

            `//需要用到的库`
            `#include "UnityCG.cginc"`
            `#include "Lighting.cginc"`
            `#include "AutoLight.cginc"`

            `float4 _Color;`
            `float _FresnelScale;`
            `samplerCUBE _Cubemap;`

            `//经由公式得知要计算菲涅尔效应我们应该得知道物体的法线`
            `//世界空间下的观察角度和反射角（反射角用来采样）`
            `struct appdata`
            `{`
                `float4 vertex : POSITION;`
                `//用来接收物体的法线并传递给片元着色器`
                `float3 normal : NORMAL;`
            `};`

            `struct v2f`
            `{`
                `//用来接受顶点着色器的信息并传递出去`
                `float4 pos : SV_POSITION;`
                `float3 worldnormal : TEXCOORD0;`
                `float3 worldpos : TEXCOORD1;`
                `float3 worldviewdir : TEXCOORD2;`
                `float3 worldrefl : TEXCOORD3;`
            `};`

            `v2f vert (appdata v)`
            `{`
                `v2f o;`
                `o.pos = UnityObjectToClipPos(v.vertex);`
                `o.worldnormal = UnityObjectToWorldNormal(v.normal);`
                `o.worldpos = mul(unity_ObjectToWorld,v.vertex);`
                `o.worldviewdir = UnityWorldSpaceViewDir(o.worldpos);`
                `o.worldrefl = reflect(-o.worldviewdir,o.worldnormal);`
                `//可以用来接收阴影信息`
                `TRANSFER_SHADOW(o);`

                `return o;`
            `}`

            `fixed4 frag (v2f i) : SV_Target`
            `{`
                `fixed3 worldnormal = normalize(i.worldnormal);`
                `fixed3 worldlightdir = normalize(UnityWorldSpaceLightDir(i.worldpos));`
                `fixed3 worldviewdir = normalize(i.worldviewdir);`

                `//计算环境光照`
                `fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.rgb;`

                `//计算光照衰减值，平滑阴影`
                `UNITY_LIGHT_ATTENUATION(atten,i,i.worldpos);`

                `//按反射角采样出来应该反射出天空盒那一块的像素`
                `fixed3 reflection = texCUBE(_Cubemap,i.worldrefl).rgb;`

                `fixed fresnel = _FresnelScale + (1-_FresnelScale)*pow(1-dot(worldviewdir,worldnormal),5);`

                `fixed3 diffuse = _LightColor0.rgb + _Color.rgb * max(0,dot(worldnormal,worldlightdir));`

                `//最终合并颜色并输出`
                `fixed3 color = ambient + lerp(diffuse,reflection,saturate(fresnel))*atten;`

                `return fixed4(color,1);`
            `}`
            `ENDCG`
        `}`
    `}`
`}`

## 详细讲解部分
`fixed3 color = ambient + lerp(diffuse,reflection,saturate(fresnel))*atten;`
为什么呢采用lerp插值计算盒saturate将值限制在 0 到 1 之间？
lerp 是线性插值函数，公式为：a + (b - a) * t
 `a` = `diffuse`（漫反射颜色）
`b` = `reflection`（反射颜色）
`t` = `saturate(fresnel)`（插值因子）
lerp效果渐变过渡，自然真实，saturate确保插值因子始终在有效范围内。

 `fixed3 reflection = texCUBE(_Cubemap,i.worldrefl).rgb;`
如何采样的？从立方体贴图中按照 i.worldrefl 方向，采样出该方向看到的颜色。
差不多如图示的意思但是这里的坐标系只是方便理解真实情况不是这样的。
![](FNE2.webp)


