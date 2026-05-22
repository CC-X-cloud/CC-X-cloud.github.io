---
title: Shader渲染—法线映射和视差映射
date: 2026-05-21 16:57:43
tags:
  - 建模
categories: 渲染
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/NP.webp
---
## 开篇提醒
如果你对纹理相关的知识还不熟悉，建议查看实时渲染第四版的第六章，或者我之后会写的纹理相关文章，这里只针对如何实现法线映射和视差映射做讲解。

## 法线贴图和深度贴图
法线贴图是一张存储了模型切线空间下法线向量信息的贴图，而深度图就是存储了模型深度的信息。
这些信息就被我们利用算法计算出正确的法线，然后利用法线来计算光照模型。

## 视差映射配合法线贴图的原理
黑色是物体的实际表面，红色表示高度贴图表示的值。
有一个重点就是理解法线贴图的值在哪里？图上有A和B的两点表示在物体实际表面上的点，A和B点可以构建出切线空间，就是法线和切线和副法线为基向量构成的（橙色坐标系是A的切线空间），那么这个橙色的坐标系中有一个绿色的向量和红色的表面是垂直的，这个绿色向量计算法线贴图存储的信息。
当我们的着色器计算时计算到A点，获得A点的UV坐标。
那么我们就可以知道如果我的向量$V$（切线空间下的视角方向向量），我希望它看到的颜色应该是在B点采样出的法线向量计算出的光照模型，那么我就需要得到一个偏移向量来从A点的UV坐标偏移到B点，来采样。
![](NP1.webp)




## 法线映射和视差映射的推导
### 法线贴图信息的利用

我们已经知道法线贴图的信息是存储在切线空间下的，那么 我们的计算需要统一坐标空间，就延申出两种思路，一种是把切线转到世界空间，一种是把世界转到切线空间，我们来看一些对比图

|特性|方法一：世界/模型转切线|方法二：切线转世界/模型|
|---|---|---|
|**核心操作**|把 `L` 和 `V` 变换到切线空间|把采样出的 `N` 变换到世界空间|
|**重计算在哪里**|**顶点着色器** (变换L/V)|**片元着色器** (变换N)|
|**片元负担**|低|高 (需矩阵乘法，可能需正交化)|
|**法线输出**|仅用于本物体本帧光照|可写入G-Buffer，全局可用|
|**管线适配**|传统前向渲染|延迟渲染、任何需要世界法线的管线|
|**与视差映射配合**|**天然完美配合**|需要额外的空间逆变换，较繁琐|
两种方法并无优劣看那种合适就可以，后续采用方法二计算。
 转换矩阵：
- 切线空间到世界空间的转换矩阵为一个3×3的旋转矩阵，一般称为TBN矩阵
- 世界空间到切线空间的转换矩阵为上述TBN矩阵的逆矩阵，因为是正交矩阵，所以逆矩阵就是它的转置矩阵
Tip：关于这里是怎么推导的请期待后续文章，这里不过多演示
![](NP2.webp)

## 深度贴图利用
深度图主要存储了模型深度值，配合法线贴图使用。深度图的使用是用来计算偏移值。

### 如何计算偏移量

前提我们所以的计算都是在切线空间下的，切线空间可以理解为一个点的局部坐标。
当我们计算A点时，我们可以得到$V$的切线空间下的向量，为了方便计算我们就假设$V$
和A点的X轴在同一平面（xAz）上，去掉y轴，这时候分解向量$V$为$V.z$和$V.xy$我们又已经知道了$H_A$的深度信息，可以构建出绿色和蓝色的相似三角形。
由于相似三角行成比例就有
$$\frac{a}{B} = \frac{b}{A}$$
代入我们的变量，就得到了关键等式：
$$\frac{H}{V.z} = \frac{|\text{Offset}|}{|V_{xy}|}$$
我们想要的是偏移向量 `Offset`，它不仅有大小，还有方向。它的大小可以这样解出
$$|\text{Offset}| = \frac{H \cdot |V_{xy}|}{V.z}$$
​那么方向呢？视线在切线平面内的投影方向 `V_xy`，就是我们偏移的方向。把 `Offset` 向量表示为

$$\text{Offset} = \text{normalize}(V_{xy}) \times |\text{Offset}|$$
$$\text{Offset} = V_{xy} \times \frac{H}{V.z}$$

![](NP3.webp)
## Shader实现
### 实现分析
我们已经知道了如何求的偏移量的公式，得到的偏移量其实一个二维向量，用偏移量加上现在的UV坐标，得到的偏移UV坐标就是我们采样颜色贴图，和法线贴图来计算光照模型用的UV坐标。


{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}

