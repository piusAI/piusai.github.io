---
layout: post
published: true
title: 『Efficient GPU Screen-Space RayTracing』
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-31 17:22:00 +0900
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
- 핵심 방법 : 원근 보정된 DDA Line Rasterization ALgorithm을 적절히 변형해, Depth Buffer를 이용한 효율적인 GPU 기반 3차원 Screen space RT를 구현한다.
- Linear Ray Marching과의 차이 : 연속적인 pixel 집합을 정확히 샘플링하므로, Oversampling 발생하지 않고, 효율적이다.

### 1. Introduction

#### 1.1 Ray Casting vs Ray Tracing
- 전통적 구분 :
  - **Ray Casting(이하 RC)** : 시점부터 광선을 쏴서 첫번째 교차점을 찾는 기본 연산
  - **Ray Tracing(이하 RT)**: Tunner Whitted가 정의한 개념, 반사 · 굴절을 위해 교차점에서 광선 재귀적으로 뻗어나감
- 실제 두 용어가 거의 혼용되어 사용됨, 보통 RT라는 표현을 더 대중적으로 쓴다

#### 1.2 다양한 Scene Data Structure와 하드웨어 환경
- RT의 성능은 어떤 Scene Data 구조를 썼는지에 따라 갈린다.
  - ex) 동적 삼각형 BVH, 암시적 표면, 밀집/희소 voxel, height map 등

#### 1.3 Depth Buffer RT인 이유?
- **장점** :  
  - 씬 복잡도를 자연스레 제한을 준다.
  - Rasterizer 과정의 부산물로, 효율적으로 빌드되어서, 동적 씬 대응 가능
  - Geometry data 직접 다루지 않고도, AO, Reflection, Refraction, Full Screen-Space PathTracing까지도 DepthBuffer만을 이용해 구현 가능
- **한계** : 
  - Depth 복잡도 너무 높은 영역
  - Viewport 바깥 영역
    $\rightarrow$ Depth Buffer만으로 표현 불가
- **해결책** : Guard Band를 두거나, 시야 frustum 바깥영역은 voxel grid 같은 data 구조로 Fallback하여 Trace 품질을 보완한다.

#### 1.4 Linear 3D Iteration(1.1)
Crytek Linear 3D Raymarching : Crysis3, JustCause2 같은 상용 대작 게임에서 거울 반사를 구현하기 위해 사용됨.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RayMarching3D.png" alt="RayMarching3D" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RayMarching3D</strong> </td>  </tr> </table>

위 그림과 같이 붉은 색 많을수록 과도히 샘플링된 영역, Hardware가 좋아져서 sample수를 억지로 늘리더라도, 기존 **3D space Linear Marching**방식에는 근본적인 구조적 결함이 있다.
- 3D Linear Marching ≠ Screen(2D) Linear Marching
  - 3D 공간을 직선으로 등간격 Sampling하더라도, 화면(2D)상의 Pixel grid위에서는 등간격으로 매핑되지 않는다.
- **문제점** : 
  - 어떤 Pixel들은 아예 건너뛰게 되어 반사되어야할 object를 놓치는 에러 발생한다.
  - Step 수 많이 늘리더라도, 일부 Depth Buffer Pixel은 여전히 놓치고, 다른 Pixel은 중복 샘플링 된다. → OverSampling으로 **GPU대역폭, ALU연산 낭비**

#### 1.5 Linear 2D iteration
2D화면 공간에서 Ray를 순회(permutation)할때 만족해야 할 네 조건:

|조건|설명|
|---|---|
|**연속성 (Adjacency)**|샘플링되는 Pixel들이 화면상 서로 인접(수평, 수직, 대각선)하도록 위치하며, 중간에 건너뛰는 Pixel이 없어야 한다.|
|**중복 제거 (Single Sampling per Pixel)**|각 Pixel을 최대 한 번만 샘플링하여 불필요한 연산 낭비를 없앤다.|
|**Frustum Clipping**|Ray가 View Frustum 바깥으로 벗어나는 부분을 깔끔하게 Clipping한다.|
|**GPU 자원 최적화**|Register 소비를 최소화하고, 분기 발산(Divergent Execution)과 무거운 연산을 줄여 현대 GPU 아키텍처에 친화적이어야 한다.|


- 고전적인 알고리즘 : Bresenham 알고리즘, DDA, Heightmap Permutation(순회)
- 고전적인 Rasterization 기법을 현대 GPU에 접목하면 반사 RT에 매우 적합!

---

### 2. Challenges of 2D Iteration (2)

