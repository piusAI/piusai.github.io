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
Game Engine에서 Effect를 제작하면서 Refraction을 그토록 많이 활용했지만,

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/EffectRefraction.png" alt="EffectRefraction" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>EffectRefraction</strong> </td> <td width="38%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/RefracitonGFX.png" alt="RefracitonGFX" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RefracitonGFX</strong> </td> </tr> </table>

한 번씩 World의 object가 다 가려지고, World Scene의 Color가 굴절 `SceneColor`에 보인다던지, 캐릭터와 카메라 사이에 보여야 하는 굴절인데 또는 캐릭터가 가려지고 굴절이 나와야하는데 하는 경우들이 많았었는데, 이를 대응하기 쉽지 않았다. 이것은 굴절에 대한 정확히 어떤 RenderPipeline에 따라서 기술적인 대응을 못하고 있었던것 같다.  
하지만 DirectX에서 Back Raytrace로는 빠르게 그리고 clip space에 들어오기에 빠르지만, 굴절이 정확한 표현이 되지 않다, deferred Renderer에서 Refraciton이 표현되기 힘든 이유라던지. 

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/WaterRefraction.png" alt="WaterRefraction" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>WaterRefraction</strong> </td> <td width="55%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Refraction/IceRefraction.png" alt="IceRefraction" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>IceRefraction</strong> </td> </tr> </table>
위 [Ice GFX Portfolio](https://www.artstation.com/artwork/EzX3ln)를 제작시, Houdini에서 Flip Simulation / Destruction Simulation으로 가져온이후, Material Refraction시 놓쳤던 작업이었는데, 혹시 이 Refraction RenderPipeline을 이해한다면 확실히 이해하고, 오류를 해결할 수있지 않을까 하면서 Refraction Rendering이 연산하는 방법과, Unreal Engine이 채택한 Refraction기법의 이해와, 논문으로 `.usf`, ComputeShader를 추가해서 정확한 굴절 표현이나, Shader 최적화던지 할 수있을것만 같다.

---
### 00. Refraction Rendering의 계보

| Time      | Summary                                      | Concept                                  |
| --------- | -------------------------------------------- | ---------------------------------------- |
| 1980      | Whitted 레이 트레이싱                              | 굴절/반사 레이를 재귀로 쏨. 정확하지만 느림                |
| 1996      | 포톤 매핑 (Jensen)                               | Caustic 까지 가능한 오프라인 GI                   |
| 2000년초    | 환경맵 기반 굴절                                    | 레이 추적 없이 환경맵 조회. 빠르지만 자기 가림, 주변 물체 반영 불가 |
| 2005      | Wyman 이미지 공간 굴절                              | 깊이 두 장으로 탈출점을 근사. Kim & Hong             |
| 2005~2007 | Distance impostor, Caustics mapping, Eikonal | GPU 근사 방법들이 쏟아진 시기                       |
| 2010년대    | 스크린 스페이스 레이 마칭(SSR 등)                        | 깊이 버퍼를 레이 마칭. 게임 엔진에 정착                  |
| 2018~     | 하드웨어 레이 트레이싱 (RTX)                           | 실시간에서 정확한 굴절이 가능해졌지만 비용이 큼               |


---
### 01. Refraction Concept

#### 01-A. 스넬의 법칙 (Snell's Law)

스넬의 법칙 컨셉 :  

$$ n_i\sin\theta_i=n_t\sin\theta_t $$

- $n_i$ · $n_t$ : 입사 · 투과 매질의 굴절률(Refractive Index)
- $\theta_i$ : 입사각 (입사 벡터와 Normal Vector의 각도)
- $\theta_t$ : 굴절각 ( 굴절 벡터와 Normal Vector의 각도)

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

`hlsl::refract()`는 일반적으로 입사 방향 표면 쪽으로 향하는 벡터로 정의, normal vector는 입사 매질에서 투과 매질 쪽을 향하도록 사용
`refract()`는 전반사(TIR)일때, 0 vector를 반환


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

### 02 Paper Review

#### 02-A. Whitted Raytracing 

[Whitted Raytracing](https://www.cs.drexel.edu/~deb39/Classes/Papers/p343-whitted.pdf)

##### 1. 기본 개념
Camera → Surface Intersection → Ray Tracing → Shading → Pixel Intensity
카메라에서 광선 발사, 표면점과의 교차점에서 조명 및 반사 굴절 방향 계산.  
반사와 굴절 광선을 재귀적으로 추적하며, 각 교차점에서 계산한 빛의 기여도를 합산해 최종 픽셀 색상을 결정.

##### 2. Lambert's Cosine Law:  
$$I(\theta) = I_0 \cos\theta$$
- $I_0$ : 입사 광원의 세기
- $\mathbf{N}$ : 표면의 단위 법선 벡터
- $\mathbf{L}$ : 표면에서 광원 방향을 향하는 단위 벡터
- $\theta$ : 법선과 광원 방향 사이의 각도
표면이 광원을 정면으로 향할수록 빛을 많이 받고, 표면이 광원과 평행할 수록 확산 광의 기여도 감소
$\rightarrow$ 추후 dot product로 발전
##### 3.Phong illumination :  

$$I = I_a + k_d \sum_{j=1}^{ls} (\bar N \cdot \bar L_j) + k_s \sum_{j=1}^{ls} (\bar N \cdot \bar L_j')^n$$

Ambient + Diffuse + Specular 조합으로 표면의 기본적인 조명 결과를 계산.


##### 4. Whitted Ray Trace 의 반사 굴절 확장

개념적으로 최종 조명은 다음과 같다

$$I=I_{\mathrm{local}}+k_r I_R+k_t I_T$$

- $I_{\mathrm{local}}$: 표면에서 직접 계산한 조명
- $I_R$: 반사 방향으로 추적한 광선이 가져오는 빛의 기여도
- $I_T$: 굴절 방향으로 추적한 광선이 가져오는 빛의 기여도
- $k_r$, $k_t$ : 반사/투과 조절 계수

반사 광선과 굴절 광선은 새로운 교차점을 찾아 동일한 과정으로 재귀 수행.  
광선이 다른 물체에 도달하면 해당 지점의 조명 결과를 계산하고,   
그 결과를 이전 교차점의 조명 계산에 반영

##### 5. Ray Tree 구조와 교차 검사 최적화

반사 및 굴절 광선이 추가 광선을 생성하므로, ray의 추적관계는 tree로 표현 가능

- 노드(Node) : 광선과 표면의 교차점 및 해당 지점의 조명 계산
- 자식 광선(Child Rays) : 교차점에서 생성되는 반사 및 굴절 광선
- Bounding Sphere : 물체 둘러싸는 구를 이용해 ray - object 교차 가능성을 빠르게 검사. 구와 교차하지 않으면 더 복잡한 표면 교차 계산 생략 가능
- 광선 추적비용 : 반사/굴절이 재귀적 반복되면 ray수가 증가하므로, 재귀 depth 제한이나, 교차 검사 최적화 필요

BVH(Bounding Volume Hierarchy)와는 다름.  
물체별 경계 sphere를 사용하는 것은 개별 물체 교차 검사를 단순화 하기 때문이고, BVH는 여러 물체의 경계volume을 계층적으로 구성해, 탐색 비용 줄이는 별도 가속 구조.

핵심 : Whitted RT는 반사와 굴절 방향으로 광선을 재귀적으로 추적해, 실제 장면의 다른 표면에서 얻은 빛을 계산, 물리적인 광선 경로 직접 추적할 수있지만, ray와 교차 검사 비용이 증가하기에 실시간 렌더링에서는 계산 비용이 중요한 제약이 됨!

#### 02-B. Approximate Interactive Refraction - WymanRefraction 

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


#### 02-C. Generic Refraction Simulation

Concepts:  
- 빛이 튀는 모든 환경을 실행중에 계산하는 방식
- 환경 맵을 미리 구워 Runtime에 사용하는 방식
$\rightarrow$ 텍스쳐 메모리 사용량이 많아 성능 저하가 발생한다.

- Scene내 굴절되는 객체 이미지를 Color로 가져오는것이 아닌, Texture Coordinate를 변경하는 방식으로 표현!

현재 Back buffer를 굴절 맵으로 사용, Texture coordinate에 변위 더해 굴절 효과 Simulation하는 기본 기법


기본 굴절 기법 단계
1. 모든 굴절 Mesh 건너 뛰고 Scene geometry를 Texture로 렌더링
   후속 pass에서 Rendering될 굴절 객체 뒤에 보이는 객체를 결정하는데 사용 가능
   : 텍스쳐$S$.
2. 굴절 메쉬 Rendering, $S$에서 값 조회, 해당값에 Perturbation(교란) 적용

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RefractionMask.png" alt="RefractionMask" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RefractionMask</strong> </td>  </tr> </table>
앞 Geometry의 Perturbation을 받지 않도록 위와 가운데 이미지와 같이 FrameBuffer alphachannel로 잘라서 그 영역의 경우만 Refraction적용되도록 함

---
### 03. 현재 게임 엔진의 굴절 방식

#### 03-A. Unreal engine
기본 Screen Space 방식으로 Material Refraction 입력이 Normal, IOR로 화면 UV를 Offset해서, SceneCololr를 Sampling하는 구조, 정확한 굴절이 필요하다면, Path Tracer / HW RT 경로를 따로 쓴다.
TAA/MSAA과 같은 방식으로 추가 보정

#### 03-B. Unity engine HDRP
Refraction Model을 Planar, sphere, Thin으로 고른 후, Thickness 값으로 탈줄점 근사.  
Wyman의 "두께로 두번쨰 굴절 위치 추정"하는 발상이 단순화 하여 정착한 형태, 배경은 color 피라미드(블러 mip)에서 샘플링.


#### 03-C. 공통 한계

화면 밖 가려진 영역의 배경 쓸 수 없고, 투명체 끼리 겹치면 정확치 않고, 굴절 배경에 TAA를 걸면, Motion vector가 어긋난다, Screen Space Offset + 두께근사, RT Fallback
2Pass normal 조회는 잘 쓰이지않고, 아이디어만 흡수된 상태