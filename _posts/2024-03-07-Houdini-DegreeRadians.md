---
layout: post
published: true
title: Houdini DegreeRadians
subtitle: DegreeRadians
date: 2024-03-07 15:45:00 +0900
description: Houdini String Attribute Controll
categories:
  - Art
tags:
  - Houdini
  - Art
---
**θ,Vex에서 Radians**을 알아 봅시다.  

---

## Radians, Degree

먼저 Radians, Degree vex를 알아보기 전에 radians, **θ**가 하는 역할을 알아보겠습니다.

  

### 1Radians 구하기

우리가 알고있는 **Degree**는 일반적인 각도입니다.

#### θ ,1radians이란?


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DegreeRadians/DR001.jpeg" alt="DR001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

* θ는 호의 중심각  
* r은 반지름  
부채꼴을 둘러싼 SG호의 길이가 r일때, 1라디안을 구할 수있는데,

1라디안은, 180/𝝿 입니다.
우리의 Degree와 다르게 활용 하는데요.
gree를 θ로 바꾸어 주는 함수가 Radians입니다.

### Radians, degrees vex
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DegreeRadians/DR002.png" alt="DR002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
Radians Vex Function입니다.  
그렇다면 Raians을 Degree로도 변환해줘야겠죠
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/DegreeRadians/DR003.png" alt="DR003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

### 마무리

다음 포스팅은 이를 활용해 **부채꼴의 길이**를 측정해 활용해 보도록 하겠습니다.