- Ray Marching $\equiv$ 원근 보정 Line Rasterizer
	- 3D 공간상 Ray 2D 화면 공간에서 고유하고, 인접한 pixel들로 순회하는것은 View Volume 내부에 존재하는 Ray Segment를 **원근 보정하여 Rasterization하는것과 완전히 동일**하다
    - 따라서 Thin-Line Rasterization 알고리즘을 Ray Casting 목적에 맞게 변형, 적용한다.


#### 2.1 GPU에서 DDA가 효율적인 이유?
GPU 아키텍쳐에는 float 연산이 충분한 처리량 제공하며, DDA의 연산 구조가 GPU에 적합하다. DDA 내부 반복문은 대부분 다음과 같은 연산으로 구성된다.

$$x_{n+1}=x_n+\Delta x$$
 
$$y_{n+1}=y_n+\Delta y$$

단순한 덧셈 연산으로, GPU에서는 FMA(Fused Multiply - Add)와 같은 명령어 활용키 좋은 구조


#### 2.2 기존의 DDA가 GPU에 비효율적인 이유?

``` hlsl
if(delta.x < delta.y)
{
	// Path A
}
else
{
	// Path B
}
```

위처럼 일부 Thread는 PathA, 나머지는 Path B를 실행하면, GPU는 일반적으로 두 실행 경로를 각각 처리하면서 해당 경로에 참여하지 않는 **Lane을 비활성화** 한다.
- Path A 실행시 : B 해당 Lane 비활성화
- Path B 실행시 : A 해당 Lane 비활성화

실제 Ray를 2D 화면에서 어느 방향으로든 진행할 수있으니 아래와 같은 예외 처리가 필요하다.

1. 사분면에 따라 X, Y의 진행 방향 변경
2. Viewport 경계를 벗어나는 Ray Clipping
3. 시작점과 끝점이 같은 Single-Pixel Line처리

이렇듯 이론 자체로의 DDA 로직이 단순하지만, 실제 게임 엔진 환경에서는 이러한 예외 처리를 해야하는데, Branch 동반하기에 GPU 병렬 처리 떨어뜨린다.

---

### 3. Homogeneous Coordinate, Perspective Projection

3D Space위의 한점을 Q라 하자. 이 점을 View-Projection Matrix $M$으로 변환 하면

$$H=M \begin{pmatrix} Q\\ 1 \end{pmatrix}$$

여기서 $H$는 Homogenous Clip Space 좌표이다.

$$H=(H_x,H_y,H_z,H_w)$$

이후 Perspective Division을 수행하면 NDC 좌표를 얻을 수 있겠다.

$$Q_{\mathrm{NDC}}= \left( \frac{H_x}{H_w}, \frac{H_y}{H_w}, \frac{H_z}{H_w} \right)$$

