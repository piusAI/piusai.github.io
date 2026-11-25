---
layout: post
published: false
title: MatCut키워드
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-11-22 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
use_math: true
author: PIUS
---

#### Manifold
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/Manifold.png" alt="Manifold" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>

Proxy Mesh는 형상 분석을 위해서만 사용, 최종 분할은 원본 메시에서 진행

### 2. MAT/Q-MAT/SEG-MAT

- MAT: 물체 내부 중심 구조 표현
- Q-MAT : 중심축 변환 이차 오차 최소화 방식으로 계산 기술
- SEG-MAT : 중심축 구조 이용해 물체 부품으로 분할하는 기술

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/MAT.svg" alt="MAT" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>

SEG-MAT는 이런 중심축의 구조적 변화를 분석해서 부품 나누는 기술.
-> 막대기처럼 형상 동일, 재질만 바뀌는 물체는 구분하기 어렵기에,

MATCUT을 개발함.


### 3. SegFormer / Semantic Segmentation

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/SegFormer.svg" alt="SegFormer" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>
Segformer : 2D Img 에서 pixel단위로 클래스 예측하는 기존 Ai 모델

MATCUT : Apple DMS Dataset으로 사전학습된 SegFormer-B5를 사용.


---
#### 4. Multi-view Rendering / Face ID Buffer / UV Buffer

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/FibonacciSphereCam.svg" alt="FibonacciSphereCam" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>
24개 방향에서 `640 * 640` Resolusion Rendering
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="33.33%" style="text-align: center; border: none; padding: 5px;">
      <img src="/assets/postimg/Graphics/MatCut/RGBBF.svg" alt="RGB Buffer" style="width: 100%; max-width: 100%; height: auto;">
      <br><b>RGB Buffer</b>
    </td>
    <td width="33.33%" style="text-align: center; border: none; padding: 5px;">
      <img src="/assets/postimg/Graphics/MatCut/FaceIDBF.svg" alt="FaceID Buffer" style="width: 100%; max-width: 100%; height: auto;">
      <br><b>FaceID Buffer</b>
    </td>
    <td width="33.33%" style="text-align: center; border: none; padding: 5px;">
      <img src="/assets/postimg/Graphics/MatCut/UVBF.svg" alt="UV Buffer" style="width: 100%; max-width: 100%; height: auto;">
      <br><b>UV Buffer</b>
    </td>
  </tr>
</table>
- RGB Buffer : 화면에 보이는 색상
- Face ID Buffer : 픽셀이 어느 면에 속하는지
- UV Buffer : 픽셀에 대응하는 UV 좌표


### 6. Energy Function / Graph Cut


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/GraphCut.svg" alt="FibonacciSphereCam" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>

Graph로 표현한다면
- Node : 원본 Mesh Face
- Edge : 인접한 Face

1. Geometry Cost - 해당 Face가 어떤 형상 부품에 속하는것이 적합한가?
2. Material Cost - 해당 Face가 어떤 재질에 속할 확률이 높은가?
3. Pairwise Cost - 서로 붙어있는 두 Face에 다른 레벨을 부여할 만한 이유가 충분한가?

전체 에너지 표현 : 

$$
E = E_{geometry}+E_{material}+E_{cut}
$$

세 비용의 합이 낮아지도록 라벨 결정 : Graph Cut의 역할


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="33.33%" style="text-align: center; border: none; padding: 5px;">
      <img src="/assets/postimg/Graphics/MatCut/ExpansionKeep.png" alt="RGB Buffer" style="width: 100%; max-width: 100%; height: auto;">
      <br><b>ExpansionKeep</b>
    </td>
    <td width="33.33%" style="text-align: center; border: none; padding: 5px;">
      <img src="/assets/postimg/Graphics/MatCut/ExpansionCut.png" alt="ExpansionCut" style="width: 100%; max-width: 100%; height: auto;">
      <br><b>ExpansionCut</b>
    </td>
  </tr>
</table>

분할로 감소하는 비용이 절단 비용의 임계점을 넘어갔을때 Cut으로 분할한다.


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/CutCost.png" alt="CutCost" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>
- JSD : Jensen-Shannon Divergence

두 Face의 재질 확률 분포가 얼마나 다른지 측정하는데에 사용.  
재질 분포 차이가 크고, 신뢰도 높으면 두 절단 비용을 감소시킨다.

### IoU?
Intersection over Union
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Graphics/MatCut/IoU.svg" alt="IoU" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>
정답 영역과 예측 영역이 얼마나 겹치는가?