Shader "Unlit/HighNormal"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Bump("Normal Tex",2D) = "bump"{}
        _High("High Tex",2D) = "white"{}

        _BumpSacle("BumpSacle",Range(0,2)) = 1
        _HighScale("HightScale",Float) = 1

        _Specular("Specular",Color) = (1,1,1,1)
        _Gloss("Gloss",Range(0,1)) = 0.5
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        Pass
        {
            Tags{"RenderPipeline" = "UniversalPipeline"  "RenderType" = "Opaque"}
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            TEXTURE2D(_MainTex);   SAMPLER(sampler_MainTex);
            TEXTURE2D(_Bump);   SAMPLER(sampler_Bump);
            TEXTURE2D(_High);   SAMPLER(sampler_High);

            CBUFFER_START(UnityPerMaterial)
            float4 _MainTex_ST;
            float4 _Bump_ST;
            float4 _High_ST;
            float _BumpSacle;
            float _HighScale;
            float4 _Specular;
            float _Gloss;
            CBUFFER_END

            struct Attributes{
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float4 tangentOS : TANGENT;
                float2 uv : TEXCOORD0;
            
            };
            struct Varyings{
                float4 positionCS : SV_POSITION;
                float3 normalWS : TEXCOORD0;
                float4 tangentWS : TEXCOORD1;
                float3 worldpos : TEXCOORD2;
                float3 viewdirTS : TEXCOORD3;
                float2 uv : TEXCOORD4;
                float3 binormalWS : TEXCOORD5;
            };
            Varyings vert (Attributes v){
                Varyings o;

                o.uv = TRANSFORM_TEX(v.uv,_MainTex);

                o.positionCS = TransformObjectToHClip(v.positionOS);
                o.normalWS = TransformObjectToWorldNormal(v.normalOS);
                o.tangentWS.xyz = TransformObjectToWorldDir(v.tangentOS.xyz);
                o.tangentWS.w = v.tangentOS.w;  // 保留镜像符号
                o.binormalWS = cross(o.normalWS,o.tangentWS.xyz) * v.tangentOS.w;
                o.worldpos = TransformObjectToWorld(v.positionOS.xyz);

                float3 viewdirWS = GetCameraPositionWS() - o.worldpos;
                float3x3 TBN = float3x3(o.tangentWS.xyz,o.binormalWS,o.normalWS);
                //意思是乘逆矩阵
                o.viewdirTS = mul(viewdirWS,TBN);
                return o;
            }
            float4 frag(Varyings i) : SV_Target
            {
                // 1. 朴素视差偏移
                float curhigh = SAMPLE_TEXTURE2D(_High, sampler_High, i.uv).r;
                float heightOffset = curhigh - 0.5;
                float2 offset = (heightOffset * _HighScale) / (abs(i.viewdirTS.z) + 0.001) * i.viewdirTS.xy;
                float2 parallaxUV = i.uv - offset;

                // 2. 采样并解码法线
                float3 normalTS = UnpackNormalScale(SAMPLE_TEXTURE2D(_Bump, sampler_Bump, parallaxUV), _BumpSacle);

                // 3. TBN变换法线到世界空间
                float3x3 TBN = float3x3(i.tangentWS.xyz, i.binormalWS, i.normalWS);
                float3 worldnormal = normalize(mul(TBN, normalTS));

                // 4. 采样主纹理
                float3 albedo = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, parallaxUV).rgb;

                // 5. 光照
                Light mainlight = GetMainLight();
                float3 lightdir = mainlight.direction;
                float3 lightcolor = mainlight.color;
                float3 viewdirWS = normalize(GetCameraPositionWS() - i.worldpos);

                float NdotL = saturate(dot(worldnormal, lightdir));
                float3 diffuse = albedo * lightcolor * NdotL;

                float3 halfdir = normalize(lightdir + viewdirWS);
                float HdotN = saturate(dot(halfdir, worldnormal));
                float3 specular = lightcolor * pow(HdotN, _Gloss * 256);

                float3 ambient = SampleSH(worldnormal)*albedo;

                return float4(diffuse + specular + ambient, 1);
            }


            ENDHLSL
        }
    }
}
{% endcodeblock %}


## 疑点解析
### 为什么得出来的偏移向量可以直接作用到现在在计算的点的UV坐标上？
UV 展开 = 切线空间的 2D 投影，UV图上存储了模型的顶点信息
当你对一个模型进行 UV 展开时点的切线空间存储了信息：
- **U 方向** = 切线空间的 **X 轴**（切线 Tangent）
- **V 方向** = 切线空间的 **Y 轴**（副法线 Binormal）
- - **表面法线** = 切线空间的 **Z 轴**（法线 Normal）
### o.viewdirTS = mul(viewdirWS,TBN); 为什么就是乘以逆矩阵？
先来理解mul函数的性质

|写法|数学含义|维度要求|
|---|---|---|
|`mul(vector, matrix)`|**行向量** × 矩阵|vector 是行向量 (1×N)|
|`mul(matrix, vector)`|矩阵 × **列向量**|vector 是列向量 (N×1)|
|`mul(matrixA, matrixB)`|矩阵 × 矩阵|标准矩阵乘法|
而且因为 TBN 是正交矩阵（逆=转置），所以成立，感兴趣可以自己试试，这里不过多介绍