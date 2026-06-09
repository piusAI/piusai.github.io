---
layout: post
published: true
title: "Houdini에서의 Level!"
subtitle: "What is Level in houdini?"
date: 2023-12-13 15:45:00 +0900
description: "Houdini Level"
categories: [Art]
tags: [Houdini, Art]
---

Maya, C4d, Blender, 3dMaxs와 같은   
다른 3D DCC tool과는 다르게, Houdini는 Context구조로 되어있는 것이 또 특징입니다.  
**Level, Context**를 알아봅시다.

## Context

Houdini 처음 시작할때, 가장 다른 것중 하나일 것입니다. Maya에서는 어떤 Object, Edge 선택상태에 따라서 Tool이 이루어 지지만, Houdini에서는 조금 다릅니다. 물론 그러한 선택 상태를 활용하여, 툴이 Assign 되는 경우가 있지만, 어떤 Context에 있느냐에 따라서 활용되지 못하는 경우가 허다합니다.  
  
그렇다면 어떤 Level에, 어떤 Context에 있는지 확인 가능해야겠습니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L001.png" alt="L001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Network View 열기]</strong> </td>  </tr> </table>



위와같이 Network View를 열어주세요.  
  
크게 Scene 전체를 컨트롤할 수있는, Object Level, 조금 더 세부적인 Context, Sop Dop Rop Context등으로 나뉘어지는 보다 상세한 작업이 가능한 Context를 알아봅시다.

### Scene Structure Level | 씬 구성하는 광범위한 Level

후디니에 딱 들어오면 가장 상위의 Context로 Object Level이 펼쳐집니다.  
Houdini는 Scene Structure Level이 가장 상위에 있어, 전체 Scene에 대한 제어를 제공합니다. 이 Level에서는 Controll 역할을 하여 이용자가 3D Scene의 중요한 Structure, Hierachy(계층) 및 Transformation(변형)을 형성할 수 있도록 해줍니다.  

#### Object Level

OBJ Level에서는 장면 구성, 계층 및 변환에 관한 것입니다.  
이는 장면의 Root 수준 역할을 하며 Geometry 노드, Null 노드, Camera 노드 및 Light 노드가 포함됩니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L001.png" alt="L001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Network View 열기]</strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L002.png" alt="L002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="400%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L003.png" alt="L003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Object Level]</strong> </td> </tr> </table>
OBJ는 다양한 요소, 변환 및 계층 간의 관계를 포함하여 장면의 전체 구조를 관리합니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L004.png" alt="L004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Object Level Node 열기]</strong> </td>  </tr> </table>

Object Level에서는 이렇게 장면 요소를 정의하는 노드를 활용하여 Scene Structure 작업을 합니다.

자세한 Geometry 조작보다는 **Scene의 광범위한 구성에 중점**을 둡니다.
Node ex) Cam, Geomtry, Null, Light 작업등의 Object Level Node

### Context | 좀더 세부적인 Level

위의 Object Level에서의 광범위한 작업과는 대조되는 SOP, DOP 및 ROP Context는 이용자가 3D Dcc의 복잡성에 대해 자세히 알수 있도록 제공 합니다.  
각각 표면 작업(SOP), 역학(DOP) 및 렌더링(ROP)을 전문으로 하는 이러한 Context는 보다 상세하고 집중된 ToolKit을 제공합니다. 


#### Sop(Surface OPeration) Level | Detail Level
SOP는 형상의 조작 및 모델링, Attribute Set을 다룹니다.


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L005.png" alt="L005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Sop</strong> </td>  </tr> </table>
  
주로 3D OBJ의 Surface, 표면을 생성하고 수정하는 데 사용됩니다.  
  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L006.png" alt="L006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Dop simulation Sop작업</strong> </td>  </tr> </table>

Dop simulation Sop작업

SOP는 3D 형상의 모델링, 텍스처링, 변형 및 변환과 같은 다양한 작업을 수행할 수 있습니다. 


DCC tool에서 활용해왔던, 아래와 같은 Maya나 등등 3d software에서 활용 해왔던 단계는 이 단계일 것입니다.  


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L007.png" alt="L007" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Maya Scene View</strong> </td>  </tr> </table>

Node ex) Box, Sphere, PolyExtrude, Transform 및 Boolean 작업등의 Sop Node  

#### Dop(Dynamic OPeration) Level | Simulation Level

DOP는 시뮬레이션과 역학에 중점을 둡니다. 물리 시뮬레이션의 초기 설정, 시뮬레이션 동작 및 결과를 다룹니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L008.png" alt="L008" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>DOP</strong> </td>  </tr> </table>


이 Context는 Smoke, Fire, Destruction, 천 시뮬레이션 및 유체 시뮬레이션과 같은 효과를 만드는 데 사용됩니다.  
FX Simulation은 이 Level에서 활용 되기도 합니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L009.png" alt="L009" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>DOP Fracture Simulation</strong> </td>  </tr> </table>

그렇지만 모두다 Simulation이 여기서 일어 나는 것은 아닙니다.  
Sop에서도 Animation 물론 가능하고, 또한 Simulation 가능한 Sop Solver까지 있습니다.

<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L010.png" alt="L010" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Sop solver</strong> </td>  </tr> </table>

Node ex)  Smoke Solver, RBD Buillet Solver, 유체 시뮬레이션용 Flip Solver 및 POP(Particle Simulation) 노드.

#### Rop(Render OPeration) Level | Output Level

ROP는 최종 이미지 또는 애니메이션의 렌더링 및 출력을 처리합니다. 여기서는 렌더링 매개변수를 설정하고 출력 형식을 지정합니다.


<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L011.png" alt="L011" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Output Level</strong> </td>  </tr> </table>

ROP 노드에는 렌더링 엔진, 이미지 형식 및 프로젝트의 최종 출력과 관련된 기타 옵션에 대한 설정이 포함됩니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Level/L012.png" alt="L012" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> ROP Cam</td>  </tr> </table>

Ex): Mantra 렌더러용 Mantra ROP, Redshift 렌더러용 Redshift ROP, Alembic 파일 내보내기용 Alembic ROP. 

#### SHop(Shading Operation) Level

Shader 연산자를 사용하여 Material 및 Shading을 설정하는 레벨입니다.  
이 레벨에서는 물체의 외관을 정의하고 빛과 재질 속성을 설정하여 렌더링 결과를 제어할 수 있습니다.

#### COP (Composit OPeration)

Composit 연산자를 사용하여 이미지를 합성하고 편집하는 레벨입니다. 이 레벨에서는 이미지를 필터링, 합성, 색상 보정 등 다양한 작업을 수행할 수 있습니다. 

#### CHop(Channel Operation) Level | Channel Level

주로 애니메이션, 오디오와 같이 시간에 따라 변하는 데이터를 나타내는 Channel 조작을 다룹니다.  
모션, 오디오 및 기타 시간 기반 데이터를 생성하고 편집하는 데 사용됩니다.

