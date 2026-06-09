---
layout: post
published: true
title: "Houdini Component?"
subtitle: "What is Component in houdini"
date: 2023-12-18 15:45:00 +0900
description: "Houdini Level"
categories: [Art]
tags: [Houdini, Art]
---
Houdini Component에 대해서 알아봅시다.

Attribute Level에서 활용되는  
**Point, Vertex, Primitives**가 무엇일까요?   
결론부터 말씀드리자면 Component로 각 구성 요소입니다.

---

## Point, Vertex, Primitives, Detail
Maya에서는 Vertex만을 활용했는데, Point와 Vertex를 나누어서 Houdini에서는 활용해서 처음에 이해가 되지 않았습니다.  

### Component에 따른 Attribute Level

Side fx에서 제공하는 Docs에서 Attribute Level을 한장의 이미지로 설명할 수 있습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC001.jpeg" alt="HC001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Houdini Docs</strong> </td>  </tr> </table>
그렇다면 하나씩 뜯어 볼까요?

#### Primivites
Primitives 레벨의 Attribute는 다수의 Vtx를 포함한 Object를 다룹니다.  
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC002.png" alt="HC002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Runover : Primitives</strong> </td>  </tr> </table>
  
``` hlsl
if(@P.y>0){
s@name='U_sogi';
}
else{
s@name='D_sogi';
}

//만약 P.y가 0보다 크다면 U_sogi라는 name Data를 string으로 Primitive Attribute에 넣어주고
아니라면, D_sogi라는 name Data를 string으로 Primitive Attribute에 넣어줘
```

  
이를 이용하여 Object 전체에 일괄적으로 Attribute를 부여하거나 다양한 Group을 조작할 수 있습니다.  
Geometry의 Component중 하나의 면이 하나의 Primitive 수로 다루어 집니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC003.png" alt="HC003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Primitive Attribute</strong> </td>  </tr> </table>
위와 같이 U_sogi와 D_sogi로 name attribute가 Primitives에 들어갔습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC004.png" alt="HC004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Pack Done</strong> </td>  </tr> </table>

Assemble Sop을 활용하여 Packed Primitives를 만든다면  
이름이 같은 U_sogi는 U_sogi로, D_sogi는 D_sogi로 Packed 됩니다.  
(Create Name Attribute는 새롭게 이름을 지정하니 기존에 있던 이름으로 만들때는, 체크해제합니다.)

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC005.png" alt="HC005" style="width: 100%; max-width: 100%; height: auto;"> </td> </tr> </table>
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC006.png" alt="HC006" style="width: 100%; max-width: 100%; height: auto;"> </td> </tr> </table>
Assemble Sop 특성상 Point 두개로 Packed 되었습니다.

  

#### Point
포인트 레벨의 Attribute는 각각의 점에 대한 정보를 담고 있습니다.  
항상 가지고있는 Attribute라면, Point 정보는 항상 가지어 Geometry SpreadSheet에서 표현해줍니다.  
이를 통해 개별적인 Point에 대한 조작과 변형을 자유롭게 할 수 있습니다.

``` hlsl
  vector Pos = Point(1,"P",0);
// Input2에서의 0번째 Point, Point정보 "P"를 가져와줘
```

여타 vex들을 가지고 놀때, Attribute 정보를 가지고 작업시, Point에서 정보를 활용 많이 하는 경우가 있습니다.

#### Detail

디테일 레벨의 Attribute는 Object 전체에 대한 특성을 담고 있습니다.  
전체 Object의 Attribute가 같은경우 하나의 Detail Attribute로 심을 수있습니다.


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC007.png" alt="HC007" style="width: 100%; max-width: 100%; height: auto;"> </td> </tr> </table>
Box를 측정한 area는 정육각형이기 때문에 같은 area 데이터를 갖고있습니다.  
그렇기 때문에 Attribute Promote를 활용하여 Primitives에있던 Attribute에서 Detail Attribute로 승격하여

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC008.png" alt="HC008" style="width: 100%; max-width: 100%; height: auto;"> </td> </tr> </table>
하나의 Data로 활용할 수 있겠습니다.

이를 활용하여 전체적인 설정이나 효과를 적용할 수 있습니다.
``` hlsl
vector Col = detail(1, "Cd");
// Input2에서의 Detail 정보 "Cd"를 가져와줘
```
추가적으로 아래같이도 활용 할 수있겠습니다.


#### Edge

Edge도 활용이 없지는 않습니다.  
Edge 라인을 갖고 Extrude등을 할때, Group등을 활용하기 때문에 필요시 Attribute나 Group을 잡아 작업 합니다.

  
Extrude를 한번 해볼까요?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC009.png" alt="HC009" style="width: 100%; max-width: 100%; height: auto;"> </td> </tr> </table>
윗 그룹 Face를 선택하여 Group을 잡고, Seam Group을 활성화하기위하여 체크합니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC010.png" alt="HC010" style="width: 100%; max-width: 100%; height: auto;"> </td> </tr> </table>
Display Goup, Attribute List 활성화

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC011.png" alt="HC011" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Edge Select</strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Component/HC012.png" alt="HC012" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Edge Group</strong> </td> </tr> </table>
이렇게 Edge들을 활성화하여 이 Edge Group들을 활성화하여 작업 할 수있습니다.


#### Vertex
버텍스 레벨의 Attribute는 인접한 포인트 간의 관계를 다룹니다.
UV와 같은 정보도 Vertex정보로 작업 가능하겠죠?
이를 활용하여 Geometry 형태와 표면 특성을 부여할 수 있습니다.  

