---
layout: post
published: false
title: Concurrent Binary Trees
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-26 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
CBT ; Concurrent BinaryTree는 GPU에서 terrain등을 Adaptive Triangluation하기 위한 자료구조!

## Concurrent BinaryTrees

#### 00. Abstract
기존 Terrain Rendering 기법의 한계
- 확장 1 : Plane기준만이 아닌 임의의 폴리곤 메시로 확장하는 방법
- 확장 2: CBT로 인코딩 하는것이 아니라, CBT를 메모리 풀 관리자로 사용
   : Allocate - Release활용 가능
   : CPU에서 Quadtree를 구현하면 GPU 병렬성과 잘 맞지 않음
> Coarse mesh로부터 실시간 렌더링 할 수있다!!

#### 01.Introduction
##### 01 Motivation
- Clipmaps : 시점 고도에 따라 Balancing이 어렵다
- Projected Grids ; 움직이는 표면(물)에는 좋지만 Camera Flying Through동안에 static geometry에서 artifacts 발생!
- Adaptive Grids(QuadTree) : CPU에서 구현되거나 해상도 제한이 있다
##### 02 Contributions & Outline
- Section 02 : 기존 CBT에서의 Square뿐 아니라, HalfEdge Meshes 에 대해 Adaptive Triangle를 분할하는 ROAM과 유사한 알고리즘 설명
- Section 03 : CBT를 메모리 메니저로 사용하여 순차 알고리즘 병렬 구현
- Section 04 : 다양한 렌더링 결과, 자세한 성능 측정을 통한 접근법 검증
ROAM, CBT 알고리즘 개선으로 **Cm Detail**로 렌더링
- CBT : Triangle을 encoding!


#### 02.Section 02:ADPTIVE TRIANGULATION Algorithm
- 2.1 알고리즘 위치 제시
- 2.2 halfedge Operator사용한 Initialize
- 2.3 시간에 따른 Adaptive Triangle계산 

이등분 기반 subdivision 방식 HalfEdge Mesh 표현.


##### 2.1 배경
Half Edge Mesh사용, 
`Twin, Next, Prev Vert, Edge, Face`연산자

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/TerrainOpti/CBT/CBT_Figure01.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HalfEdge Mesh 표현</strong> </td>  </tr> </table>
Blue Half Edge : 일반 HalfEdge
Green Half Edge : Border HalfEdge

삼각형 이등분의 주요 어려움 : 임의 메시에 대해 초기화하는것이 `NP-Hard`로 알려져있다
> "Our Contribution in this particluar area is to show that the halfedges of the input mesh provide an intrinsic tirangluation that Provides a valid Initialization"

NP-hard한 초기화 문제를 input mesh가 이미 가진 half-edge구조 자체를 그대로 써, 별도 계산없이 풀어버렸다!

1. 매우 간단한 알고리즘
2. input Mesh Topology 보존 : Animation자산에 적합


#### 2.2 Root Bisector Vertices


<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/TerrainOpti/CBT/CBT_Figure03.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Overview Triangluation Method</strong> </td>  </tr> </table>

>halfedge $h_{7}$ in Figure 3 (a) maps to the root bisector $b^{0}_{7}$ in Figure 3 (b). Note that we refer to a bisector using the notation $b^{d}_{j}$, where 𝑑 ≥0 denotes its subdivision level and 𝑗 ≥0 its index.

(a) 에서 Root bisector $h_{7}$은 halfedge가 (b)의 $b^{0}_{7}$에 매핑,
$b^{d}_{j}$라고 표기하고, ($d$≥0) : Sub-div level, ($j$≥0) : index

---
##### HalfEdge `Vert`연산자 알아보기
- `Vert(h)`는 계산하는것이 아닌 조회하는 함수
- "halfedge `h`가 어느정점(vtx)에서 출발하는가"를 저장된 표에서 확인
- `Vert(h)`는 vtx를 가리키는 포인터, index buffer와 비슷
## Figure 2 halfedge 테이블
- h : halfedge ID
Figure2 Table : Next, Vert만 가져옴

| h        | 0   | 1   | 2   | 3   | 4   | 5   | 6   | 7     | 8     | 9   | 10  | 11  |
| -------- | --- | --- | --- | --- | --- | --- | --- | ----- | ----- | --- | --- | --- |
| Next     | 1   | 2   | 3   | 0   | 5   | 6   | 4   | 8     | 9     | 10  | 11  | 7   |
| **Vert** | 3   | 2   | 1   | 0   | 3   | 4   | 2   | **1** | **2** | 4   | 5   | 6   |


> :=   ; 정의한다


