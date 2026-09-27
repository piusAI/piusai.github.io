---
layout: post
published: false
title: Concurrent Binary Trees
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-27 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
[AnimationSandMud](https://arxiv.org/pdf/2302.08683)는 현 RenderTarget의 초석이되는 개념!

실시간 아님! 프레임 단위로 계산, 결과 저장후 렌더링하는 오프라인 batch 시뮬레인이었어서
## Animation Sand Mud Snow

#### 00. Abstract
시연을 하기위해 모래, 진흙, 눈속에서 달리는 사람, 타이어자국, 충돌 넘어지는 Character의 발자국을 보여준다. 세 표면 발자국 모양이 다르지만, 5가지 본질적으로 독립적인 매개 변수로만 제어

#### 01. 서론
정적인 풍경으로는 충분치 않다. 미묘한 움직임이 필요
애니메이션 플롯, 의도된 메시지에서 모순된 동작은 시청자 산만하게 만들 수도 있다.
애니메이션 원칙중 하나가 player가 없어서 허전해서 안된다.

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/tirepresence.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Tire Tire Presence On/Off</strong> </td>  </tr> </table>

> The Presence of tracks makes it clear that the ground is  soft sand rather than hard rock, and that other bikers have already passed through the area

자국의 존재로써 지면이 단단한 바위가아닌, 부드러운 모래임이 명확히 하고,
다른 바이커들이 이미 이 지역을 지나갔음을 보여준다

영화 감독 팀 아예 한 구성으로  "Continuity girl"과 같이 일관성 유지하는 책임 지는 팀도 있다.

애니메이션 결과 평과
- 인간 주자 비디오영상, 자전거 타이어자국, 추락하는 타이어, 넘어지는 주자


#### 02. 배경
배우 움직임에 따라 변화시킬수있는 환경상태에 집중하겠지만
배우움직임과 독립적으로 환경 일부 애니메이션 방법도있다.

> They developed a model of soil that allows interactions between the soil and the blades of diggin machinery.
> Soil Spread over a terrain is modeled using a **heightFIeld**, and soil that is pushed in front of a bulldozers' blade is modeled as **discrete chunks**

토양을 height field를 사용해 모델링하고, 
불도저의 칼날 앞에서 밀려나오는 토양은 이산/별개 덩어리로 모델링됨

서로 다른 규모의 현상 모델링하는데 초점 맞추어서, 실시간성을 위해 다양한 땅 재료 모델링하는데 더 외관 지향 접근법을 채택!

Related worked
흩날리는 나뭇잎, 물 애니메이션도 있고, 파동함수를 기반으로한 procedual modeling으로 해양 파도도 모델링한것도 있고, 물-모래 인터랙션도 좀 다르고, 물웅덩이 시뮬레이션, 식물이 시간이 지남에 따라 자라고 발달해야하는지의 식물모델 생성 기술도있고 등등

환경 요인이 물체 표면의 시간에 따라 어떻게 연속성있게 모델링해야하는지 시뮬레이션도 있다.

#### 03. Simulation of Sand, Mud, and Snow

material의 수직 기둥으로 정의된 height field 구성,
displacement 및 압축 알고리즘 사용해 단단한 Geometry가 지면 물질에 중돌시, 새엇ㅇ되는 변형을 animation, 타이어 자국, 지면의 다른 패턴을 만든다.
모래, 진흙, 눈과 같은 다양한 지면 material의 행동을 생성하도록 조정!


##### 3. 1. 지면 물질 모델
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/Heightfield.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeightField</strong> </td>  </tr> </table>

> Each Grid point within the height field represent a **vertical column** of ground material with the **top of the column** centered at the grid point
- Uniform grid의 grid point 마다 vertical Column(수직 기둥)이 하나씩 서있다 
- 기둥의 Top이 바로 grid point에 놓여있고, 꼭대기 높이가 곧 **Height field**이다.
추후 물체가 충돌하면 이 기둥들이 눌리거나, 밀려서 이동(displacement)하면서 발자국 모양이 생기는 원리이다.

연속 볼륨을 이차원 격자로 나누어서 **height field** 정의, 볼륨 표면 discrete(이산화)한다.
아래 그림은 1cm, 2meter의 사이클, 타이어 폭은 8cm.
격자 해상도가 작은 특징 결정하지만, 결과 모양에 현저히 영향미치지는 않는다.

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationPaper/SandMudSnow/GridResolution.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Grid resoution</strong> </td>  </tr> </table>
격자 각 점 높이에 대한 초기 조건은 절차적으로 생성, 다양한 소스로 가져올 수있음


##### 3.2 지표면 재료 운동
열 맨위에 의해 나타나는 Heightfield는 rigid Geometric object에 의해서 grid를 밀려들어가며 변형된다

Geomtry object : 신발자국, 자전거 바퀴, 인간 geometry
Rigid(단단한) Geometry 움직임 : 달리기, 자전거, 넘어지는 인간

지면 시스템은 움직임의 출처를 가리지 않는 범용 인터페이스라서, dynamic simulation / keyframe / motioncapture든 무엇이든 입력으로 받을수있다

기둥 height는 더이상 물체 표면에 침투 되지않을때까지 감소,
displacement된 material은 주변 기둥으로 밖으로 밀려난다

>A series of erosion steps are the performed to reduce the magnitude of the slopes between neighboring columns.

displacement로 재질 옆 기둥으로 밀려나면 경사(slope)가 갑자기 튈 수있는데, erosion으로 인접 기둥간 경사 완만하게 깎아주며 자연스레 만드는 후처리 한다.

분사 현상 모방을 위해. 입자 생성 된다

Collision, Displacement, erosion, Particle Generation | 충돌, 변위, 침식, 입자생성
알고리즘의 단계가 있다.

###### Collision - 충돌감지
- Ray Casting!
- 각 기둥마다 바닥에서 Top vtx를 향해 ray를 쏜다
- ray가 vtx에 닫기전에 rigid object(발, 타이어 등)과 교차하면 "침투했다"고 판단
- 침투했으면 기둥 top을 그 교차점(intersection point)높이로 끌어내림
- 어떤 기둥이 이동했다는것을 나타내기위해 flag설정 + 높이 변화 기록
-> AABB계층 구조로 rigid body의 폴리곤 분할! BVH의 초기형태

**Contour Map** 계산
- Displacement를 위한 사전 준비
- 충돌한 기둥들에 대해 Vtx Coloring 으로 계산
- 알고리즘:  충돌한 기둥 =0으로 시작->   이웃중 라벨 없는 기둥 : 인접한 라벨중 가장 작은값 + 1"
- 발자국 중심에서 바깥으로 갈수록 숫자가 커지는 **BFS distance Field**!

##### Displacement
$∆h = αm$
$m$ : Displaced material 눌러 없어져야할 총 재질량
$a$ : 압축 비율 ; 사용자가 조절하는 파라미터, "밀려난 재질중 몇% 가 압축되어 사라지고 몇% 가 옆으로 밀려나는가" 결정
$∆h$ : 실제로 옆 기둥들에 분배 되어야할 재질량

분배 방식:
- 압축되지 않은 나머지 재질은 Contour(외곽선) Map에서 값이 더 낮은 이웃들(충돌 지점에서 가장 가까운 충돌 아직 안한 고리) 에게 분배
- 발자국 바로 바깥쪽 "첫번째 링"의 기둥들 높이가 올라감

$a$가 크면 압축 재질이 사라지는 느낌 : 딱딱하게 눌리기만 하는 눈같은 재질
$a$가 작으면 많이 밀려나서 두둑하게 솟아오름 : 모래처럼 흙더미가 쌓이는 느낌
--> "모래, 진흙, 눈"을 가르는 5개 파라미터중 하나

##### Erosion
Displacement가 첫번쨰 Ring에만 재질을 몰아넣어서, 부자연스레 확 솟아오른 모야이된다.
-> 흙더미 모양으로 퍼트리는 단계가 Erosion

슬로프 계산
$$s = \tan^{-1}\frac{h_{ij} - h_{kl}}{d}$$
- 인접한 두 기둥 높이차를 거리로 나누어서 경사(각도)를 구함
--> 두 지점사이의 경사도

둘중 하나 Rigid object와 접촉중인 기둥이면 별도 임계값 $θ_{in}$ 사용
(발이 닿아있는 안쪽 경사는 독립적으로 조절 가능)


이동식 계산
$$\Delta h_a = \frac{\sum(h_{ij} - h_{kl})}{n}$$

- 경사가 가파른 n개 이웃 기둥들에 대해 높이차의 **평균**
- $σ$(fractional constant)를 곱해서 그 값만큼 이동
- 모든 slope가 $θ_{stop}$이하가 될 때까지 **반복**
-> 급격한 경사를 깎아서 완만하게 만드는 반복적 완화(relaxation) 알고리즘
voxel더미가 자연스럽게 흘러내리는것처럼 흉내내는 절차적 트릭

- erosion과정중 일부 기둥이 rigid object를 침투하게 될 수 도있어서  이 erosion은 다음 스텝에서 collision 단계가 다시 돌면서 교정!
프레임마다 반복되며 수렴해가는 구조.

