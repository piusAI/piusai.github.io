---
layout: post
published: false
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
Ray-Marching을 대체하는 새로운 Screen-Space 굴절 방식인 Newton방법을 사용하는 굴절 방식이다.

참고 : [Ultrafast Screen-Space Refractions and Caustics via Newton's Method](https://jcgt.org/published/0015/01/03/paper-lowres.pdf?utm_source=chatgpt.com)

---
### Abstract
- 핵심 방법 : Refraction된 Ray depth와 G-Buffer depth의 차이를 Fcuntion으로 정의하고, 현재의 G-buffer point의 Tangent plane을 이용해, intersection을 반복적으로 개선한다.
- Depth Discontinuities나, 잘못된 초기 추즉핱은 실패 사례분석, 감지하고, 완화, 해결하기위한 실용적인 전략 제안
- Refraction Ray에대한 접근방식으로 Screen-space Caustic의 계산 속도를 향상 시킴.

### 1. Introduction
유리/물같은 투명재질을 사실적으로 Rendering하려면 Refraction, Caustic을 시뮬레이션 하는것이 중요한데, Offline RayTracing(이하 RT)는 이러한 효과를 매우 정확히 모델링 가능하지만, 시각적 품질과 성능 사이 균형을 맞추기위해 근사법을 사용해야함.
- 일반적인 해결법 : Screen space에서 작업하는 DepthBuffer와 normal buffer를 이용해 화면에 보이는 Geometry만을 고려하는데 이를 통해 전체 Scene Geometry 몰라도 interfactive한 굴절효과, 근사적 Caustic 효과 구현 가능 [Efficient GPU Scree-Space Raytracing](https://piusai.github.io/graphics/2026/10/31/Efficient-GPU-Screen-spaceRT.html)
- RayMarching을 단계별로 진행하는 Hierarchical Search도 있지만, 단계별로 진행해야해서 bottle-neck현상이 남아있음.
- 잠재적인 해가 1차원 공간에 존재한다는 점을 활용하여 근 찾기 괴정을 최적화한다.
- Newton's iterative root-finding mothd(뉴턴 반복적 근 찾기 방법)으로 이차 수렴 특징으로 Screen-space 환경에서도 효율성을 높일 수있을것이라고 생각한다.

광선에서 교차점 탐색을

$$
\mathbf r(t)=\mathbf o+t\mathbf d
$$

-  **"어떤 $t$에서 광선이 표면과 만나는가?"** 
- 광선의 시작점 $o$에서 방향 $d$로  $t$만큼 진행한 3D 위치이다.

$$
\pi(\mathbf r(t))=(u,v)
$$

- 3D 위치를 화면에 Projection해서 **어느 Pixel에 해당하는지**를 구함
- $\pi$ : projection

$$
\boxed{ f(t)= \underbrace{z_{\mathrm{ray}}(t)}_{\text{광선 위 점의 깊이}} - \underbrace{z_{\mathrm{surface}}(u,v)}_{\text{그 픽셀의 표면 깊이}} }
$$

$$
\boxed{f(t)=0 \Rightarrow \text{화면 공간에서의 교차 후보}}
$$

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/SurfaceDepthRayDepth.png" alt="SurfaceDepthRayDepth" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SurfaceDepth와 RayDepth</strong> </td>  </tr> </table>
이렇게 $f(t)=0$ 라면, Ray Depth buffer의 표면이 같은 깊이에 있다는 뜻으로 교차점 후보가 된다.


Newton 방법의 기본 식 :

$$
t_{n+1} = t_n-\frac{f(t_n)}{f'(t_n)}
$$
현재 추정한 위치 $t_n$에서 오차 $f(t_n)$와 기울기 $f'(t_n)$를 보고 다음 위치를 정하는 것.

Hierarchical RayMarching을 Newton법으로 대채해 Ray 다음 단계 결정하는 실시간 Screen space 알고리즘 개발

#### 알고리즘 구체적인 형태

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/NewtonFigure1.png" alt="NewtonFigure1" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>NewtonFigure</strong> </td>  </tr> </table>
- $W$ : 굴절 광선이 출발하는 투명체의 Surface 지점
- $t$ : 굴절 광선 방향
- $p_i$ : G-buffer에서 Sampling해 얻은 실제 기록된 표면의 위치
- $\pi_i$ : $p_i$의 위치와 Normal로 만든 접평면
- $q_i$ : 굴절 광선과 접평면의 교차점

**(a) 초기화** : 배경 표면을 평면으로 근사
> G-buffer에서 $p_0$를 sampling하고, 굴절 Fragment $W$뒤에 접평면 $\pi_0$를 구성

**(b)** 광선과 평면을 교차시켜 다음 후보를 얻는다
> 굴절 광선 $\pi_o$의 교차점 $q_1$을 구한다. $q_1$을 화면에 투영하고, 그 화면 좌표에서 G-bbufer를 다시 샘플링하여 $p_1$를 얻는다.

$q_1$과 $p_1$이 휘어있거나 다른 물체가 있을떄 표면 위치 $p_1$과 다를 수있으니 $p_1$은 Ray위에 있다고 보장되어있지않음

**(c)** 두점이 충분히 가까워 질떄까지 반복한다
> $p_1$이 굴절 광선에 충분히 가까우면 수렴한것으로 판단, 그렇지 않으면 $p_1$에서 새 접평면 $\pi_1$을 만들고, $p_i\approx q_i$ 이 되거나, 최대 반복횟수에 도달할때까지 반복!

1. $P_0$의 접평면과 광선을 교차시켜 $q_1$계산
2. $q_1$의 화면 좌표에서 실제 표면 $p_1$확인 
3. 아직 차이가 크다면 $p_1$의 접평면으로 $q_2$계산
4. 다시 표면 $p_2$확인후 반복

-> 광선위 후보와 Buffer의 surface 위치가 일치하도록 계속 갱신한다



---
### 2. Our Method


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

---


- GPU 프로그래밍 모범 사례
	- **분기 제거** : 조건문 대신 조건부 대입 (Conditional Assignment)을 사용해 워프 발산 방지
	- **연산 최적화** : 비싼 나눗셈 대신에 미리 계산된 역수와의 곱셈을 이용해, 정수연산보다 빠른 float 연산 적극 활용
	- **명령어 캐시 최적화** : 내부 로프 안의 명령어 수 최소화, FMUL 연산 선호토록 구성
	- **Register 최소화** : peak 레지스터 사용량 줄여 동시성 높힘

- Morgan McGuire, Michael Mara, **Efficient GPU Screen-Space Ray Tracing**, Journal of Computer Graphics Techniques, Vol. 3, No. 4, pp. 73–85, 2014.
- JCGT: [https://jcgt.org/published/0003/04/04/](https://jcgt.org/published/0003/04/04/)
- Full Paper: [https://jcgt.org/published/0003/04/04/paper.pdf](https://jcgt.org/published/0003/04/04/paper.pdf)