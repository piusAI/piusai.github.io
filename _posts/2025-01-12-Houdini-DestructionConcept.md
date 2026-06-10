---
layout: post
published: true
title: Houdini Destruction Concept
subtitle: Constraint
date: 2025-01-12 15:45:00 +0900
description: Houdini Destruction Concpet
categories:
  - Art
tags:
  - Houdini
  - Art
---

Houdini Destruction Concept !

---
## Destruction Insight01 - Sopsolver활용
Destruction 작업시, 보편적으로 활용하는 방법이 하나 있습니다.
그것은 Constraint을 조작하는 방법인데요,

### Destruction
Sopsolver에 Constraint의 Primitive를 삭제하는 방법입니다.

#### Destruction base scene setup
그대로 Conetwist의 constraint으로 설정 하였습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintDestruction/CD001.png" alt="CD001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Dopnet sim</strong> </td>  </tr> </table>
constraint network의 input3으로 sopsolver를 연결시켜줍니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintDestruction/CD002.gif" alt="CD002" style="width: 100%; max-width: 100%; height: auto;"> <br></td> </tr> </table>
그대로 Cone Twist constranit의 영향을 잘 받고 있습니다.

#### Sopsolver에서 Constraint 영향 지우기
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintDestruction/CD003.png" alt="CD003" style="width: 100%; max-width: 100%; height: auto;"> <br> </td>  </tr> </table>
f@angle attribute를 활용해서 몇도 이상 꺾일때 삭제하도록 합니다.
### 마무리
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintDestruction/CD004.gif" alt="CD004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Dopnet sim</strong> </td>  </tr> </table>

이렇게 간단하게 cone twist constraint을 angle이상이 될때 삭제하여 마무리 합니다.