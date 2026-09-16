---
layout: post
published: false
title: ROAM Algorithm이란?
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-08 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
Classic Tessellation Optimize방식, ROAM 알고리즘



<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://www.youtube.com/watch?v=dRh20M9mC5s" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/dRh20M9mC5s/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>UE ROAM Algorithm</strong>
    </td>
  </tr>
</table>

## ROAM ; RealTime Optimally Adapting Mesh
97년 Mark Duchaineau등이 발표한 대표적인 지형 LOD(Level of Detail) 알고리즘.
[ROAM Algorithm](https://cognigraph.com/ROAM_homepage), [ROAM 원문](https://www.classes.cs.uchicago.edu/archive/2003/fall/23700/docs/roam.pdf?utm_source=chatgpt.com) 

- **목적**
  1. 대규모 지형(Procedural Terrain)을 Rendering할때, 카메라시점 - 지형 굴곡에 따라 mesh 정밀도를 동적으로 최적화 하기 위함
  2. 카메라/오차에 따라 Terrain의 삼각형을 실시간으로 `Split/Merge`하면서 필요한곳 Subdivide하는 `View-Dependent Adaptive Mesh` 알고리즘
- 특징
  1. 카메라 가깝냐 / 지형 굴곡이 심한가 : Triangle의 Tessellation을 높은 밀도로 분할
  2. **무균열(Crack-Free / T-Junction Free)** 보장 기법을 통해 지형 구멍(Gap)이 생기지 않도록 완전한 Manifold-mesh를 생성!

### 1 ROAM 핵심 원리

#### 1.1 Binary Triangle Tree ; BTT 이진 삼각형 트리
ROAM은 삼각형을 두개의 직각 이등변 삼각형으로 나눈다.  
각 삼각형은 **빗변(Hypotenuse)의 Midpoint**를 기준으로 2개의 자식 직각 이등변 삼각형으로 이등분

```

       Apex (V0)                           Apex (V0)
         /\                                 / | \
        /  \         --- Bisect --->       /  |  \
       /    \                             /   |   \
      /______\                           /____|____\

  V1 (Left)   V2 (Right)             V1     Mid     V2
    (Hypotenuse)                        (Left) (Right)

```

- Runtime에 Triangle Tree를 Recursive하게 갱신, View-Dependent Mesh를 만든다

**분할 규칙**

  - 부모 삼각형: $(V_0, V_1, V_2)$
  - 빗변 중점: $V_{mid} = \frac{V_1 + V_2}{2}$
  - 왼쪽 자식 ($T_{left}$): $(V_{mid}, V_0, V_1)$
  - 오른쪽 자식 ($T_{right}$): $(V_{mid}, V_2, V_0)$
#### 다이아몬드 분할(Diamond split)과 크랙(T-junction)방지

#### 1.2 Keyword


```

     T-Junction 발생 (Crack)             Diamond 분할로 크랙 제거

         /\                                     /\
        /  \                                   /  \
       /____\   (인접 삼각형 미분할 시 틈 발생)  /____\
      /\    /\                               /\  | /\
     /__\  /__\                             /__\_|/__\

```


```
                ROOT
                 /\
                /  \
             SPLIT SPLIT
              /\     /\
             /  \   /  \
            ... ... ... ...

      ↓ Error / Camera / View

	       가까움 + 오차 큼
	              ↓
	            SPLIT

	       멀음 + 오차 작음
	              ↓
	             MERGE

	       Neighbor 관계
	              ↓
	       Crack 방지
```

- T Junction이 일어남 -> 이를 Diamond 동기화로 해결

#### 1.3 MainAlgorithm ; Priority기반 Greedy Refinement

- 위의 Diamond에 Priority 값을 매김
  ++ Camera - object 거리, View Frustrum안에 있는지 반영
- Split Queue에서 가장 prioirty 높은 (오차 가장 큰) 삼각형 꺼내서 Split
- Merge Queue에서 가장 Prioirty 낮은(합쳐도 티 안나는) Diamond 꺼내서 Merge
- 이 과정을 매 프레임 돌려 Pixel error 이내로 수렴할떄까지, Frame 시간이 다 될때까지 반복


ROAM의 목표 : 오차 기반으로 어디를 더 Subdivide할지, Merge할지를 priority를 매겨서 최적 근사 Mesh를 만들자!






## ChunkedLOD

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/TerrainOpti/clipmaps_01.jpg" alt="PS001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Geometry Clipmap</strong> </td> <td width="30%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/TerrainOpti/clipmaps_02.jpg" alt="PS002.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Terrian Render Geo Clipmap</strong> </td> </tr> </table>
**Streaming Scheduling**  
사용자가 이동중일때 필요한 Data가 Display Cache에 동적으로 load되어 화면이 렌더링되어야함.