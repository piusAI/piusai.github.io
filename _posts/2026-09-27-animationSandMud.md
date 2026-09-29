---
layout: post
published: true
title: 『Animation Sand Mud Snow(1999)』
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-27 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
[AnimationSandMud](https://arxiv.org/pdf/2302.08683)는 현 RenderTarget기반 지면 변형(Deformation) 기법의 초석이되는 개념

Sumner, O'Brien, Hodgins, *_Computer Graphics Forum_* Vol. 18 No. 1, 1999[arXiv:2302.08683](https://arxiv.org/pdf/2302.08683) 논문을 리뷰해보자.


실시간은 아니다. 프레임 단위로 계산 후, 결과 저장해 렌더링하는 Offline-Batch Simulation

--- 
## 0. Abstract
모래, 진흙, 눈 속에서 달리는 사람, 자전거 타이어 자국, 충돌·낙상하는 캐릭터의 발자국을 시연.
세 표면의 발자국 모양은 서로 다르지만, ***5가지의 본질적으로 독립적인 매개변수****만으로 제어된다.

## 01. 서론(introduction)
- 정적인 풍경으로는 충분치 않다. 캐릭터 행동에 반응하는 미묘한 환경 변화가 필요
- 애니메이션 플롯, 의도된 메시지에서 모순된 동작은 시청자 산만하게 만들 수도 있다.
- 이런 반응이 없으면 오히려 시청자가 "왜 없지? "하고 위화감을 느낌 - 애니메이션 원칙 상 의도치 않은 "결핍"은 산만함을 유발
- 영화 산업에도 장면간 일관성만 전담하는 아예 한 구성으로  "Continuity girl / floor secretary / second assistant director" 역할이 존재
- 컴퓨터 animation은 세계 통제를 할 수도 있지만, 반대로 장면마다 물체를 임의로 재배치하다 모순된 세계를 만들 위험도 도사림
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/tirepresence.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Tire Tire Presence On/Off</strong> </td>  </tr> </table>

> The Presence of tracks makes it clear that the ground is  soft sand rather than hard rock, and that other bikers have already passed through the area

"자국의 존재로써 지면이 단단한 바위가아닌, 부드러운 모래임이 명확히 하고, 다른 바이커들이 이미 이 지역을 지나갔음을 보여준다"

**평가 대상 결과물**
- 인간 주자 비디오 비교, 자전거 타이어자국, 추락하는 자전거, 넘어지는 주자

---
## 02. 배경(Background)
이 논문은 **배우 움직임에 반응해 변화하는 환경**에 초점을 두지만, 움직임과 독립적으로 환경 애니메이션 기법들도 관련 연구로 검토.

> They developed a model of soil that allows interactions between the soil and the blades of diggin machinery.
> Soil Spread over a terrain is modeled using a **heightFIeld**, and soil that is pushed in front of a bulldozers' blade is modeled as **discrete chunks**

토양을 height field를 사용해 모델링하고, 
불도저의 칼날 앞에서 밀려나오는 토양은 이산/별개 덩어리로 모델링됨

| 연구                             | 방법                                         | 비고                                                    |
| ------------------------------ | ------------------------------------------ | ----------------------------------------------------- |
| Lundin                         | 물체 밑면을 렌더링해 bump map 생성 → 지면에 적용           | 최초의 지면 발자국 애니메이션 사례. 물리 시뮬레이션이 아닌 가벼운 트릭              |
| **Li & Moshell** (가장 근접한 선행연구) | Height field + 밀려나는 흙은 discrete chunk로 모델링 | 불도저 블레이드-토양 상호작용. 물리 기반이지만 특정 동작(수평력에 의한 변위·미끄러짐)에 초점 |
| Chanclou, Luciani, Habibi      | 탄성 시트(elastic sheet)처럼 소성 변형               | 실제 지면 재질을 사실적으로 모델링하는 방법은 제시 안 함                      |
| Nishita et al.                 | Metaball 기반 눈 모델링·렌더링                      | 물체 위/옆 눈 쌓임, 다중 산란 렌더링                                |
  
- 본 논문은 Li & Moshell과 유사하지만, 서로 다른 규모의 현상 모델링하는데 초점 맞추어서, 실시간성을 위해 다양한 땅 재료 모델링하는데 더 외관 지향 접근법을 채택! — 다양한 지면 재질을 폭넓게 표현하기 위함

- 기타 관련 연구: 물/구름/기체, 불, 번개, 낙엽 애니메이션. 특히 물은 파동함수 기반 procedural 모델(Peachey, Fournier & Reeves) → shallow water equation 기반 일반화(Kass & Miller, 모래가 젖는 표현 포함) → 다른 물체와 상호작용하는 물 시뮬레이션(O'Brien & Hodgins) → Navier-Stokes 기반(Foster & Metaxas)으로 발전. 파티클 기반 물보라 모델도 다수 존재

- 식물이 환경과 상호작용하며 성장하는 grammar 기반 모델(Měch & Prusinkiewicz), 물체 표면이 시간에 따라 변화하는 시뮬레이션(Dorsey et al.)도 언급
---
## 03. Simulation of Sand, Mud, and Snow

Height field/ Verticle Column(수직 기둥) 기반 deformable 지면 모델.  
Displacement 및 압축 알고리즘으로 Rigid Geometry 충돌시 타이어 자국, 애니메이션, 파라미터 조정만으로 모래, 진흙, 눈과 같은 다양한 지면 material의 Behavior를 재현!

### 3. 1. 지면 물질 모델(Model of  Ground Material)
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/HeightFieldFigure.png" alt="Heightfield" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeightField</strong> </td>  </tr> </table>

> Each Grid point within the height field represent a **vertical column** of ground material with the **top of the column** centered at the grid point
- Uniform grid의 grid point 마다 vertical Column(수직 기둥)이 하나씩 서있dma
- 기둥의 Top이 바로 grid point에 정렬 되어있고, 그 꼭대기 높이가 곧 **Height field** 값.

추후 물체가 충돌하면 이 기둥들이 눌리거나, 밀려서 이동(displacement)하면서 발자국 모양이 생기는 원리이다.

- 연속 볼륨을 이차원 격자로 나누어서 **height field** 정의, 볼륨 표면 discrete(이산화)한다.

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/GridResolution.png" alt="GridResolution" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Grid resoution</strong> </td>  </tr> </table>
위 그림과 같이 격자 해상도가 표현 가능한 최소 Feature를 결정하지만, **전체적인 지형 모양 자체에는 큰 영향 없음**
- 초기 높이 조건은 절차적으로 생성 가능 (정수 격자 위 노이즈 + Catmull-Rom 스플라인 보간, 일종의 2D Perlin noise 변형) 또는 실제 지형 데이터·모델링 툴 출력·이전 시뮬레이션 결과를 활용 가능


### 3.2 지표면 재료 운동(Motion of the Ground Material)

기둥 Top으로 표현되는 Heightfield는 rigid Geometric object가 grid를 밀고 들어오면서 변형됨.

- 예시 Geomtry object : 러너의 신발 자국, 자전거 바퀴+프레임 , Rigging Human Model
- 이 Rigid(단단한) Geometry 움직임은  : 달리기, 자전거, 넘어지는 인간
- 이 rigid body들의 움직임은 **별도의 동역학 시뮬레이션(딱딱하고 평평한 바닥 위에서 달리기/자전거/낙상 계산)으로 만들어졌고, 그 결과(위치·방향의 trajectory)가 지면 시뮬레이션의 *입력*으로 들어감
- 이 입력은 범용적(generic)으로 설계되어 있어서, *동역학 시뮬레이션이 아니어도* 키프레임 애니메이션이나 모션캡처 데이터를 그대로 넣어도 동일하게 작동함 — 지면 시스템은 움직임의 출처를 가리지 않는 인터페이스
- 매 타임스텝마다 rigid object가 height field와 교차하는지 테스트 → 침투한 기둥은 더 이상 침투하지 않을 때까지 높이 감소 → 밀려난(displaced) 재질은 압축되거나 주변 기둥으로 밀려남

>A series of erosion steps are the performed to reduce the magnitude of the slopes between neighboring columns.

- 이후 인접 기둥 간 경사(slope)를 완만하게 만드는 **erosion**을 반복 수행 (displacement가 첫 번째 링에만 재질을 몰아넣어 생기는 부자연스러운 급경사를 완화)
- 마지막으로 rigid object의 접촉면에서 **Particle**을 생성해 충돌 시 흩날리는 재질(스프레이)을 모방

알고리즘은 **Collision → Displacement → Erosion → Particle Generation** 4단계로 구성.

#### Collision (충돌감지)
- **Ray Casting**:  각 기둥마다 바닥에서 Top vtx를 향해 ray를 쏜다
- ray가 vtx에 닿기 전, rigid object(발, 타이어 등)과 교차하면 "침투했다"고 판단 →   
  침투했으면 기둥 top을 그 교차점(intersection point)높이로 끌어내림
- 이동된 기둥에 flag설정 + 높이 변화 기록
- 성능 최적화 : rigid body polygon을 **AABB**계층 구조로 분할해 Ray-Polygon 교차 테스트 비용 절(BVH의 초기 형태)

**Contour Map 계산** (displacement를 위한 사전 준비)
- Vertex coloring 알고리즘으로, 충돌한 각 기둥에서 "충돌하지 않은 가장 가까운 기둥까지의 거리"를 계산
- 초기화: 충돌 안 한 기둥 = 0
- 반복: 라벨 없는 기둥은 인접한 라벨 중 가장 작은 값 + 1 부여
- 결과: 발자국 중심에서 바깥으로 갈수록 값이 커지는 ****BFS 거리 필드****
- 4-way(상하좌우) vs 8-way(대각선 포함) connectivity 실험 결과, **8-way가 더 부드러운 결과**를 준다고 밝힘 — 논문에서 실제 사용한 방식

#### Displacement (재질 재분배)
$∆h = αm$
- $m$ : 눌려서 없어져야할 총 displaced Material양
- $a$ : 압축 비율 - 사용자가 조절하는 파라미터.  
  "밀려난 재질중 몇% 가 압축되어 사라지고 몇% 가 옆으로 밀려나는가" 결정
- $∆h$ : 실제로 주변 기둥에 분배될 재질량

분배 방식:
- 압축되지 않은 나머지 재질은 Contour(외곽선) Map에서 값이 더 낮은 이웃들(=충돌 지점에서 가장 가까운, 아직 충돌 안한 고리) 에게 **분배** → 발자국 바로 바깥쪽 첫 번째 링 높이가 올라감
- $a$가 크면(≈1) : 대부분 옆으로 밀려남 → 눈처럼 압축 없이 수북히 쌓이는 느낌
- $a$가 작으면 : 상당 부분 압축되어 사라짐 → 모래쳐럼 다져지는 느낌
- 모래, 진흙, 눈을 가르는 5개 파라미터중 하나 (Displacement 알고리즘용)

#### Erosion(침식 - 경사 완화)
**목적** : Displacement가 첫 번째 Ring에만 재질을 몰아넣어 생기는 부자연스러운 급경사를, 자연스러운 흙더미 모양으로 완화 
-> 흙더미 모양으로 퍼트리는 단계가 Erosion

**슬로프 계산**
$$s = \tan^{-1}\frac{h_{ij} - h_{kl}}{d}$$
- 인접한 두 기둥의 높이차를 거리로 나눈 경사도- 두 지점사이의 경사도

**규칙**:
- slope가 임계값 $θ_{out}$보다 크면 → 높은 기둥에서 낮은 기둥으로 재질 이동
- 둘 중 하나가 rigid object와 접촉 중인 기둥이면 별도 임계값 $θ_{in}$ 사용 (안쪽 경사를 독립적으로 조절)


이동식 계산:
$$\Delta h_a = \frac{\sum(h_{ij} - h_{kl})}{n}$$
- 경사가 가파른 n개 이웃 기둥들에 대해 높이차의 **평균**

- 여기에 $σ$(fractional constant)를 곱해서 그 값만큼 이동 (한번에 다 옮기지 않음)
- 모든 slope가 $θ_{stop}$이하가 될 때까지 **반복**
→ 급격한 경사를 깎아서 완만하게 만드는 반복적 완화(relaxation) 알고리즘.  
흙 더미가 자연스럽게 흘러내리는것처럼 흉내내는 절차적 트릭

- erosion 과정 중 일부 기둥이 rigid object를 침투하게 될 수 있는데, 이는 **다음 스텝의 collision 단계**에서 교정됨 → 프레임마다 반복되며 수렴해가는 구조.

#### Particle Generation(입자 생성)
**부착량** :
- Rigid geometry의 지면과 닿은 각 삼각형이 접촉하는 동안 재질을 "묻힘"
- "묻힘" = 삼각형 면적 x 재질별 상수(**adhesion Constant**)

떨어지는 양 (지수감쇠)
$$\Delta v = v\left(e^{-(t - t_c)/h} - e^{-(t - t_c + \Delta t)/h}\right)$$
- $v$: 삼각형에 처음 묻은 재질 부피
- $t_c$ : 삼각형이 땅에서 떨어진 시각
- $h$ : *half-life* 파라미터 - 재질이 얼마나 빨리 떨어지는지
→ "묻은 흙이 시간이 지날 수록 지수적으로 떨어져나간다"는 물리적으로 그럴듯한 근사

$Δv$를 입자 하나의 $φ$부피로 나누면 : 이번 타임 스텝에 생성할 입자개수 $n=v/φ$


**초기 위치** ( barycentric coordinate 샘플링):  
난수 $\rho_a, \rho_b$로 barycentric weight(무게중심) $b_a, b_b, b_c$를 만들어 삼각형 세 꼭짓점의 가중합 → 삼각형 위 균일 분포 샘플링 표준 기법

**초기 속도** :
- 삼각형 위 균일하게 랜덤한 점을 뽑는 표준기법 - 두개 난수 $ρ_a, ρ_b$를 이용해 barycentric weight($b_a$, $b_b$, $b_c$)를 만들고, 이걸로 삼각형 세 꼭짓점의 가중합 구함
$$\dot{p}_0 = \nu + \omega \times p_0$$
- 물체의 선속도($v$) + 각속도($\omega$)에 의한 회전성분 ($\omega * P_0$, 즉 물체가 회전하면서 그 지점이 갖는 tangent velocity)
- 추가로 random Noise를 섞어서 더 자연스럽게 흩어지게 한다

**입자 생성 여부 판정(가속도 기반 확률)**
- 조건 : $(|ṗ_0|/s)^γ > ρ$
- 물체가 급격히 가속할수록 입자가 튈 확률이 높아짐( 급정거 / 급충돌시 흙이 더 많이 튄다)
- $s$ : 모든 후보가 무조건 떨어지는 최소 가속도
- $γ$ : 속도에 따른 확률 곡선의 형태 조절
- $ρ$ : 랜덤 값 
**시간 분산** : 입자를 타임 스텝 시작 시점에만 딱딱 생성하면 "얇은 판(sheet)처럼 뚝뚝 끊겨 보이므로", 생성 시각을 타임스텝 구간 내에서 랜덤 분산시키고, 해당 시점의 위치/속도를 보간

- 생성된 Particle은 중력 영향을 받아 낙하하며, 기둥의 표면에 닿으면 그 부피가 기둥에 더해짐.

## 3.3 구현 및 최적화 (Implementation and Optimization)
Terrain Simulation은 넓은 영역 다뤄야함() 해변 달리기, 눈위를 달리는 스키, 모래골짜기 동물떼 등).
  
전체 지형에 대한 단순한 구현은 메모리/계산요구로 실현 가능치 않을 것→ **활성 영역만 저장/시뮬레이션* + **병렬 처리**로 최적화.
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/activearea.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Active area - Figure 7</strong> </td>  </tr> </table>
아래 두절은, 활성부분 저장, 시뮬레이션, Compute Parallel해서 합리적인 성능 달성할수있는 최적화 방법 설명

### Algorithm Complexity
- 지면 모델 2차원 grid여서, 가장 단순한 구현은 2D 노드 배열
- i x j 격자라면 노드 수·연산 시간·메모리가 grid 수에 **선형 비례**해서 증가

**메모리 계산 예시**
- 10m x 10m 땅, 해상도 1cm -> 100x1000 grid : 100만개 노드.
- 노드당 10 byte : 10MB
이 작은 Patch도 상당한 Resource를 요구함!!

**Active area 결정 방식**:
- Rigid object들의 BBOX 약간 확대(enlarged)해서 지면 표면에 투영(project) -> 그 투영 범위 안의 노드들만 "active"마킹
- 이 Active 노드에만 Collision / Displacement / erosion 알고리즘 적용
- 노드는 전체 격자에 대해 미리 할당되는 것이 아니라, 필요(active)할때, 생성되고, ij위치를 **해시테이블 index**로 사용해 마치 단순 배열인 것처럼 알고리즘 구현
- 연산 시간이 전체 격자 크기가 아니라 **씬의 rigid object 크기에 대한 함수**가 됨 → 메모리 요구량 대폭 감소
- 단, 노드가 활성 상태를 벗어나도 **한번 변형된 노드의 상태는 계속 저장**해야 함 — 그렇지 않으면 같은 자리를 다시 지나갈 때 이전 발자국이 사라지므로. "한번 변형된 지면은 영구적으로 기억해야 한다."

### Parallel Implementation
- 활성 노드만 simulation 하더라도, 연산시간은 Rigid object 투영 면적에 따라 **선형 증가**(캐릭터 추가시 활성 영역이 대략 2배!)
- 캐릭터들이 **독립적인 지면 patch**와 상호작용시, 병렬처리로 연산 시간 절감 가능!

**구조** :  
- 부모 프로세스가 grid 상태를 유지·관리, 상호작용할 각 캐릭터마다 **자식 프로세스** 생성 (초기화 시)
- 부모-자식은 **UNIX socket**으로 통신, 단일 멀티프로세서 머신 또는 여러 단일 프로세서 머신에 분산 가능
- 각 자식은 다른 자식의 진행 상황과 무관하게 **최대한 빨리** 자기 캐릭터로 인한 grid 변화를 계산
- 한 타임스텝 계산이 끝나면 부모에게 보고 → 다음 스텝에 자기 캐릭터 bounding box에 새로 들어올 grid cell 정보를 대기

**동기화 문제** :  
- 자식 A가 다음 step 계산 준비가 끝났는데, A의 bbox안 cell에 대해 아직 자식 B가 변경 사항을 보고 하지 않았다면 : 부모가 A를 강제로 대기시킴
- 예: 사이클 리스트가 러너의 발자국 위를 지나가는 장면에서, 사이클 리스트 처리 자식이 그 발자국 위치에 먼저 도착했는데, 러너 처리 자식이 아직 발자국을 생성을 끝내지 못했을 수 있음  
  → 순서 꼬이지않도록 사이클리스트 쪽을 잠시 대기
- 두 캐릭터의 bbox가 같은 타임 스텝에 겹치면, 겹치는동안 **하나의 자식 process를 합쳐서** 계산(겹침이 풀릴때까지)


**성능 결과**:
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/CharacterSerialParallel.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Serial vs Parallel -Figure 8</strong> </td>  </tr> </table>

- Serial : 캐릭터 수에 따라 Linear 비례 증가
- Parallel : 이상적으로는 캐릭터마다 processor 하나씩이라 일정(Constant) 해야하는데, 통신 오버해드 + 캐릭터 간 상호작용(대기/동기화) 떄문에 증가하되, 단, Serial보다 훨씬 완만한 기울기!!

---

## 4. Animation parameters

**목적** :
Animator가 다양한 지면 재질의 다양성을 쉽게 만들어 낼 수있는 틀 제공.  
사용자가 조절가능한 5개 파라미터

| 파라미터          | 기호         | 역할                 | 알고리즘            |
| ------------- | ---------- | ------------------ | --------------- |
| Inside slope  | $θ_{in}$   | 물체와 접촉 중인 안쪽 경사 조절 | Erosion관련       |
| Outside slope | $θ_{out}$  | 흙더미 바깥쪽 경사 조절      | Erosion관련       |
| Roughness     | $σ$        | 표면의 불규칙성           | Erosion관련       |
| Liquidity     | $θ_{stop}$ | 재질이 얼마나 "물처럼" 흐르는지 | Erosion관련       |
| Compression   | $α$        | 재질이 압축되는 비율 (밀도감)  | Displacement 관련 |
- **Inside / Outside Slope** : 값이 작을수록 Erosion이 많이 일어나 완만한 경사.
- **Roughness** :   Erosion시 기둥 간 이동하는 재질 조절. (작으면 매끈한 흙더미)
- **Liquidity** :  타임 스텝당 erosion 반복 횟수 결정.( erosion적으면 : 표면이 물처럼 바깥으로 흘러나가는 것처럼 보임(liquid스타일) / erosion aksgdmaus : 빠르게 최종 형태로 수렴)
- **Compression** : 1이면 전부 옆으로 displace(밀려남), 1보다 작으면 일부 압축되어 사라짐 - 밀도가 다른 물질(가볍고 푹신함 vs 무겁고 잘 안눌리는 모래)

|        | Sand  | Mud  | Snow |
| ------ | ----- | ---- | ---- |
| θ_in   | 0.8   | 1.57 | 1.57 |
| θ_out  | 0.436 | 1.1  | 1.57 |
| σ      | 0.2   | 0.2  | 0.2  |
| θ_stop | 0.8   | 1.1  | 1.57 |
| α      | 0.3   | 0.41 | 0.0  |
관찰 포인트:
-  $1.57 \approx \pi/2$(90°) — mud·snow의 $θ_{in}, θ_{out}, θ_{stop}$이 거의 다 1.57: erosion을 거의 허용하지 않아 눌린 모양이 그대로 유지됨.  
- 반면 sand는 값이 작아 경사가 쉽게 무너지고 흘러내림 (모래의 실제 물성과 일치)

- snow의 $\alpha = 0.0$: 압축이 전혀 없음 
  → 모든 재질이 옆으로 밀려남. 발자국 옆에 눈이 수북이 쌓이는 특성을 반영
- sand의 $\alpha = 0.3$: 상당 부분이 압축되어 사라짐 (모래가 눌리며 다져지는 느낌)

**파티클 관련 추가 파라미터**: adhesion(부착력), particle size(입자 크기), fall-off rate(낙하 속도) 등
- 실제로는 sand 애니메이션에만 파티클 사용, mud·snow에는 미사용
- 이유: 뛰는 정도의 동작에서는 눈이 흩날리는 입자보다 **뭉쳐진 덩어리(clump)** 로 튀는 게  더 자연스러워 보인다고 저자들이 관찰. (스키처럼 더 역동적인 동작이면 snow도 스프레이를 일으킬 수 있다고 언급)

⚠️ RenderMan으로 오프라인 렌더링, 단일 폴리곤이 더 나은 결과