아래부터는 새롭게 정의되는 삼각형(Bisector) $T_0$ = { $v_0$, $v_1$, $v_2$ }
$$
v_0 :=VERT(h_7)
$$
- vert($h_7$ ) = 1
- $v_0$ : 메시 **정점 1번**
$$
v_1 :=VERT(NEXT(h_7)) = VERT(h_8)
$$
- next($h_7$) = 8
- Vert($h_8$) = 2
- $v_1$ = 메시 **정점 2번**
$$v_2 := \frac{1}{5}\big(\text{VERT}(h_7) + \text{VERT}(h_8) + \cdots + \text{VERT}(h_{11})\big)$$
- $h_7 \to h_8 \to h_9 \to h_{10} \to h_{11}$ 순서로 `Next`타고 한바퀴
- halfedge를 `Vert`Table에서 조회해 나온 vtx 좌표 모두 **더해 평균**
- face의 중심(centroid) $\to$ 삼각형의 세번째 꼭짓점 $v_2$
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/TerrainOpti/CBT/CBT_Figure01_01.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>오각형의 평균점을 구하는이유</strong> </td>  </tr> </table>


#### Algorithm 01 | Root Bisector Vertices

pseudu CODE :
>**function** RootBisectorVertices(halfedgeID: integer)
    nextID ← Next(halfedgeID)
    $v_0$ ← Vert(halfedgeID) // 시작점
	$v_1$ ← Vert(nextID)     // 다음 정점
    $v_2$ ← $v_0$               // centroid 누적 시작값 (초기화)
    n ← 1                 // centroid 누적된 정점 개수
    h ← nextID
    **while** h ≠ halfedgeID **do** // 자기 자신으로 다시 돌아오기전까지 (face 한바퀴)
        $v_2$ ← $v_2$ + Vert(h)     //  누적 평균을 구하기위한 합
        n ← n + 1             // n 각형?
        h ← Next(h)           // 다음 while문
    **end while**
    $v_2$ ← $v_2$ / n              //  합 → 평균 = centroid
    **return** `[v0, v1, v2]ᵀ`
**end function**


$v_{0}$ : HalfEdge 시작 vtx
$v_{1}$ : 다음 HalfEdge vtx
$v_{2}$ : while loop 안에서 Next타고 한바퀴 돌아 n(각형)개수로 나눔

> S-div 0 : 하나도 나누어지지 않은 VTX Bisector하기!
#### Algorithm 02 | Bisector Vertices

"While loop로 부모르 거슬러 올라가는 iteration 방식"


pseudu CODE :
> function BisectorVertices($b^{d}_{j}$ : bisector)
> 	halfedgeID ← j / $2^d$                  // root bisector index
> 	$M ← I$                                      // Identity(항등) 초기화
> 	$h ← j$                                       // j 인덱스 h에 넣고 시작
> 	*while* h ≠ halfedgeID **do**
> 		$b ← BitwiseAnd(h,1)$  // h 마지막 비트 (0 / 1)을 읽는다
> 		$M ← M * M_b$
> 		$h ← h/2$                          // 한 단계위 부모로 이동
> 	end while
> 	return $M * RootBisectorVertices(halfedgeID)$
> endfunction


 $b ← BitwiseAnd(h,1)$ : h의 가장 마지막 비트, 이단계에서 첫째 자식? 둘쨰 자식이었는지 확인
 - 자식 index $2j$(짝) :  마지막 비트 0 : 첫째자식
 - 자식 index $2j$ +1(홀):  마지막 비트 1 : 둘째자식
$$M_b = \begin{cases} M_0 & b=0 \text{ (첫째 자식)} \\ M_1 & b=1 \text{ (둘째 자식)} \end{cases}$$
 
내가 왼쪽 자식이었는지 오른쪽 자식이었는지 기록해서 행렬로 곱해서 쌓는 과정
> ↦ ; 찾는다

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/TerrainOpti/CBT/CBT_Figure03_01.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>이웃 삼각형</strong> </td>  </tr> </table>
 ❶ 다음 면조각
$Next(h_7) = h_8 ↦ b^0_8$
- $h_7$의 다음 h8에서 $b^0_8$ 삼각형

❷같은 면 내, 내 이전 조각
$Prev(h_7) = h_11 ↦ b^0_{11}$
- $h_7$의 이전 h1에서 $b^0_8$ 삼각형

❸같은 면 내, 내 이전 조각
$Twin(h_7) = h_1 ↦ b^0_1$
- $h_7$의 짝(반대 면 halfedge) h_1에서 $b^0_8$ 삼각형
- 변 사이에 두고 건너 편에있는 면 조각


