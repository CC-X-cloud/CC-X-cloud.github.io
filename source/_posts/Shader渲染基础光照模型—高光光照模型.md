---
title: Shader渲染基础光照模型—高光光照模型
date: 2026-05-20 17:46:30
tags:
  - Shader
categories: 渲染
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/GG.webp
---
## 开篇提示
从这篇文章起，之后的所以shader编写将会转为HLSL语言，CG已经有点过时了。
## 高光反射（镜面反射）
**镜面反射**是指若反射面比较光滑，当平行入射的光线射到这个反射面时，仍会平行地向一个方向反射出来，这种反射就属于镜面反射。
高光可以表示出一个物体表面的光滑程度，越光滑表面高光越强。可以想象一个球，有一个手电筒直接打在球上，球的表面越光滑，高光点越小越集中，越出超漫反射就越严重，高光点就越大。
其中关于高光反射的经验方程是
$$L_{specular} = k_s \cdot I \cdot \max(0, V \cdot R)^n$$
- **$k_s$**: 材质的高光反射系数。
- **$I$**: 入射光强度。
- **$V$**: 从顶点指向摄像机的单位向量（View Vector）。
- **$R$**: 光线在表面的单位反射向量（Reflection Vector）。
- **$n$**: 高光指数（Shininess），控制高光的“紧凑”程度。$n$ 越大，高光点越小、越锐利。
Tis:高光指数可以表现未一个材质的光滑度（gloss）。
### **反射向量 $R$ 的计算公式：**

$$R = 2(N \cdot L)N - L$$
（其中 $N$ 为表面法线，$L$ 为指向光源的单位向量）
接下来我将演示如何推导出R（反射向量），在Unity中关照方向是从光照点指向光源
![](GG1.webp)

**反射向量 $R$ 的计算公式疑点讲解**：
为什么想到用投影向量？根据镜面反射的定理，反射光向量和光线向量的模长一定，根据公式反推使R向量加上L向量就可以得到一个未知的向量L1，观察形式可知是投影向量。
### 高光反射的经验方程理解
经验公式描述了这么一个经验现象，把光想成一条条的向量，经过反射之后只有和v重合的可以被看到，经验公式通过计算V和L的夹角（通过点积cos）当夹角为0是点积的结果为1，那么不是一的呢？任何小于一的数都会被指数函数无限缩小就表现为光线迅速减弱，$\max(0, V \cdot R)^n$不要忘记还有n。
下面给一个示意图
![](GG2.webp)
## 效果分析
那么就想上面说的高光反射的效果会让物体有一个高光点，来看一下shader编写中的公式形式
$$c_{specular} = (c_{light} \cdot m_{specular}) \cdot \max(0, \hat{v} \cdot \hat{r})^{m_{gloss}}$$
我们需要物体颜色，光照颜色，观察方向的向量，反射向量，和一个Gloss（通常是自己定）
## Shader编写
和漫反射一样，高光的光照模型也有逐顶点和逐像素两种方式，具体差别和漫反射一样，可以翻看一下漫反射那节
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
Shader "Unlit/Blinn-Phong_URP"
{
    Properties
    {
        _Color("Base Color", Color) = (1,1,1,1)
        _Specular("Specular Color", Color) = (1,1,1,1)
        _Gloss("Gloss/Shininess", Range(1,256)) = 20
    }

    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }

        Pass
        {
            // 修改为 URP 的前向渲染标签
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            // 包含 URP 必要库
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float3 positionWS : TEXCOORD0;
                float3 normalWS   : TEXCOORD1;
            };

            CBUFFER_START(UnityPerMaterial)
                float4 _Color;
                float4 _Specular;
                float _Gloss;
            CBUFFER_END

            Varyings vert(Attributes input)
            {
                Varyings output;
                
                // 获取世界空间位置和裁剪空间位置
                VertexPositionInputs posInputs = GetVertexPositionInputs(input.positionOS.xyz);
                output.positionCS = posInputs.positionCS;
                output.positionWS = posInputs.positionWS; // 关键：计算光照需要它

                // 获取世界空间法线
                output.normalWS = TransformObjectToWorldNormal(input.normalOS);
                
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                // 1. 获取主光源数据 (URP 标准写法)
                Light mainLight = GetMainLight();
                half3 lightDir = normalize(mainLight.direction);
                half3 lightColor = mainLight.color;

                // 2. 准备向量
                half3 normal = normalize(input.normalWS);
                // 使用世界空间坐标计算视线方向
                half3 viewDir = normalize(GetWorldSpaceViewDir(input.positionWS));
                half3 relfect = 2*(dot(normal, lightDir))*normal - lightDir;

                // 3. 计算漫反射 (Lambert)
                half3 diffuse = _Color.rgb * lightColor * saturate(dot(normal, lightDir));

                // 4. 计算高光
                // 注意：pow 的输入需要大于 0
                half specData = saturate(dot(viewDir, relfect));
                half3 specular = _Specular.rgb * lightColor * pow(specData, _Gloss);

                // 5. 组合结果
                return half4(diffuse + specular, 1.0);
            }
            ENDHLSL
        }
    }
}
{% endcodeblock %}
### 半角向量优化（Bilnn-Phone）
先看公式吧
$c_{specular} = (c_{light} \cdot m_{specular}) \cdot \max(0, \hat{n} \cdot \hat{h})^{m_{gloss}}$其中H的计算公式为
$h = \frac{v + l}{|v + l|}$​
下面我们来讲解这个公式的由来
先看这个特殊情况光正好可以被看到你就会发现这时候L向量和V向量相加得出的新向量与N向量重合（微面元理论），这时候这个新向量就代表了V和L向量的角平分线所在的线段，所以叫半角向量。
![](GG3.webp)
这样计算的优点是Phong 模型在计算 $V \cdot R$ 时，如果观察角度大于 90 度，点积会迅速变为负数，导致高光在边缘处戛然而止。
而在 Blinn-Phong 中，因为 $H$ 是平分线，$N \cdot H$ 的夹角增加速度比 $V \cdot R$ 慢（大约是一半）。这使得高光带的衰减更加柔和，能更真实地模拟金属或塑料在边缘处的漫射高光。
对比Phong性能优势

