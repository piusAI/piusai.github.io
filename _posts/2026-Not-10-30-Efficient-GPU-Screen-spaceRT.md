---
layout: post
published: false
title: 『Efficient GPU Screen-Space RayTracing』
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-29 17:22:00 +0900
description: Refraction
categories:
  - Graphics
tags:
  - Graphics
use_math: true
author: PIUS
---
Depth Buffer를 이용한 Screen Space 3D RT를 위한 GPU 솔루션

참고 : [GPU Screen-Space RT](https://jcgt.org/published/0003/04/04/)

---
### Abstract
- 핵심 방법 : 원근 보정된 DDA Line Rasterization ALgorithm을 적절히 변형해, Depth Buffer를 이용한 효율적인 GPU 기반 3차원 Screen space RT를 구현한다
- Linear Ray Marching과 달리 연속적인 pixel 집합을 정확히 샘플링해, Oversampling발생x 효율적

### Introduction

#### 1. Ray Casting vs Ray Tracing
- 전통적 구분 : Ray Casting(이하 RC)은 시점부터 광선을 쏴서 첫번째 교차점을 찾는 기본 연산인 반면, Ray Tracing(이하 RT)는 Tunner Whitted가 정의한 반사 굴절을 위해 교차점에서 광선 재귀적으로 뻗어나가는 개념
- 용어가 거의 혼용되어 사용됨, 보통 RT라는 표현을 더 대중적으로 쓴다

#### 2. 다양한 Scene Data Structure와 하드웨어 환경
- RT의 성능은 어떤 Scene Data 구조를 썼는지에 따라 갈린다. - ex) 동적 삼각형 BVH, 암시적 표면, 밀집/희소 voxel, height map 등

#### 3. Depth Buffer RT인 이유?
- 장점 
  - 씬 복잡도를 자연스레 제한을 준다.
  - Rasterizer 과정의 부산물로, 효율적으로 빌드되어서, 동적 씬 대응 가나ㅡㅇ
  - Geometry data 직접 다루지 않고, AO, Reflection, Refraction, full Screen-Space PathTracing까지도 DepthBuffer만을 이용해 구현 가능
- 한계 : Depth 복잡도 너무 높은 영역, Viewport 바깥 영역은 Depth Buffer만으로 표현 불가
- 해결책 : Guard Band를 두거나, 시야 frustum 바깥영역은 voxel grid 같은 data 구조로 Fallback하여 Trace 품질을 보완한다.

#### 4. Linear 3D Iteration(1.1)
Crytek Linear 3D Raymarching : Crysis3, JustCause2 같은 상용 대작 게임에서 거울 반사를 구현하기 위해 사용됨.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RayMarching3D.png" alt="RayMarching3D" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RayMarching3D</strong> </td>  </tr> </table>

위 그림과 같이 붉은 색 많을수록 과도히 샘플링 된것.
Hardware가 좋아져서 sample수를 억지로 늘리더라도, 기존 **3D space Linear Marching**방식에는 근본적인 구조적 결함이 있다.
- 3D Linear Marching Screen(2D) Linear Marching
  3D 공간을 직선으로 등간격 Sampling 하는것은 화면(2D)상의 Pixel grid위에서는 등간격으로 매핑되지 않는다.
- 문제점 : 
  - 어떤 픽셀들은 아예 건너뛰게 되어 반사되어야할 object를 놓치는 에러 유발
  - Step 수 많이 늘리더라도, 많은 Depth-buffer 픽셀 마찬가지로 놓쳐짐 -> OverSampling으로 GPU대역폭, ALU연산 낭비



#### 5. Linear 2D iteration (1.2)
- 네가지 조건
	- 연속성(Adjacency) :  샘플링 되는 pixel들이 화면상 서로 인접(수평,수직,대각선)하도록 위치, 중간에 건너뛰는 픽셀 없도록
	-  중복 제거(Single Sampling per Pixel) : 각 픽셀 최대 한번 샘플링, 불필요한 연산 낭비 없음
	- Frustum Clipping :  Ray가 ViewFrustum 내부를 벗어나는 부분을 깔끔히 Cliping
	- GPU 자원 최적화 :  Register 소비 최소화, 분기 발산(Divergent Execution)과 무거운 연산 줄여 현대 GPU 아키텍쳐에 매우 친화적

- 고전적인 알고리즘 : Bresenham 알고리즘, DDA, Heightmap Permutation(순회)
- 현대 GPU에 접목해, 고전적인 Rasterization 기법이 반사 RT에 매우 적합!

