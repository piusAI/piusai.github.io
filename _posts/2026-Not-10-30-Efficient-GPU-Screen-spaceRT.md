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


Casting & Tracing 용오를 혼용한다.


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

광선의 시작점$P_0$에서 방향 $\mathbf d$를 따라 일정한 거리 $\Delta t$만큼 이동

$$\mathbf p(t)=\mathbf o+t\mathbf d$$

$$t_{i+1}=t_i+\Delta t$$

- $\mathbf o$: 광선의 시작점
- $\mathbf p(t)$: 현재 검사 위치


|구분|DDA Line Rasterization|Linear Ray Marching|
|---|---|---|
|목적|화면에 직선 픽셀 출력|광선 경로상의 표면 탐색|
|이동 방식|픽셀 좌표 증가|광선 방향으로 일정 거리 이동|
|검사 대상|픽셀 좌표|깊이 버퍼, SDF, 볼륨 등|
|종료 조건|끝점 도달|교차 검출 또는 최대 거리 도달|
