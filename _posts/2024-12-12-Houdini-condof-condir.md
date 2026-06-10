---
layout: post
published: true
title: Houdini condof condir
subtitle: Constraint
date: 2024-12-12 15:45:00 +0900
description: Houdini condof condir이란
categories:
  - Art
tags:
  - Houdini
  - Art
---
Constraint적용시 방향과 위치를 설정하는 condof, condir에 대해 알아봅시다.

Constraint을 적용하려 할 때, 방향과 위치를 정해보고 싶다고요~?
그 attribute은 바로  
**condir와 condof입니다.**

지체하지 말고 바로 들어 갈 까 요 !

---
## Constraint Attribute, condir & condof
constraint는 object 간의 물리적 관계 설정하고, 어떻게 상호작용할지 결정하는 큰 역할을 합니다.

position과, rotation으로 설정한다면, 그에 맞춘 특정 각도와, 방향을 컨트롤 할 수 있겠죠?  

이를 컨트롤 가능토록 하는 attribute는
condof와 condir인데요,
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC001.png" alt="CC001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[condir, condof의 예시 사용]</strong> </td>  </tr> </table>

이를 이해한다면 하나의 축을 기준으로 Rotate나, transform 돌리거나 이동할 수 있겠습니다.

### condir와 condof의 문법
#### v@condir
**Con**straint **Dir**ection의 약자입니다.
Class는 **Primitive/point**, Data type은 **vector**
=>Constraint가 object에 대해 주는 rotation방향입니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC002.png" alt="CC002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[condir, condof의 예시 사용]</strong> </td>  </tr> </table>

z축으로 설정함.

{1,0,0} : constraint이 **x**축을 따라 적용
{0,1,0} : constraint이 **y**축을 따라 적용.
{0,0,1} : constraint이 **z 축을** 따라 적용.

``` hlsl
v@condir={0,0,1};
```

condir를 통해서 Constraint의 힘이나 움직임이 어느 방향으로 제한할지를 결정하는 것이기 때문에,
sim에서 object의 작용을 더욱 컨트롤할 수 있겠죠~?

우리가 원하는 특정방향이나, constraint이 작동하게 하거나 가능하겠습니다!

#### i@condof

**Con**straint **D**egrees **O**f **F**reedom의 약자입니다.

Class는 **Primitive/point**, Data type은 **integer**
Constraint이 object에 대해 어느 정도의 자유도를 허용할지 control 할 수 있는데요,

  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC003.png" alt="CC003" style="width: 100%; max-width: 100%; height: auto;"> <br> </td>  </tr> </table>
아래 gif는 모두다

``` hlsl
i@condif={0,0,1};
```

로 **Z**축으로 설정하였습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC004.gif" alt="CC004" style="width: 100%; max-width: 100%; height: auto;"> <br> </td>  </tr> </table>
**0 : 복원력 없음**

``` hlsl
i@condof=0;
```
복원력 없이 어디로든 움직일 수 있는 상태.

=>모든 방향으로 회전되는것을 알 수 있습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC005.gif" alt="CC005" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>
**1 : condir 방향만을 제외한 모든 방향으로 움직임**

``` hlsl
i@condof=1;
```
condir로 설정한, 해당 축 외의 축에 따라 완전한 자유가 이루어 짐.
=>xy방향으로 회전되는것을 알 수 있습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC006.gif" alt="CC006" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>
**2 : Condir방향으로만 움직임**

``` hlsl
i@condof=2;
```
**condir**로 설정한 해당 축만을 따라 **Rotation**만 **허용.**
=> Z축 방향으로만 회전되는것을 알 수 있습니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC007.gif" alt="CC007" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>
**3 : 움직임이 고정**

``` hlsl
i@condof=3;
```
Pin을 찍은듯이 그 자리에 고정

#### Runover Point vs primitive?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC008.png" alt="CC008" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>
눈치 채신 분이 있을지 모르겠는데, condof와 condir의 Class에서는

constraint은 기본적으로 **primitive**가 맞지만,  
**point**로 class를 넣었을때가 primitive로 넣었을때보다 더 정밀하게 컨트롤 할 수 있어서

v@condir, i@condof는 point를 활용 하는것이 맞습니다.

### 마무리

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/CondofCondir/CC009.gif" alt="CC009" style="width: 100%; max-width: 100%; height: auto;"> <br></td>  </tr> </table>

Primitive로 설정해서  **condir, condof를** 설정한다면 이런 작업 가능하겠죠?

**Hint!**
: wrangle 

``` hlsl
s@constraint_name='rot';
s@constraint_type='rotation';

v@condir={0,0,1};
i@condof=2;
```
 위는 condir방향인, z축만을 허용한 fig 쪽의 Constraint이고
 
``` hlsl
s@constraint_name='pos';
s@constraint_type='position';

v@condir={0,0,1};
i@condof=3;
```
=>Pin을 찍은 듯이, 움직임이 고정 될 수 있도록 condof를 3으로 설정하였습니다.