| **步骤**   | **Phong 模型 (反射向量 R)**        | **Blinn-Phong (半角向量 H)**         |
| -------- | ---------------------------- | -------------------------------- |
| **几何逻辑** | 计算 $L$ 关于 $N$ 的对称向量 $R$      | 计算 $L$ 和 $V$ 的角平分线 $H$           |
| **计算开销** | $2(N \cdot L)N - L$ (包含较多乘法) | $(L + V) / \text{length}$ (加法为主) |
| **性能优势** | 必须针对每个顶点/像素重新计算 $R$          | 若光源和相机无限远，$H$ 是常数                |
在代码实现上其实很明了了
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
half3 relfect = 2*(dot(normal, lightDir))*normal - lightDir;
half specData = saturate(dot(viewDir, relfect));
//去掉reflect的计算替换为
half3 halfvector = normalize(viewDir+lightDir);
half specData = saturate(dot(normal, halfvector));
{% endcodeblock %}

## 疑点解析
为何还有混合Lambert光照的效果？单单从代码上理解没有这一部分，物体将只有高光点，就像是被聚光灯照到了一样不符合平行光照的效果。
从模拟现实的方面添加混合是为了更加贴近现实中的物体受光感
当光线照射到非金属（电介质）表面时，会发生两种物理现象：
- **折射与散射（Lambert 部分）**：光线进入物体表面内部，经过多次碰撞后随机散射出来。这部分光失去了方向性，形成了我们看到的“底色”（Diffuse）。
- **表面反射（高光部分）**：光线直接从表面弹射走。这部分光保留了光源的颜色和方向感，形成了“亮点”（Specular）。
如果没有 Lambert，物体看起来就像一个透明的玻璃球或纯黑的金属壳；如果没有高光，物体看起来就像粗糙的石灰粉末。**混合两者，才能让物体看起来像“有涂层的实体”。**