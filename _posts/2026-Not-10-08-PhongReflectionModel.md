---
layout: post
published: false
title: Phong Illumination, Phong Shading
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-08 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
Phong, Reflection


--- 
### 1. Phong Illumination(Reflection) Model
### 1.1 Ambient


$$I_a = k_a \cdot i_a$$

- $k_a$: 재질의 ambient 반사 계수
- $i_a$: ambient 광원의 세기

### 1.2 Diffuse

$$I_d = k_d \cdot i_d \cdot \max(\hat{L} \cdot \hat{N}, 0)$$
- $k_d$: 재질의 diffuse 반사 계수
- $i_d$: diffuse 광원의 세기
- $\hat{L}$: 표면점에서 광원으로 향하는 단위 벡터
- $\hat{N}$: 표면의 단위 법선 벡터

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/Lambertlaw.png" alt="Lambertlaw" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Lambertlaw</strong> </td>  </tr> </table>
- LightVector - Surface Normal과의 사이각으로 dotproduct 로 처리,
- 0도보다는 아래 내려가지 못하도록 `max( dot(L,N), 0)`으로 잡음


### 1.3 Specular
$$I_s = k_s \cdot i_s \cdot \max(0,\ \hat{R} \cdot \hat{V})^{\alpha}$$
**반사 벡터(Refelction vector)**:
$$\hat{R} = 2(\hat{L} \cdot \hat{N})\hat{N} - \hat{L}$$

- $k_s$: 재질의 specular 반사 계수
- $i_s$: specular 광원의 세기
- $\hat{R}$: 광원 방향 $\hat{L}$이 법선 $\hat{N}$에 대해 반사된 단위 벡터
- $\hat{V}$: 표면점에서 카메라(관찰자)로 향하는 단위 벡터
- `α`: shininess(광택 지수). 클수록 하이라이트가 작고 날카로워짐
  `α`로 계수 표현해서 날카롭게 만듦

$$I = I_a + I_d + I_s$$ $$I = k_a\, i_a + k_d\, i_d\, \max(0,\ \hat{L} \cdot \hat{N}) + k_s\, i_s\, \max(0,\ \hat{R} \cdot \hat{V})^{\alpha}$$
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/SpecularLightVector.png" alt="Lambertlaw" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SpecularLightVector</strong> </td>  </tr> </table>

그럼에도 불구하고 R을 구해 내야하는데


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/Specularlaw.png" alt="Specularlaw" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Specularlaw</strong> </td>  </tr> </table>
E를 쉽게 구하니까 중간 vector를 그냥 구해서 R대신 쓰자
-> Blinn Phong shader, Half Vecetor라고함




---
### 2. Shading Model

결국에는 도달하는 빛의 Intensity를 어떻게 연산하겠냐의 차이이다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/Shading_models.png" alt="GraphBasic" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Shading_models(Wikimedia-commns)</strong> </td>  </tr> </table>

위 세가지 Shading Model에 대해서 결정할 수있는 Shading을 phong shading model이라 한다.

가장 극명하게 FLAT, GOURAUD, PHONG은 부드럽게 처리하는 Interpolation이 들어간게 분명하다.  
그렇다면 어떤 연산으로 달리 처리할까?

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/vertexInterpolation.png" alt="GraphBasic" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>vertexInterpolation</strong> </td>  </tr> </table>
위 이미지와 같이 Polygon의 정보는 Vertex마다 정보를 갖고있다. 그 중점이 되는 값들은 어떻게 처리를하냐에 따라서 달라지는것이 ShadingModel이다.

이렇게 물을 수 있다.
> vtx기준으로 normal이 들어가는데, 그렇다면 Face내부인, Pixel에 있는 빛의 계산은 어떻게 표현하는것일까?



#### 1. Flat Shading
- 삼각형 당 한번 만 조명 계산 - 동일한 삼각형내의 Face는 동일한 빛을 받아 intensity값을 계산한다.
- Face Normal 사용, 계산된 색 면 전체에 동일하게 적용


