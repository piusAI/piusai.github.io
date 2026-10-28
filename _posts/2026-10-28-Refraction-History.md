---
layout: post
published: true
title: Refraction 톺아보기 - 게시물 진행중
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-28 17:22:00 +0900
description: Refraction
categories:
  - Graphics
tags:
  - Graphics
use_math: true
author: PIUS
---
GameEngine에서 Effect를 제작하면서 Refraction을 그 토록 많이 활용했지만,

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/EffectRefraction.png" alt="EffectRefraction" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>EffectRefraction</strong> </td> <td width="38%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/RefracitonGFX.png" alt="RefracitonGFX" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RefracitonGFX</strong> </td> </tr> </table>

한 번씩 World의 object가 다 가려지고, World Scene의 Color가 굴절 `SceneColor`에 보인다던지, 캐릭터와 카메라 사이에 보여야 하는 굴절인데 또는 캐릭터가 가려지고 굴절이 나와야하는데 하는 경우들이 많았었는데, 이를 대응하기 쉽지 않았다. 이것은 굴절에 대한 정확히 어떤 RenderPipeline에 따라서 기술적인 대응을 못하고 있었던것 같다.  
하지만 DirectX에서 Back Raytrace로는 빠르게 그리고 clip space에 들어오기에 빠르지만, 굴절이 정확한 표현이 되지 않다, deferred Renderer에서 Refraciton이 표현되기 힘든 이유라던지. 

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/WaterRefraction.png" alt="WaterRefraction" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>WaterRefraction</strong> </td> <td width="55%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/IceRefraction.png" alt="IceRefraction" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>IceRefraction</strong> </td> </tr> </table>
위 [Ice GFX Portfolio](https://www.artstation.com/artwork/EzX3ln)를 제작시, Houdini에서 Flip Simulation / Destruction Simulation으로 가져온이후, Material Refraction시 놓쳤던 작업이었는데, 혹시 이 Refraction RenderPipeline을 이해한다면 확실히 이해하고, 오류를 해결할 수있지 않을까 하면서 Refraction Rendering이 연산하는 방법과, Unreal Engine이 채택한 Refraction기법과 논문으로 `.usf`, ComputeShader를 추가해서 정확한 굴절 표현이나, Shader 최적화던지 할 수있을것만 같다.

---
### 00. Refraction Rendering 흐름

| Time      | Summary                                      | Concept                                  |
| --------- | -------------------------------------------- | ---------------------------------------- |
| 1980      | Whitted 레이 트레이싱                              | 굴절/반사 레이를 재귀로 쏨. 정확하지만 느림                |
| 1996      | 포톤 매핑 (Jensen)                               | Caustic 까지 가능한 오프라인 GI                   |
| 2000년대 초  | 환경맵 기반 굴절                                    | 레이 추적 없이 환경맵 조회. 빠르지만 자기 가림, 주변 물체 반영 불가 |
| 2005      | Wyman 이미지 공간 굴절                              | 깊이 두 장으로 탈출점을 근사. 이 논문(Kim & Hong)의 계보   |
| 2005~2007 | Distance impostor, Caustics mapping, Eikonal | GPU 근사 방법들이 쏟아진 시기                       |
| 2010년대    | 스크린 스페이스 레이 마칭(SSR 등)                        | 깊이 버퍼를 레이 마칭. 게임 엔진에 정착                  |
| 2018~     | 하드웨어 레이 트레이싱 (RTX)                           | 실시간에서 정확한 굴절이 가능해졌지만 비용이 큼               |

### 01. Refraction Concept

#### 01-A. 스넬의 법칙 (Snell's Law)

스넬의 법칙 컨셉 :  

$$ n_i\sin\theta_i=n_t\sin\theta_t $$

- $n_i$ · $n_t$ : 입사 · 투과 매질의 굴절률(Refractive Index)
- $\theta_i$ : 입사각 (입사 벡터와 Normal Vector의 각도)
- $\theta_t$ : 반사각 ( 굴절 벡터와 Normal Vector의 각도)

- 입사 매질과 투과 매질의 굴절률 이용해 굴절 방향 $t$계산

$$\sin\theta_t=\frac{n_i}{n_t}\sin\theta_i$$

굴절 률의 비 :  

$$\eta=\frac{n_i}{n_t}$$

굴절률의 비를 활용한 식:

$$\sin\theta_t=\eta\sin\theta_i$$

#### 01-B. Refraction Dir 계산

$$\mathbf{T}=\operatorname{refract}(\mathbf{I},\mathbf{N},\eta)$$

- $I$ : 입사 방향
- $N$: 법선
- $\eta$ : 매질 굴절률 비율
`hlsl::refract()`도 이러한 방향 계산에 사용


#### 01-C. 화면 공간 샘플링

$$\mathbf{u}'=\mathbf{u}+\Delta\mathbf{u}$$

- $\mathbf u$ : 현재 화면 좌표
- $\Delta\mathbf u$ :굴절 방향과 깊이 등의 정보로 추정한 화면 좌표 Offset
- $\mathbf u'$ : 배경을 조회할 후보 좌표
$\rightarrow$ 이 방식이 빠르지만, 실제 굴절 경로에 대응 하는 `BG color`라는 보장은 없음

#### 01-D. 깊이 불연속성
: 인접한 Pixel의 깊이 차이를 단순화

$$D(\mathbf u)= \left|z(\mathbf u)-z(\mathbf u+\Delta\mathbf u)\right|$$

이 값이 크다면 두 Sample 사이에 depth경계 있을 수있음.
- 추가, Depth 좌표계, Surface Tangent, Sample간 거리 고려해야함

---

### 02. Approximate Interactive Refraction - WymanRefraction 

[Approximate Image-Space Approach for Interactive Refraction](https://cwyman.org/papers/sig05_approxISRefr.pdf)

##### Concept 
: 화면 좌표만 구하고 끝내는것이 아닌, Buffer에 저장

1. 교차점 $P_{1}$에서 굴절방향 $T_1$을 계산
2. 물체의 앞면과 뒷면 사이 거리 정보를 이용해 두번째 굴절이 일어날 위치 $\widetilde P_2$를 근사
3. $\widetilde P_2$를 screen space로 projection
4. 해당 좌표로 back buffer · Depth/Stencil buffer 조회하여 두번째 normal vector얻는다
5. 두번째 굴절방향. $T_2$를 계산

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/approxiFigure.png" alt="approxiFigure" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>approxiFigure</strong> </td>  </tr> </table>



두번째 굴절 위치는 다음과 같다:  

$$\widetilde P_2=P_1+\widetilde d\,\mathbf T_1$$

- $\widetilde d$ : 실제 광선 - 뒷면 교차 거리를 구하기보다, 쉽게 계산 가능한 두 거리 $d_v$, $d_N$을 보간해서 얻기

$$\widetilde d= \frac{\theta_t}{\theta_i}d_V+ \left(1-\frac{\theta_t}{\theta_i}\right)d_N$$
