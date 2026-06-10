---
layout: post
published: true
title: Houdini DegreeRadians
subtitle: DegreeRadians
date: 2024-03-14 15:45:00 +0900
description: Houdini DegreeRadians
categories:
  - Art
tags:
  - Houdini
  - Art
---

**이동 거리만큼의 θ**를 Rotate에 활용해봅시다!

---

## 이동 경로만큼의 회전값구하기
θ,**radians**에 대하여 궁금하시면 아래의 포스팅을 참고해주세요!

[Degree, Radians(θ)의 이해와 변환](https://piusai.github.io/art/2024/03/07/Houdini-DegreeRadians.html)

  
  

### 이동 거리, Translate X, set
#### Translate X 위치 이동
이동 거리 Translate의 X를 $T만큼 줍니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DistanceTheta/DT001.png" alt="DT001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
Expression을 활용하여 $T활용하였습니다.
자동적으로 호의 길이만큼의 회전을 활용해보겠습니다.

  

#### Rotate Z설정해주기
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DistanceTheta/DT002.jpeg" alt="DT002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

360 **°** : 2πr = θ : Trans.x  
=>360도가 2πr, 원의 둘레일때, **어떤 각도**가 Trans.x의 값일까요?

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DistanceTheta/DT003.jpeg" alt="DT003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
마지막으로 반지름을 구해줍니다.  
r= $SIZEY/2  
SizeY를 활용하여서 반지름을 확인했습니다.  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DistanceTheta/DT004.png" alt="DT004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
Rotate.z = -(180*ch("tx")) / ($PI*$SIZEY/2)
-를 붙힌 이유는 **뒤집혀 돌아가기 때문**입니다.
  

### 마무리

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DistanceTheta/DT005.gif" alt="DT005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

위와 같이 자동적으로 추적하여 Rotate 각을 찾았습니다.