$$I_{face} = k_d \cdot i_d \cdot \max(0,\ \hat{L} \cdot \hat{N}_{face})$$

- 동일한 삼각형(Face) 내에 있는  픽셀에 하나의 LightModel - Normal만을 계산해서 빠르다
- HardenEdge와 같이 끊겨서 보인다
  - Quad Modeling은 사각형이 최소primitive로, 두 삼각형이 같은 normal을 같기에 같은 intensity

-> 3가지 vtx의 cross product로 normal vector를 구한다.


**연산 순서**:  
1. vtx로 부터 삼각형 하나의 normal vector을 구한다
2. 1번으로 구한 normal vector와 light model과 연산
   -> output : 삼각형 내부는 모두 동일한 pixel intensity

- 각 pixel 단위로 intensity 계산 결과 : 개별적인 Primitive(삼각형/Face) 마다의 같은 빛의 intensity 연산(hardenedge)

#### 2. Gouraud Shade
- Flat Shade는 너무 끊겨서 보인다

- 각 Pixel에 해당하는 Light Intensity 구하는 방법:  
  -> Vtx마다 normal로부터 lightmodel과 연산 (Vertex Shader에서 수행)

- 계산된 색상을 삼각형 내부에서 **선형 보간**
	  - Scanline 기준으로 삼각형 두변 따라 보간, 또 한번더 가로방향으로 한번 더 보간
명암을 계산 다시 하자
- 선형 보간을 두번해서, 삼각형 내부에있어서 보간을 다 할수가있다
- 색이 부드럽게 이어져서 **매끄럽게** 나온다
- 단점 : 하이라이트(Specular)가 삼각형 내부에 있고, vtx에는 걸리지 않으면 **놓치거나 번져보임**

**Gouraud shade 연산**:  
1. **FlatShade로부터 구한 삼각형 face의 normal**로부터 vtx의 normal을 평균내 구한다
2. vtx normal과 light vector를 연산
   -> output : vertex intensity
3. Edge에 물린 pixel 대해서 vtx를 선형 보간으로 intensity를 계산한다
   -> output : 테두리 pixel intensity
4. 3번으로 테두리 pixel intensity로 내부의 픽셀도 한번 더 보간한다
   -> output : 삼각형 내부 pixel intensity

- 각 pixel 단위로 intensity 계산 결과 : pixel마다의 부드러운 intensity 연산(highlight퍼짐)



#### 3. Phong
- Gouraud가 highlight이 퍼지기 때문에 ShadSpecular의 highlight를 계산할 수있도록

- 각 Pixel에 해당하는 Light Intensity 구하는 방법:  
  -> 각 Pixel에 해당하는 normal을 다시 구해서 light model과 다시 연산

**Phong 연산**:  
1. **Gouraud Shade로부터 구한 각 vertex의 normal**을 각 pixel에 해당하는 Normal을 보간해 구한다
   -> Output : Pixel vertex Normal
2. 1번으로 부터의 pixel Normal로 Light Vector 을 연산한다(조명 모델 재 적용)
   ->Output : pixel Intensity

- 각 pixel 단위로 intensity 계산 결과 : 부드러운 Highlight연산





<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/HardenSmooth.jpg" alt="GraphBasic" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Harden/Soften Edge</strong> </td>  </tr> </table>
- 위와 같이 Dcc tool에서도 Harden/Soften Edge로 개별적인 model vtx를 설정으로도 가능
- shade단에서가아니라 object vertex normal 속성으로 들어가겠다.


---
### Ambient Light

$$I_{ambient} = k_a \cdot i_a$$
``` hlsl
float3 ambient = gAmbientLight * mat.Ambient;
```
- 광원 위치/방향의 영향 없어서 거리 감쇠도 없음
- Model Normal 계산하지도 않음

### Directional Light

- 감쇠 없음
- homogeneous coordinate : 0

### Point Light

- vertex - point light - distance거리 계산

### Spotlight
- $P_{s}$: apex
- $I_{S}$: light driection
- θ : angle

선형이 아니어서 코드로하기보다 그래프로 처리한다?
light fall off





---
