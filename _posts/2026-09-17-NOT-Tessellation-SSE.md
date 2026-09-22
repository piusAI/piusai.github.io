---
layout: post
published: false
title: SSE Algorithm이란?
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-16 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
**SSE** : "현재 LOD가 원본/ 더 높은 해상도에 비해 얼마나 **Error**가 일어났는가?"


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://youtu.be/svTce_AMMVA" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/svTce_AMMVA/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>UE SSE Algorithm</strong>
    </td>
  </tr>
</table>

## SSE ; Screen-Space Error
97년 Mark Duchaineau등이 발표한 대표적인 지형 LOD(Level of Detail) 알고리즘.
[ROAM Algorithm](https://cognigraph.com/ROAM_homepage), [ROAM 원문](https://www.classes.cs.uchicago.edu/archive/2003/fall/23700/docs/roam.pdf?utm_source=chatgpt.com) 

- **목적**
  1. 대규모 지형(Procedural Terrain)을 Rendering할때, 카메라시점과 지형 굴곡에 따라 mesh 정밀도를 동적으로 최적화
  2. 카메라/오차에 따라 Terrain의 삼각형을 실시간으로 `Split/Merge`하면서 필요한 곳만 Subdivide하는 `View-Dependent Adaptive Mesh` 알고리즘
- 특징
  1. 카메라 가깝거나 지형 굴곡이 심한영역은 Triangle의 Tessellation을 높은 밀도로 분할
  2. **무균열(Crack-Free / T-Junction Free)** 보장 기법을 통해 지형 구멍(Gap)이 생기지 않도록 완전한 Manifold-mesh를 생성
  3. Frame to Frame Coherence를 활용해 매 프레임 Mesh를 처음부터 다시 만들지 않고, 직전 프레임 상태에서 필요한 부분만 incremental하게 갱신!
     이것이 **ROAM**을 Real-Time으로 만드는 핵심 요인!
  
### 1 ROAM 핵심 원리

#### 1.1 Binary Triangle Tree ; BTT 이진 삼각형 트리
ROAM은 삼각형을 두개의 직각 이등변 삼각형으로 나눈뒤, 각 삼각형을 재귀적으로 이등분 한다.

각 삼각형은 **빗변(Hypotenuse)의 Midpoint**를 기준으로 2개의 자식 직각 이등변 삼각형으로 분할

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
 
**다이아몬드(Diamond)**
빗변을 공유하는 두 삼각형을 합친 구조
```
	     /\  
        /  \ 
       /____\   
       \    /
        \  /
       　\/
```
다이아몬드는 BTT 트리 구조에서 "빗변을 사이에 두고 마주보는 두 삼각형 쌍"
이후 T-Junction 방지의 기본단위!
### 2 Diamond Split, T-Junctino 방지

#### 2.1 T-Junction이 생기는 이유???
삼각형을 독립적으로 분할하면, 인접 삼각형이 아직 분할되지 않을때 해상도 차이로 인해  
공유 Edge위에 T-Junction이 발생!

```

     T-Junction 발생 (Crack)             Diamond 분할로 크랙 제거

         /\                                     /\
        /  \                                   /  \
       /____\   (인접 삼각형 미분할 시 틈 발생)  /____\ ← 틈
      /\    /\                               /\ |  /\
     /__\  /__\                             /__\| /__\
												↑틈
```
한쪽 삼각형은 공유 Edge에 midpoint vertex가 생겨 edge가 꺾여있는데,
반대쪽 삼각형은 Edge를 여전히 하나의 직선으로만 취급함.  
이 어긋남 때문에 지형 높이차가 존재한다면, 시각적으로 Crack이 보이고, 평평한 경우에도 보간 불연속으로 seam이 생길 수 있다.

#### 2.2 Force Split (강제 분할)

ROAM은 이 문제를 **사전에 차단**하는 방식을 쓴다. 핵심은 "공유 Edge에 대한 분할 여부는  항상 두 삼각형이 동기화 되어야 한다"는 제약이다.
  1. 삼각형 $T$를 분할할 때, $T$의 빗변 이웃(Base Neighbor) $T_{base}$를 확인.
  2. 만약 $T_{base}$의 빗변 이웃이 $T$가 아니라면($T_{base}$가 더 낮은 레벨(Coarser)의 Sub-div level있는 경우), **$T_{base}$를 먼저 재귀적으로 분할(Force Split)** 하여 두 삼각형의 레벨을 맞춤
  3. 이과정 필요시 이웃 - 이웃까지 연쇠적(cascade)전파
  4. 두 삼각형이 함께 분할되고 자식 노드들의 이웃 포인터를 상호 연결함으로써 **크랙이 100% 제거**
 → 독립적으로의 Split막는것이 아닌 같이 연쇄되도록 하는것이 핵심
 
#### 2.3 Merge 조건(Split과의 비대칭성)
Merge는 Split과 반대 방향의 제약을 가진다.

- 한 다이아몬드의 두 자식(Child)삼각형이 모두 leaf(더 이상 하위로 분할 되지 않은 상태)일때만 그 다이아몬드는 merge 가능한 후보가 된다
- 한쪽만 Leaf - 다른쪽이 이미 더 쪼개있다면 Merge 불가
- Split처럼 상대를 강제로 끌고오는 cascade는 merge에는 없고, **조건이 충족될 때 까지 대기**하는 구조.
> Split은 강제전파(Force), Merge는 조건부 대기(Wait)
> 이 비대칭성이 diamond 구조의 핵심 동작 원리다.

### 3. Screen-Space Error Metric
###### 분할 여부 판단 기준
1. Geometry Variance / Curvature  | 지형의 굴곡/ 곡률
2. Camera Distance | 카메라-지형 거리
$$\text{Error} = \frac{(\text{Variance} \times W_v + \text{EdgeLength} \times W_e) \times C}{\text{Distance}(\text{Camera}, \text{Midpoint})}$$

-  **Variance ($\Delta h$)**: 빗변 중점의 실제 지형 고도(Sampled Height)와 두 양 끝점의 linear interpolate height간의 절대 차이:
  $$\Delta h = |H(V_{mid}) - \frac{H(V_1) + H(V_2)}{2}|$$

- **ErrorThreadshold** : 화면상 허용 오차 한계치,  
  이 값보다 크고 `Depth < MaxSubdivisionDepth`인 경우, 계속해서 하위 트리로 분할

###### Wedgie(웨지)
매 프레임 오차를 계산하기 위해 Sub-tree를 전체 순회하면 비용이 너무 크다.
이를 해결하기 위해 ROAM은 각 노드에 **Wedgie**를 저장한다.
- Wedgie는 "이 노드 이하로 앞으로 생길 수 있는 모든 자손 노드 들이 높이 편차를 감싸는 보수적(conservative) bounding volume" 이다.
- 각 노드가 자신의 하위 트리 전체에 대한 오차 상한을 미리 들고 있는 셈, Sub-Tree를 순회하지 않고도 O(1)에 가깝게 "이 아래를 더 쪼개야하는가 판단"
- Priority Queue갱신 비용을 실질적으로 낮추는 장치

### 4. MainAlgorithm ; Priority기반 Greedy Refinement
① 각 다이아몬드(혹은 Leaf Triangle)에 Priority 값을 매김
- Split Queue의 priority : 해당 노드를 쪼갤때 줄어드는 오차 - 자식 기준
- Merge Queue의 Priority : 해당 다이아몬드를 합칠 때 생기는 오차( 부모 기준)
- Camera-Object 거리, View Frustum 포함 여부도 반영(Frustrum 밖 노드는 우선수위 낮추거나 계산 자체 Skip)
② Split Queue에서 Priority 높은 (오차 가장 큰) 삼각형 꺼내서 Split
-  필요시 Force-split Casecade 발생
③ Merge Queue에서 가장 Prioirty 낮은(합쳐도 티 안나는) Diamond 꺼내서 Merge
- 두 자식 모두 Leaf인 경우만 후보
④ 이 과정을 매 프레임 돌려 Pixel 오차 이내로 수렴할떄까지, Frame 시간이 다 될때까지 반복
⑤ 다음 Frame에서 Tree를 처음부터 재구축 하지않고, **직전 프레임의 트리상태 이어받아** 카메라 이등분 만큼만 incremental하게 Split/Merge(Frame - to - Frame Coherence)

**ROAM의 목표** : 오차 기반으로 어디를 더 Subdivide할지, Merge할지를 priority를 매겨서 **시간 예산 안에서 최적 근사 Mesh를 만들자**!


### 5. 보조 최적화 기법
- Triangle Strip 생성: BTT 구조는 자연스레 긴 Triangle strip으로 직렬화 가능,
  Fixed-function 파이프라인 시절 GPU draw call 최적화에 유리, 그러나 현대 Programmable pipeline/Instancing 환경에서 중요도 많이 낮아짐
- View Frustum culling과 결합 : Frustum 밖 노드는 Priority 계산을 스킵 / 낮은 우선순위로 밀어 연산량 절감

--- 
## 6. 한계와 이후 흐름

- ROAM은 매 프레임 CPU에서 트리 순회 + heap(priority queue) 연산을 수행해야 하므로 **CPU-bound** 알고리즘이라는 근본적 한계가 있다.
- GPU 파이프라인이 프로그래머블해지고 병렬 처리 능력이 커지면서, 이후 **GeoMipMapping**, **Geometry Clipmaps**, **CDLOD(Continuous Distance-Dependent LOD)** 등 GPU-friendly한 지형 LOD 기법들로 대체되는 흐름이 생겼다.
- Unreal Engine 등 현대 엔진은 ROAM을 그대로 쓰기보다, Nanite처럼 완전히 다른 접근(virtualized geometry)을 채택하는 방향으로 발전했다.