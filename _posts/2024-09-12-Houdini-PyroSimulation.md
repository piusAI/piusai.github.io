---
layout: post
published: true
title: Houdini PyroSimulation
subtitle: PyroSimulation
date: 2024-09-12 15:45:00 +0900
description: Houdini PyroSimulation
categories:
  - Art
tags:
  - Houdini
  - Art
---
Houdini Pyro simulation,  
꼭 필요한 **Field**와 **Work Flow**를 알아 봅시다!

---
## Pyro, 불 Simulation 웤플로우

Pyro와 같은 Simulation이 필요로하는 정보가 많습니다.

정확히 이해한다면, 원하는 샷을 만들 수 있기 때문에
꼭 필요한 Attribute(정보)를 체크 해보겠습니다
### 활용되는 중요 Field
field는 simulation상에서의 Attribute라 여기며 정확히 다룰 수 있어야 겠습니다.
#### 01 temperature
온도, 불덩어리가 뜨거워지도록, 출력 전체 규모 제어 가능.
#### 02 density
가장 직관적인 연기의 양
#### 03 flame
남은 Fuel과 같은 반응물의 수명을 저장.
그을음, 온도(Temperature)높이고, 팽창(Expansion)을 일으킬 수 있음
#### 04 burn
- 초기 폭발의 원천으로 활용됨
- 추후 flame의 source volume으로 활용하기도함.
#### 05 vel
원하는 방향이 있다면, Advection(이류)로도 활용됨
  

### Pyro WorkFlow

그렇다면 Dop내부에서 진행되는 순서는 어떨까요?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/PyroSimulation/PS001.png" alt="PS001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/PyroSimulation/PS002.png" alt="PS002.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
#### 01 **temperature**의 확산과 냉각
높은 온도 > 낮은 온도로 전달, Cooling은 복사 열 손실을 포착

#### 02 **density**, **flame**, temperature, vel의 이류 시작
힘을 어디로 갈지 Field로 판단함.
- burn > flame으로 받아서 온도를 높히기도 함.

#### 03 **flame** 의 수명 확인
fuel와 같은 반응물의 Life를 확인 한 이후에 추가 시뮬레이션

#### 04 density의 소산(dissipate)
density,연기의 밀도가 줄어들면서 사라짐.

#### 05 Simulation의 Container 크기 조정
resize gas field와 같이 bound box의 조절.

#### 06 Force,(힘) buoancy(부력),  viscosity(점성)
Force, Garvity나 Wind의 힘이 있다면 vel Field에 적용됨.
**Buoancy**, 상승하는 효과.
**Viscosity**, 꿀과 같은 점성을 연기에도 넣음.

#### 07 Shape 연산자
Turbulence(난류), Shredding(파쇄), Disturbance(교란)과 같은 모양을 만들어줌.

#### 08 Pressure
divergence(발산)을 시켜 빠른 확장을 일으 킬 수 있도록
목표 divergence(발산)에 의해 expansion, 축소 일치하는지 확인

### 마무리
Attribute나, Field를 영어로 알고있고 정확히 단어 뜻을 이해한다면,더 쉽게 이해할 수 있어 함께 작성했습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/PyroSimulation/PS003.png" alt="PS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="90%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/PyroSimulation/PS004.png" alt="PS004.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
**FireBall Shelf**로 확인하면서 공부하시면 더욱 눈으로 보면서 쉽게 확인 가능합니다~
[Understanding how pyro works](https://www.sidefx.com/docs/houdini/pyro/background.html)