- Ray Marching $\equiv$ 원근 보정 Line Rasterizer
	  - 3D 공간상 Ray 2D 화면 공간에서 고유하고, 인접한 pixel들로 순회하는것
	    -> View Volume 내부에 존재하는 Ray Segment를 **원근 보정된 Rasterization하는것과 완전히 동일**
    - Thin-Line Rasterization 알고리즘을 Ray Casting 목적에 맞게 변형, 적용

float 기반 DDA로 GPU연산에 더 좋도록?
이론자체는 DDA 로직이 단순하지만, 실제 게임 엔진 환경에서는 예외 처리를 해야하는데, Branch 동반하기에 GPU 병렬 처리 떨어뜨림.


단순 선형 보간이 아닌, 원근 보정이 필요한 이유?

$$k=\frac{1}{w}, \qquad Q'=Qk$$

화면 공간에서 $Qk$와 $k$를 선형 보간하고,

$$Q=\frac{Q'}{k}$$
로 다시 복원 한다.

### An Efficient GPU DDA Solution

- 8개 사분면의 통합
  - $P$ : DDA는 4개 사분면, 2개 순회 방향으로 8개 분기 케이스 존재 : GPU Divergence 유발

- 해결책 : Axis Permutation(축 치환), Step Direction
  - $s01$ . Axis Permutation

``` hlsl
if(abs(delta.x) < abs(delta.y))
{
	 delta = delta.yx;
	 P0 = P0.yx;
	 P1 = P1.yx;
}
```

기울기가 가파르면 X, Y를 교환해서 항상 X축 기준으로 순회토록!

- $s02$ . Step Direction

```hlsl
float stepDir = sign(delta.x);
```

진행 방향을 부호 처리후, 양방향, 음방향을 같은 반복 구조로 통합

- GPU 프로그래밍 모범 사례
	- 분기 제거 : 조건문 대신 조건부 대입 (Conditional Assignment)을 사용해 워프 발산 방지
	- 연산 최적화 : 비싼 나눗셈 대신에 미리 계산된 역수와의 곱셈을 이용해, 정수연산보다 빠른 float 연산 적극 활용
	- 명령어 캐시 최적화 : 내부 로프 안의 명령어 수 최소화, FMUL 연산 선호토록 구성
	- Register 최소화 : peak 레지스터 사용량 줄여 동시성 높힘

---
### DDA Line Rasterization Algorithm
DDA(Digital Differential Analyzer)

점사이의 직선을 pixel단위로 근사해, 화면에 그리는 Graphics 알고리즘

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/DDA_Basic.svg" alt="RefractionMask" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>DDA Basic</strong> </td>  </tr> </table>

시작점 $P_0=(x_0,y_0)$, 끝점 $P_1=(x_1,y_1)$이 주어졌을 때, 두 점 사이의 좌표 변화량을 계산하고 조금씩 증가시키면서 픽셀을 선택

$$\Delta x=x_1-x_0,\qquad \Delta y=y_1-y_0$$
이때 필요한 스템 수는 두 변화량 중 큰 값.


$$\text{steps}=\max(|\Delta x|,|\Delta y|)$$

각 스텝에서 증가시킬 좌표 값:  

$$x_{\text{inc}}=\frac{\Delta x}{\text{steps}},\qquad y_{\text{inc}}=\frac{\Delta y}{\text{steps}}$$

기울기가 가파르든 완만하든 인접한 픽셀 선택하며 직선 그릴 수 있음.


### Linear Ray Marching
광선의 시작점에서 일정한 간격으로 이동하면서 각 위치에서 표면이나 물체와 만나는지 검사하는 방법.

광선의 시작점$P_0$에서 방향 $\mathbf D$를 따라 일정한 거리 $\Delta t$만큼 이동

$$P(t)=P_0+tD$$

$$t_{i+1}=t_i+\Delta t$$

- $P_o$ : 광선의 시작점
- $\mathbf D$: 광선의 진행 방향
- $\mathbf p(t)$ : 현재 광선의 위치
- 진행 거리


|구분|DDA Line Rasterization|Linear Ray Marching|
|---|---|---|
|목적|화면에 직선 픽셀 출력|광선 경로상의 표면 탐색|
|이동 방식|픽셀 좌표 증가|광선 방향으로 일정 거리 이동|
|검사 대상|픽셀 좌표|깊이 버퍼, SDF, 볼륨 등|
|종료 조건|끝점 도달|교차 검출 또는 최대 거리 도달|
