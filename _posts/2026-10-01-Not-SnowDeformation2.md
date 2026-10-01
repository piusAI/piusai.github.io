---
layout: post
published: false
title: Snow Deformation ②  - Shader 집중
subtitle: Constraint
date: 2026-09-30 15:50:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
Screen Position, World Position, UV Position에 집중해서 알아보자.

<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/Footprint.png" alt="Footprint" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Footprint</strong> </td>  </tr> </table>

Actor 위치를 Raw하게 그대로 받은 형태가 아니기에
`WorldpostoUV`, 와같은 여러 함수들이 존재하고, material자체에도 수학적인 위치 가 뒤엉켜서 어디지점이 눈을 `WorldPositionOffset.z`로 Vertex를 Displacement시켰는지 정확히 알기 힘들다.

그래서 RenderTarget 다음으로 Material을 집중해서 내부에서 맞물려있는 parameter와, 간단한 Material Parameter Collection를 알아보고, Player 위치를 도대체 어떻게 Grid Mesh의 정확한 위치를 **Mapping**하여 displacement까지 했는지를 로그를 찍어보도록 하겠다.

---
Texture를 실제로 Player가 들어간것처럼 아래와 같은 발자국으로 처리하겠다


## 01 Material

### M_CustomTrail
이 M_CustomTrail은 이렇게 M_CustomTrail의 Parameter와 소통한다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/M_CustomTrail.png"  alt="M_CustomTrail" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_CustomTrail</strong> </td> <td width="33%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/DrawCustomTrace.png" alt="DrawCustomTrace.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>DrawCustomTrace</strong> </td> </tr> </table>


`M_CustomTrail` ∈ `DrawCustomTrace()` ∈ `DrawCurrentAllCustomTrace()`
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MakeCustomTrace.png" alt="MakeCustomTrace" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> CustomTail Structure - RT</strong> </td>  </tr> </table>
`DrawCurrentAllCustomTrace`는 `Trace Struct`의 `Capture RT` 인 `RT_Capture`에다가 Draw함
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_CustomTrailTest.png" alt="M_CustomTrailTest" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> M_CustomTrailTest</strong> </td>  </tr> </table>
예시로 발자국 넣었을때 이런 Mateiral

###  M_Footprint
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_Footprint.png" alt="M_Footprint" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> M_Footprint</strong> </td>  </tr> </table>
M_Footprint는 Contrast주는 Curve Atlas말고는 별거 없음

`M_Footprint` ∈ `DrawFootprint()` ∈ `DrawCurrentAllFootPrint()`
```
//DrawFootprint
M_FootprintMID->SetScalarParameterValue(FName("Depth"), Depth);
```
Footprint는 `DrawCurrentAllFootprint`는 RT_Capture받지 않고 위 M_Footprint Material을 발자국으로 활용
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/SnowFoot.png" alt="SnowFoot" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> SnowFoot</strong> </td>  </tr> </table>
발바닥 자체를 Capture따는거는 발에 완전 맞춘 Texture로 아래처럼 테스트 해봄
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/Footprint_Ex.png" alt="Footprint_Ex" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Footprint_Texture Test</strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/FootprintTest.png" alt="Footprint_Ex" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>FootprintTest Tessellation</strong> </td>  </tr> </table>
이렇게 Vertex 이동이 불안정해진다. 물론 Texture를 조금 더 이쁜걸 써도 되겠지만..
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/SphereFootprint.png" alt="Footprint_Ex" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Footprint Sphere  Tessellation</strong> </td>  </tr> </table>
이런 Sphere가 조금 더 안정적임.  
발자국 밟은 지역에는 안정적으로 하고, displacement 올라와야하는 구간에는 Noise 넣을듯(?) - 확인

###  M_Footprint

## 02 Mapping

### Variable
`ASnowGenerator::` SnowFieldSize, 

### Function
`ASnowGenerator::` **WorldPosToUV**
`ASnowGenerator::` **ComputeScreenPosAndSize**

``

---
실험 구조는 [RenderTarget](https://piusai.github.io/engine/2026/09/30/RenderTarget-Inshader.html)을 간단한 기본 Texture인 Spheremask로 생각하고 Material을 뜯어보겠다


### MID 흐름
M_CustomTrailMID -> 
M_MergeMID -> RT_Snow
M_FootprintMID  ->
M_SnowSideHeap -> RT_SnowHistory



`Accumulation` : `RT_SnowHistory`가 필요하겠다
