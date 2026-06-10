---
layout: post
published: true
title: Houdini VertexPoint
subtitle: PyroSimulation
date: 2024-10-12 15:45:00 +0900
description: Houdini VertexPoint
categories:
  - Art
tags:
  - Houdini
  - Art
---
Vex   
primitive에서 N번째 Point  number를 찾아보는 방법은 어떤것이 있을까요?  
**vertexindex, vertexpoint**를 알아봅시다.

---
## Vex, VertexIndex Vertexpoint
Vertex의 특성을 활용해서 Point number를 찾을 수 있는 vex 바로 알아보입시다.  

### 예제 Settings
#### ConvertLine을 활용한 Polygon =>Line Primitive
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP001.png" alt="VP001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
먼저 Cube, box를 Line, primitive 형식으로 바꾸겠습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP002.png" alt="VP002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
**Red**:  Point number(@ptnum)
**Green** : Primitive number (@primnum)
이렇게 Point number와 primitive number를 확인하면서 들어가보겠습니다.

### Vex | vertexPoint, vertexIndex
#### VertexPoint
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP003.png" alt="VP003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

먼저 Vertex Point는 Linear vertex로 부터 **Point number**를 반환시켜준다고 합니다.
찾은 **Point number**를 활용해서 Attribute를 할당 시킬 수도 있겠죠?

***Geometry*** : 0 
Geometry는 0으로써 그대로 받아온 input1을 활용할 것이고,

***LinearVertex*** : ?
Linear Vertex를 찾기 위한 vex를 하나 더활용해야하겠군요!
#### Vertexindex
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP004.png" alt="VP004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
Linear Vertex를 찾는 vertexindex입니다.
Primitive/vtx를 linear vertex로 반환 시켜 준다고 하는데요!

***Geometry*** : 0 자기자신, input1에서 찾겠습니다.
***Primnum*** : @primnum, 각각의 primitive number를 활용합니다.
***Vertex*** : 0 각각의 primitive에서의 N(0)번째의 vertex number
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP005.png" alt="VP005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
vertex Linear를 **vertex index**에서 찾은 이후,
**vertexpoint**를 활용해서 찾은 **vertexindex**로 point number를 찾았습니다.

### 마무리
결론으로 가봅시다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP006.png" alt="PS006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="60%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP007.png" alt="PS004.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
0, 1, 2번째의 Primitive에서의 0번째 vertex를 0으로 찾았습니다.

주의해야 할 점은 공유하고 있는 point number가 아니라, **0번째의 Vertex**를 찾는 것 이었죠?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP008.png" alt="PS008" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="60%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VertexPoint/VP009.png" alt="PS0049png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
그렇기 때문에 3,4번째 Pimitive의 0번째 point number는 1로 잘 반환 되었고,
1번 Pimitive의 0번째 point number는 0으로 할당 되겠습니다.