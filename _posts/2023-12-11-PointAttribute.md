---
layout: post
published: true
title: "다른 Node에서의 정보를 가져오는, Point(Feat.Nearpoint) | 후디니 Wrangle, Vex"
subtitle: "What is Point?"
date: 2023-12-11 15:45:00 +0900
description: "Houdini Point"
categories: [Art]
tags: [Houdini, Art]
---
'Vex '  
다른 Input Node에서의 정보를 가져오고 싶다!  
**Point, Nearpoint**를 알아봅시다.

## Vex, Point

다른 노드에서의 Attribute를 가져오기 위해서 Wrangle을 활용하여 가져와볼까요?

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP001.png" alt="VP001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Attribute Transfer]</strong> </td>  </tr> </table>

Grid는 검정색으로, Sphere에서는 Green으로 색상을 설정하였습니다.

### Point Wrangle

먼저 확인해보아야 할 것은, Black Node(Color Node)의 Geometry Spread Sheet입니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP002.png" alt="VP002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[RMB - Spreadsheet]</strong> </td>  </tr> </table>



<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP003.png" alt="VP003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Black Node]</strong> </td> <td width="60%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP004.png" alt="VA004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Green Node]</strong> </td> </tr> </table>

: SpreadSheet 비교
  
이렇듯 Black Node와 Green Node에서의 Attribute(정보)는 Point Level에, @P와 @Cd가 들어가 있습니다.  
Green Node는 Sphere가 Primitives로써, Point 하나로 존재하는군요.  
Grid의 Point의 Point Num은 (50X50)으로 2500개 있을 것 입니다.  
  
먼저, 가장 기본적으로 PointWrangle만을 활용하여 Green의 Color attribute, @Cd를 가져오겠습니다.  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP005.png" alt="VP003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Scene 배치]</strong> </td>  </tr> </table>


Attribute Wrangle을 Input1에 Grid를 연결, Input2에 Sphere를 연결합니다.  
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP006.png" alt="VP006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Point Vex]</strong> </td>  </tr> </table>


```
@Cd= point(1,"Cd",0);
```

이런 형식으로 작성해야하는 이유는 아래에 있겠죠!

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP007.png" alt="VP007" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[SideFx Expression docs]</strong> </td>  </tr> </table>

  
위에서 나오듯, 하나씩 expression을 적어주면 됩니다.  
  
Geometry : Input 순서( Input1 : 0, Input2: 1, Input3: 2 ...)   
Attribute_name : Attribute 정보 (Ex, "P", "Cd", "N" ...)  
Point number : Point Number( N번째 Point)

``` hlsl
@Cd= point(1,"Cd",0);
//두번째 인풋에서의 0번째 포인트, Cd attribute를 가져와줘.
//그리고 Input1의 @Cd정보에 저장해줘
```

  
그렇다면 아래의 Point는 왜 Function의 뼈대가 다를까요?  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP008.png" alt="VP008" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[SideFx VEX Docs]</strong> </td>  </tr> </table>


위에 나오는 Expression은 Vex와 활용도가 다릅니다.  
쉽게 이야기한다면, Node위에서 발생되는 것들은 Expression으로 활용되고,  
Wrangle 위에서 실행하는 Function들은 Vex로 활용됩니다.  
  
그렇다면  주변의 Point만을 인식시켜 Grid를 수정해볼까요?  
Point만으로는 활용할수는 없을 것 입니다. 

### NearPoint Wrangle

위에서 말했듯, 주변의 Point만을 인식시킨다는 요구사항때문에, nearpoint라는 wrangle을 활용해야 할 것입니다.  
  

#### 첫번째, nearpoint(Geometry, pt)

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP009.png" alt="VP009" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[SideFx VEX Docs]</strong> </td>  </tr> </table>

 Output은, int로 몇번째의 point인지 확인 가능한 Vex입니다.   
이제, Nearpoint의 뼈대, Geometry, pt를 넣어 볼까요?

``` hlsl
int np =nearpoint(1,@P);
@Cd= point(1,"Cd",np);
```

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP010.png" alt="VP010" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[NearPoint]</strong> </td>  </tr> </table>

앗 그런데 왜 전체에다가 다 불러와졌을까요?

``` hlsl
int np =nearpoint(1,@P);
// 현재 Grid의 ptnum와 Input2에서의 Position vector(@P)랑 가까운 Point number를 np, in variable에 넣어줘
@Cd= point(1,"Cd",np);
// 현재 Grid의 Color attribute에 np번째(0번째) Color 정보를 넣어줘,
```

Nearpoint함수에서, 아직 거리계산없이, Point번호 0번째만을 인식시켜 Grid 전체에다가 Color attribute가 들어갔습니다.그렇기 때문에 NeaPoint Vex function의 두번째 활용도 써볼게요.  
  

#### 두번째, nearpoint(Geometry, pt, maxdist)

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP011.png" alt="VP011" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[NearPoint]</strong> </td>  </tr> </table>

  
maxdist까지 넣어 최대 거리도 설정해주죠.  
직접 Float 거리를 입력해도 되겠지만, chf('dist')를 활용하여 인터페이스 조작 쉽게 하겠습니다.

``` hlsl
int np =nearpoint(1,@P,chf("dist"));
@Cd= point(1,"Cd",np);
```

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP012.png" alt="VP012" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Distance 조정된 모습]</strong> </td>  </tr> </table>


``` hlsl
int np =nearpoint(1,@P,chf("dist"));
// nearpoint 찾는 함수의 최대 거리를 dist만큼 설정해줘
@Cd= point(1,"Cd",np);
```

Sphere의 Center X와 Center Z에 Sin Cos만 넣어준다면 이동도 가능하겠죠?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP013.png" alt="VP013" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[translate sin/cos]</strong> </td>  </tr> </table>



### 마무리

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VEXPoint/VP014.gif" alt="VP014" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Transfer Attribute Done]</strong> </td>  </tr> </table>

