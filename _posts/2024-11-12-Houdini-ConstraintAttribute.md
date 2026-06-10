---
layout: post
published: true
title: Houdini Constraint Attribute
subtitle: Constraint
date: 2024-11-12 15:45:00 +0900
description: Houdini Constraint
categories:
  - Art
tags:
  - Houdini
  - Art
---
Houdini Destruction  
Constraint Attribute 설정시, 제대로 설정되지 않은 경우가 많으셨죠?

가장 기초가되는 **Constraint Name, Constraint Type**에 대하여 알아봅시다!
---
## Vex, Point
Houdini destuction을 작업할때, 그냥 밋밋하게 떨어진다면 정말 재미없는 샷이 될것입니다.
재질 특성에 따라서 부서질때, 잡고있는 힘들이 다 다른데요,  
그 점을 표현해 낸다면 자유로이 작업 가능하겠죠?

그 부서지는 특성을 잘 컨트롤 하기위해서는, 여러가지 Attribute이 존재 할 것입니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA001.png" alt="CA001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Density attribute</strong> </td>  </tr> </table>
간단히 잡을 수 있는 RBD Object에서의 Density도 있지만,

이 포스팅에서는 **Constraint**으로 컨트롤 할 수있도록 제작 해볼까요~?

### Constraint의 필요와 기본 뼈대
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA002.gif" alt="CA002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>No Constraint</strong> </td>  </tr> </table>
Constraint을 잡지 않는다면, 이렇게 우두두둑 떨어지는게 당연하겠죠?

이러한 Constraint를 설정하기 위해서는 Dop내부에서는 두가지 Node를 이러한 뼈대로 활용합니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA003.png" alt="CA003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>No dopnet 내부에서의 뼈대</strong> </td>  </tr> </table>
연결 하기 위해서 이러한 노드를 설정해주셔야 합니다.
그리고 또 필요 한 것이 있습니다. (참 많죠..)

Sop 단계에서의 Constraint 두가지 Attribute가 필요한데요,
그것은 바로

### Constraint Attribute 2가지
#### 첫번째, constraint_name

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA004.png" alt="CA004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>No dopnet 내부에서의 뼈대</strong> </td>  </tr> </table>

> "  
> constraint_name은 특정 Constraint이 만들어질 이름.  
> attribute가 제대로 설정되지않으면, primitive에서 Constraint이 뜨지 않을 것이고,  
> Simulation중간에 constraint type을 sopsolver를 활용해서 수정할 수도 있다  
> "
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA005.png" alt="CA005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>No dopnet 내부에서의 뼈대</strong> </td>  </tr> </table>
Constraint_name을 입력해야지, 여러가지의 Constraint에서 활용 가능합니다.
Attribute 이름은 정확히,
**constraint_name**
입니다.

``` hlsl
s@constraint_name='ConRelGlue';
```

소대문자, 언더바 확실히 작성해주세요.
저도 처음에 작업시 굉장히 오류가 많이 떴던 곳중 하나입니다!

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA006.png" alt="CA006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="32.5%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA007.png" alt="CA007.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
물론 name string은 **glueconstraint**에서나, **hard constraint**에서나 맞춰줄 수 있습니다.
그렇기 때문에 작업 중에 수정도 가능하겠습니다.
#### 두번째, Constraint_type
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA008.png" alt="CA008" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>
constraint_name과는 다르게 Houdini에서 이름에 따라 설정해놓은 Attribute이 다른데요,

> "  
> position, rotation이나,  
> all로 constraint에 영향을 끼치도록 함.  
> 이 Attribute, Type이 제대로 설정 되지 않으면 position으로 설정해준다  
> "

회전을 잡고싶으면 **rotation**
위치를 잡고싶으면 **position**
둘다 잡고싶으면 **all**로 constraint type을 설정해주시면 되겠습니다.
constraint_name과 type 둘다 Class는 Primitive였죠?

바로 설정 해보도록 하겠습니다

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA009.png" alt="CA009" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA010.png" alt="CA010.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
voronoi로 나눠주고 Assemble하여, Constraint쪽에 primitive로 할당시켜줬습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA011.png" alt="CA011" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="26%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA012.png" alt="CA012.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>

constraint Network도 설정해주고,

Glueconstraint에서 constraint_name으로 할당했던 pig를 설정해주었습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA013.png" alt="CA013" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>
제대로 연결된다면 constraint이 빨간 라인으로 정확히 들어온 것을 확인할 수 있습니다.

### 마무리
Constraint을 잡고 있는다면 이렇게 Sleeping 구조처럼 Impulse가 있는 영역만 컨트롤 할 수 있겠습니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/ConstraintAttribute/CA014.gif" alt="CA014" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>

물론 i@active=0으로 아래에 잡고 작업한 이후에 추가 표현 하였습니다.