Perspective Projection에서 $H_w$ 로 나누는 과정이 존재키 때문에 3D 공간의 균일한 간격이 View Space에서는 균일한 간격으로 표현되지 않겠다- [참고-Projection](https://www.tangramvision.com/blog/camera-modeling-pinhole-obsession?utm_source=chatgpt.com)
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/NDC001.png" alt="NDC001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>NDC</strong> </td>  </tr> </table>

이렇게 활용하기 위해서 NDC활용함.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/NDC002.png" alt="NDC002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>NDC 예시</strong> </td>  </tr> </table>
이 때문에, 2D 화면 Pixel을 따라 이동하며, 3D 위치를 복원하려면 Perspective Projection을 고려해야한다.

---
### 4. Perspective-Correct Interpolation

3D 공간 간격과 2D 화면 공간에서 균일하지 않으므로 다음과 같이 정의한다.

$$k=\frac{1}{H_w}, \qquad Q'= Qk = \frac{Q}{H_w}$$

$Q$와 $k$를 직접 선형 보간하는것이 아닌, **화면 공간**에서 $Qk$와 $k$를 선형 보간한다  
→ **Perspective Projection**을 거친 두 점 사이에서 Screen Space의 직선을 따라 선형적으로 보간 할 수 있다.  
보간 후 원래의 3D 좌표는 다음과 같이 복원한다.

$$\boxed{ Q(x,y)=\frac{Q'(x,y)}{k(x,y)} }$$

###### 왜 이렇게 복원 해야할까?  
화면상 픽셀 하나씩 Permutation(순회)하더라도, 해당 픽셀에서 실제 Ray가 위치하는 3D 공간상 좌표를 정확하게 계산해야 Depth Buffer와 교차 여부를 비교할 수 있기 때문이다.


#### 4.1 미분 값들이 Constant인 이유?

$$\frac{\partial(Qk)}{\partial x}, \quad \frac{\partial k}{\partial x}, \quad \frac{\partial(Qk)}{\partial y}, \quad \frac{\partial k}{\partial y}$$

위 값들이 Constant라는 것이 중요한데, 앞서 정의한 $Q_k$와 $k$는 화면 공간에서 Linear Interpolation 되는 값이다. 화면 공간의 직선 방향을 따라 변화율이 일정하다.

이를 이용하면 반복문안에서 매번 복잡한 Projection 연산 필요없이 미리 계산한 변화량을 이용해 다음 픽셀의 값을 갱신할 수 있다.

$$Q'_{n+1}=Q'_n+\Delta Q'$$

$$k_{n+1}=k_n+\Delta k$$

실제 3D 위치는 다음과 같이 복원한다.

$$Q_n=\frac{Q'_n}{k_n}$$

즉, **DDA 단순한 픽셀 순회 구조에 Perspective-Correct Interpolation을 결합**한 것이다. 이를 통해 화면 Space에서 연속적인 Pixel 순회하며, 각 픽셀에 대응하는 3D Ray의 위치를 계산할 수있다.

---

### 5. An Efficient GPU DDA Solution

#### 5.1 문제 : 8개 분기 케이스
DDA는 4개 사분면, 2개 순회 방향으로 8개 분기 케이스 존재 : GPU Divergence 유발한다.

#### 5.2 해결책 : Axis Permutation(축 치환), Step Direction

##### Step 1. Axis Permutation (축 치환)

``` hlsl
bool permute = false;
if(abs(delta.x) < abs(delta.y))
{
	permute = true;
	 delta = delta.yx;
	 P0 = P0.yx;
	 P1 = P1.yx;
}
```

기울기가 가파르면 X, Y를 교환해서 항상 X축 기준으로 순회토록 한다, DepthBuffer를 읽을때는 `permutate`값에 따라 좌표를 다시 원래 순서로 되돌린다.

##### Step 2. Step Direction

```hlsl
float stepDir = sign(delta.x);
```

진행 방향을 부호 처리후, 양방향, 음방향을 **같은 반복 구조로 통합**한다



---
### 6. DDA Line Rasterization Algorithm
DDA(Digital Differential Analyzer) : 두 점 사이의 직선을 pixel단위로 근사해, 화면에 그리는 Graphics 알고리즘

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/DDA_Basic.svg" alt="RefractionMask" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>DDA Basic</strong> </td>  </tr> </table>

시작점 $P_0=(x_0,y_0)$, 끝점 $P_1=(x_1,y_1)$이 주어졌을 때, 두 점 사이의 좌표 변화량을 계산하고 조금씩 증가시키면서 픽셀을 선택한다.

$$\Delta x=x_1-x_0,\qquad \Delta y=y_1-y_0$$
이때 필요한 스템 수는 두 변화량 중 큰 값.


$$\text{steps}=\max(|\Delta x|,|\Delta y|)$$

각 Step에서 증가시킬 좌표 값:  

$$x_{\text{inc}}=\frac{\Delta x}{\text{steps}},\qquad y_{\text{inc}}=\frac{\Delta y}{\text{steps}}$$

기울기가 가파르든 완만하든 인접한 픽셀 선택하며 직선 그릴 수 있다.




---
### 7. 마무리

#### Summary

| 항목       | Linear 3D Marching | 2D DDA 기반 Marching               |
| -------- | ------------------ | -------------------------------- |
| 샘플링 공간   | 3D 등간격             | 화면 Pixel 단위                      |
| Pixel 누락 | 발생                 | 없음 (인접성 보장)                      |
| 중복 샘플링   | 발생 (Oversampling)  | 없음 (Pixel당 1회)                   |
| 3D 위치 복원 | 직접 계산              | $Q'/k$ 로 Perspective-Correct 복원  |
| GPU 친화성  | 낮음                 | Axis Permutation으로 분기 제거, FMA 중심 |


#### 참고 및 KEYWARD

- FMUL : Fused Multiply-add 명령어가 GPU연산이 더 좋다

- GPU 프로그래밍 모범 사례
	- **분기 제거** : 조건문 대신 조건부 대입 (Conditional Assignment)을 사용해 워프 발산 방지
	- **연산 최적화** : 비싼 나눗셈 대신에 미리 계산된 역수와의 곱셈을 이용해, 정수연산보다 빠른 float 연산 적극 활용
	- **명령어 캐시 최적화** : 내부 로프 안의 명령어 수 최소화, FMUL 연산 선호토록 구성
	- **Register 최소화** : peak 레지스터 사용량 줄여 동시성 높힘

- Morgan McGuire, Michael Mara, **Efficient GPU Screen-Space Ray Tracing**, Journal of Computer Graphics Techniques, Vol. 3, No. 4, pp. 73–85, 2014.
- JCGT: [https://jcgt.org/published/0003/04/04/](https://jcgt.org/published/0003/04/04/)
- Full Paper: [https://jcgt.org/published/0003/04/04/paper.pdf](https://jcgt.org/published/0003/04/04/paper.pdf)