---
##### Q1. NP-hard? 
풀 수는 있는데 문제 크기가 커지면 계산 시간이 감당 안될 정도로 폭발적으로 늘어나는 문제
ex) 오각형 하나를 삼각형 3개로 쪼개는 방법?? 
" 여러가지가 있을 텐데, 메시 전체에 대해 이 조건을 동시에 만족시키는 조합을 찾는것"

##### Q2. Bisection 기반 삼각 분할?
Adaptive Suvidision을 하면 새로운 Bisector가 필요해지는데
카메라 멀어지면 merge해서 필요없어지는 bisector들을 relase해야하는데
GPU에서 계쏙 일어남, CPU에서 관리하면 GPU-driven pipeline이 깨진다


---


#### 2.3 Progressive Subdivision Operators
ROAM분할의 의도처럼 Topology가 언제나 일치함



$$b^d_j  ↦b^{d+1}_{2j}  \text{ (첫째 자식), } \\  b^{d+1}_{2j+1}  \text{ (둘째 자식)} {cases}$$

- $b^1_{14}$가 $b^2_{28}$, $b^2_{29}$로 나뉘어진다

아래 보정 행렬에 따라 bisect(이등분선)을 두개의 새로운 이등분 선으로 분할
$$  
M_0 =  
\begin{bmatrix}  
1 & 0 & 0 \\  
0 & 0 & 1 \\  
\tfrac12 & \tfrac12 & 0  
\end{bmatrix}  
,\qquad  
M_1 =  
\begin{bmatrix}  
0 & 0 & 1 \\  
0 & 1 & 0 \\  
\tfrac12 & \tfrac12 & 0  
\end{bmatrix}  
\tag{1}  
$$
이 행렬을 $[v_0, v_1, v_2]^T$에 곱하면 : 각 행이 기존 정점들의 어떤 조합인지 나타냄:
$M_0$적용 :

$$  
M_0 \begin{bmatrix}v_0\\v_1\\v_2\end{bmatrix} =  
\begin{bmatrix}  
v_0 \\  
v_2 \\  
\tfrac{v_0+v_1}{2}  
\end{bmatrix}  
$$
-> 새 삼각형 꼭짓점 = {기존 $v_0$, 기존 $v_2$, ($v_0$과$v_1$의 중점)}

$M_1$적용 :
$$  
M_1 \begin{bmatrix}v_0\\v_1\\v_2\end{bmatrix} =  
\begin{bmatrix}  
v_2 \\  
v_1 \\  
\tfrac{v_0+v_1}{2}  
\end{bmatrix}  
$$
-> 새 삼각형 꼭짓점 = {기존 $v_2$, 기존 $v_1$, ($v_0$과$v_1$의 중점)}

세번쨰 행의 행렬은 같다 : $\left[\tfrac12, \tfrac12, 0\right]$ → 항상 "$v_0$와 $v_1$의 중점"을 계산'

이 알고리즘은 삼각형의 $v_0$-$v_1$ 변(edge)을 밑변으로 보고, 그 중점(midpoint) $m$을 새로 하나 찍어서, 꼭짓점 $v_2$(정점, apex)에서 그 중점까지 선을 그어 반으로 가른다

```
        v2 (꼭짓점/apex)
       /|\
      / | \
     /  |  \
    /   m   \      m = (v0+v1)/2  (새로 생긴 정점)
   /   /|\   \
  /   / | \   \
 v0 -----+----- v1
```

- **첫째 자식** ($M_0$): 왼쪽 삼각형 = $[v_0, v_2, m]$
- **둘째 자식** ($M_1$): 오른쪽 삼각형 = $[v_2, v_1, m]$

이렇게 밑변의 중점을 새 꼭짓점으로 세운다는 규칙을 모든 삼각형에 동일히 적용하면, 이웃 삼각형 끼리 쪼개지는 패턴이 서로 어긋나지 않고 맞물리게 만들 수 있음!



#### Algorithm 03 | Adaptive Refinement

"While loop로 부모르 거슬러 올라가는 iteration 방식"

pseudu CODE :
> procedure Refine($b_j$ : bisector)
> 	$b_k ← $Twin(b_j)$
> 	if $B_k ≠ null$ then
> 		if $Twin(B_k) ≠ b_j$ then
> 			$Refine(b_k)$
> 		end if
> 		$Split(b_j, b_k)$                                 // non-boundry
> 	else
> 		$Split(b_j)$                                     //boundry
> 	endif
> end Procedure







```
GPU
 ┌───────────────────────┐
 │ CBT Memory Pool       │
 │                       │
 │ [free][used][used]    │
 │ [used][free][used]    │
 │ [free][free][used]    │
 └───────────────────────┘
```
이것을 CBT로 관리!
