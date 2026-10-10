---
layout: post
published: false
title: Phong ReflectionModel
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-08 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
[AnimationSandMud](https://arxiv.org/pdf/2302.08683)는 현 RenderTarget기반 지면 변형(Deformation) 기법의 초석이되는 개념

Sumner, O'Brien, Hodgins, *_Computer Graphics Forum_* Vol. 18 No. 1, 1999[arXiv:2302.08683](https://arxiv.org/pdf/2302.08683) 논문을 리뷰해보자.


실시간은 아니다. 프레임 단위로 계산 후, 결과 저장해 렌더링하는 Offline-Batch Simulation

--- 
## 1. Ambient


$$I_a = k_a \cdot i_a$$

- $k_a$: 재질의 ambient 반사 계수
- $i_a$: ambient 광원의 세기

### 2. Diffuse

$$I_d = k_d \cdot i_d \cdot \max(0,\ \hat{L} \cdot \hat{N})$$
- $k_d$: 재질의 diffuse 반사 계수
- $i_d$: diffuse 광원의 세기
- $\hat{L}$: 표면점에서 광원으로 향하는 단위 벡터
- $\hat{N}$: 표면의 단위 법선 벡터


### 3. Specular
$$I_s = k_s \cdot i_s \cdot \max(0,\ \hat{R} \cdot \hat{V})^{\alpha}$$
**반사 벡터(Refelction vector)**:
$$\hat{R} = 2(\hat{L} \cdot \hat{N})\hat{N} - \hat{L}$$

- $k_s$: 재질의 specular 반사 계수
- $i_s$: specular 광원의 세기
- $\hat{R}$: 광원 방향 $\hat{L}$이 법선 $\hat{N}$에 대해 반사된 단위 벡터
- $\hat{V}$: 표면점에서 카메라(관찰자)로 향하는 단위 벡터
-  $\alpha$: shininess(광택 지수). 클수록 하이라이트가 작고 날카로워짐
  alpha로 계수 표현해서 날카롭게 만듦

$$I = I_a + I_d + I_s$$ $$I = k_a\, i_a + k_d\, i_d\, \max(0,\ \hat{L} \cdot \hat{N}) + k_s\, i_s\, \max(0,\ \hat{R} \cdot \hat{V})^{\alpha}$$

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
### Shading


<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/vertexInterpolation.png" alt="GraphBasic" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>vertexInterpolation</strong> </td>  </tr> </table>
vtx기준으로 normal이 들어가는데, 그렇다면 Face에 있는 Normal은?


<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/Shading_models.png" alt="GraphBasic" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Shading_models(Wikimedia-commns)</strong> </td>  </tr> </table>

**Lambert**(flat Shading):
- 삼각형 당 한번만 조명 계산
- Face Normal 사용, 계산된 색 면 전체에 동일하게 적용
- hardenEdge처럼 끊겨서 보임

$$I_{face} = k_d \cdot i_d \cdot \max(0,\ \hat{L} \cdot \hat{N}_{face})$$


**Gouruad Shad** :
- Vtx마다 조명을 계산 (vertex Shader에서 수행)
- 계산된 색상을 삼각형 내부에서 **선형 보간**
	  - Scanline 기준으로 삼각형 두변 따라 보간, 또 한번더 가로방향으로 한번 더 보간
명암을 계산 다시 하자
- 선형 보간을 두번해서, 삼각형 내부에있어서 보간을 다 할수가있다
- 색이 부드럽게 이어져서 **매끄럽게** 나온다
- 단점 : 하이라이트(Specular)가 삼각형 내부에 있고, vtx에는 걸리지 않으면 **놓치거나 번져보임**


**Phong** :
- Specular의 highlight를 계산할 수있도록

- 보간해서 나온 친구를 한번 바로 normal 계산을 한번 더해서 
- 각 pixel별로 light source로 부터의 intensity 를 계산