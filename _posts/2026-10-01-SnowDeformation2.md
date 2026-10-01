---
layout: post
published: true
title: Snow Deformation ②  - Material에 집중
subtitle: Constraint
date: 2026-10-01 23:50:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
[Snow Deformation 1 - RenderTarget](https://piusai.github.io/engine/2026/09/30/SnowDeformation1.html)에서 이어서, **RenderTarget과 Material이 실제로 어떻게 데이터를 주고받는지**를 정리해보자.

이번에는 Art적인 Texture의 결과보다 다음 흐름에 집중!

> **C++에서 값을 설정한다 → Material이 그 값을 입력으로 받는다 → Material이 GPU에서 계산한다 → 계산 결과를 RenderTarget에 기록한다 → 다시 다른 Material이 RenderTarget을 읽는다**

결국 이 시스템의 핵심은 **Material 자체가 데이터를 저장하는 것이 아니라, RenderTarget이 프레임 사이의 데이터를 전달하는 중간 저장소 역할을 하는 것**이다.

---

Material 테스트를 위해 Texture를 실제로 Player를 동적으로 Material을 확인하기 힘드니,  
아래와 같은 발자국 Texture로 처리하겠다.
<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/Footprint.png" alt="Footprint" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Footprint</strong> </td>  </tr> </table>


### 00. 먼저 전체 Material 한눈에 보기

|     Material     |                주요 입력                |                             하는 일                             |
| :--------------: | :---------------------------------: | :----------------------------------------------------------: |
|  `M_Footprint`   |                 없음                  |                     Character footprint                      |
|    `M_Merge`     | `RT_SnowHistory`, `RT_Intermediate` |       기존 History와 이번 프레임 입력을 **합성**해 `RT_Snow`에 **기록**       |
| `M_SnowSideHeap` |              `RT_Snow`              | `RT_SnowHistory`로 Draw (눈이 눌린 주변으로 쌓이는 Heap 처리 + History 갱신) |
|     `M_Snow`     |          `RT_SnowHistory`           |          최종 Snow Mesh Material, Surface/Defomration          |
|  `M_SnowDebug`   |              `RT_Snow`              |                      (주요치 않은 Material)                       |
| `M_CustomTrail`  |            `RT_Capture`             |                      (주요치 않은 Material)                       |
가장 중요한 두 Material은 다음과 같다
- `M_Merge` : 여러 RT 데이터를 하나의 Snow 상태로 합치는 단계
- `M_Snow` : 그 Snow 상태를 실제 Mesh deformation으로 사용하는 단계

### 01. Material과 C++는 어떻게 연결?
Snow Deformation에서 Material과 C++이 통신하는 방법은 크게 3가지 이다.
#### 01. Material Instance Dynamic -MID
C++에서 특정 MaterialInstance의 parameter를 직접 수정 할 수있다.
``` cpp
M_FootprintMID->SetScalarParameterValue(FName("Depth"), Depth);
```
**흐름** : C++ => MID Parameter 변경 => Material이 Parameter를 사용

#### 02. Draw RenderTarget - RenderTarget
``` cpp
UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_Snow, M_MergeMID);
```
이렇게 `M_Merge`를 `RT_Snow`에 그린다.

#### 03. Material ParameterCollection - MPC
여러 Material에서 공통으로 사용하는 값을 전달한다
``` cpp
UKismetMaterialLibrary::SetVectorParameterValue(
	this, MPC_Snow,
	FName("PivotAndSize"),
	FLinearColor(Pos.X, Pos.Y, SnowFieldSize.X, SnowFieldSize.Y)
);
```
C++ → MID
C++ → MPC
(추후 매핑 때 좀더 자세히 정리)

---
### 02. M_Footprint
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

---
###  03. M_Merge
`M_Merge`는 이름 그대로 **Snow 상태를 합성하는 Material**이다.

`Tick()`에서 `MergeRT()`가 호출된다.

`M_MergeMID`∈ `SetSnowAttenuation()`, `MergeRT()`
``` cpp
// ASnowGenerator::SetSnowAttenuation()
M_MergeMID->SetScalarParameterValue(FName("Attenuation"), Value);

// ASnowGenerator::MergeRT()
FVector2D ratio = DeltaOffset / SnowFieldSize;
M_MergeMID->SetVectorParameterValue(FName("Offset"), FLinearColor(ratio.X, ratioY, 0.f, 1.f));
UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_Snow, M_MergeMID);

```
- 역할 1 : Attenuation으로 점점 사라지게 한다
- 역할 2 : `RT_Snow`자체에다가 `M_MergeMID` material을 뿌림

#### `M_Merge`는 실제로 무엇을 합성하는가?

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_MergeRT.png" alt="M_Merge" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_Merge</strong> </td>  </tr> </table>
`M_Merge`안에서 `RT_Intermediate`, `RT_SnowHistory`를 합성하고있다.

단순히 RT를 복사해서 `RT_Snow`에 그리는 `M_Merge`가 아닌,
`RT_SnowHisotry` + `RT_Intermediate` -> `M_Merge` 내부계산 = `RT_Snow`구조다.

##### MF_OffsetUV
`RT_Snow`의 중점이 이동(offset)되는 이유?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/MF_OffsetUV.png"  alt="MF_OffsetUV" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>MF_OffsetUV</strong> </td> <td width="13%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/MF_OffsetResult.png" alt="MF_OffsetUVResult.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Offset의 결과 - RT_Snow</strong> </td> </tr> </table>
- MF_Offset의 Offset으로 UV의 위치를 맞춰준다.

(추후 Mapping, 위치 맞출떄 좀더 자세히 알아보자)

##### M_Merge Test 
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/M_MergeTest.png"  alt="M_MergeTest" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_MergeTest</strong> </td> <td width="60%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/M_MergeTestResult.png" alt="M_MergeTestResult.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_MergeTestResult</strong> </td> </tr> </table>
이렇게 M_Merge는 RT_Snow에 바로 그리고있기 때문에 일어나는 결과이다.

#### M_Merge 정리

> `M_Merge`는 `RT_SnowHistory`의 **기존 Snow 상태**와 `RT_Intermediate`에 기록된 이번 프레임 입력을 Attenuation / Offset 등의 Parameter와 함께 **합성**하고, 그 결과를 `RT_Snow`에 기록하는 Material.

이 부분이 이 시스템에서 가장 중요한 **"RT → Material → RT"** 구조

---

### 04. M_snowSideHeap
``` cpp
//ASnowGenerator::Tick()
UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_SnowHistory, M_SnowSideHeap);
```
`RT_SnowHistory`에다가 `M_SnowSideHeap`자체를 그림
`M_SnowSideHeap`는 눌린 눈 주변의 Heap 같은 결과를 계산하여 History에 반영하는 단계로 볼 수 있다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_SnowSideHeap.png" alt="M_SnowSideHeap" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_SnowSideHeap</strong> </td>  </tr> </table>
- MaterialParameterCollecion, `MPC_Snow`의 수치를 받아 수정된 Displacement부분
- Material Instance로 만들어서 추가 수정을 위한 Parameter임 (`SetScalarParameterValue` 활용X)
###### M_SnowSideHeap Test
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/M_SnowSideHeapTest.png"  alt="M_SnowSideHeapTest" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_SnowSideHeapTest</strong> </td> <td width="28%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/M_SnowSideHeapResult.png" alt="M_SnowSideHeapResult.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_SnowSideHeapResult</strong> </td> </tr> </table>

이렇게 테스트로 `RT_Snow`만 Emmisive Color에 직접 그리면 Displacement가 `@P.z`로 상승한다.

---

### 05. M_Snow

`M_Snow`는 설산 Terrain/Grid에 할당되어 있는 최종 Snow Material이다.
현재 구조에서는 아래 크게 세 가지 Material 결과를 Blend한다.
`Snow_Ridge_Material`,`Snow_Tracked_Material`, `Original Snow` 
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_SnowTotal.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> M_Snow</strong> </td>  </tr> </table>
위 개별적으로 하나씩 연결해보면

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_SnowSeperation.png" alt="M_SnowSeperation" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Snow_Ridge_Material | Snow_Tracked_Material | Original Snow </strong> </td>  </tr> </table>
각각 연결했을때 이렇게 나타나고 Material을 Blending하였음, Mask에 따라서

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_Snow.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Original Snow</strong> </td>  </tr> </table>
그중 Original Snow에만 집중해보자면,
`RT_SnowHistory`가 들어갔다


#### 05-01 MF_DeformationProcess
이 Material 함수가 `M_Snow`에 핵심 역할임과 동시에 실제 Mesh deformation과 연결되는 핵심 함수다.
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MF_DeformationProcess.png" alt="MF_DeformationProcess" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> MF_DeformationProcess</strong> </td>  </tr> </table>
`MF_DeformationProcess`는 WorldXY좌표를 RT UV로 바꿔서 `RT_SnowHistory`를 샘플링하고, 영역의 가장자리를 1.0으로 Fade 시킨뒤 `MPC` scalar 값을 곱해서 B 채널로 출력하는 함수
(매핑시 좀더 자세히 다루지)


###### 함수 전체 입출력

| 입력                       | 내용 / 타입         |
| ------------------------ | --------------- |
| ``RT_SnowHistory``       | Texture2D       |
| ``OriginalNormal``       | Defualt (0,0,1) |
| `Use NaniteTessellation` | Static Bool     |

| 출력                      | 내용                           | 최종 활용 (`M_Snow`)        |
| ----------------------- | ---------------------------- | ----------------------- |
| `SnowDeformationOffset` | `(0, 0, Z)` 형태의 Z 오프셋 (WPO용) | WPO Slot                |
| `SnowDeformationNormal` | 변형 + 셀프섀도가 반영된 노멀            | Normal Slot             |
| `SnowTrailMask`         | History의 **R**               | -                       |
| `SnowTrackedMask`       | History의 **G**               | `SnowTrackedMask` Blend |
| `SnowRidgeMask`         | History의 **B**               | `SnowRidgeMask` Blend   |
| `NaniteDisplacement`    | Nanite 테셀레이션용 변위             | -                       |
특히 가장 중요한 출력은 `SnowDeformationOffset`이다.
이 값이 결국 `World Position Offset`으로 들어간다.

따라서 실제 Mesh deformation은

`RT_SnowHistory` -> `MF_DeformationProcess` -> `SnowDeformationOffset`
-> `WPO`  ->  Snow Mesh Vertex 이동


---

### 06.M_CustomTrail
이 M_CustomTrail은 이렇게 M_CustomTrail의 Parameter와 소통한다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/M_CustomTrail.png"  alt="M_CustomTrail" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_CustomTrail</strong> </td> <td width="33%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/DrawCustomTrace.png" alt="DrawCustomTrace.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>DrawCustomTrace</strong> </td> </tr> </table>


`M_CustomTrail` ∈ `DrawCustomTrace()` ∈ `DrawCurrentAllCustomTrace()`
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MakeCustomTrace.png" alt="MakeCustomTrace" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> CustomTail Structure - RT</strong> </td>  </tr> </table>
`DrawCurrentAllCustomTrace`는 `Trace Struct`의 `Capture RT` 인 `RT_Capture`에다가 Draw함
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_CustomTrailTest.png" alt="M_CustomTrailTest" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> M_CustomTrailTest</strong> </td>  </tr> </table>
예시로 발자국 넣었을때 이런 Mateiral로 나오는데
->Material 끊었는데도 크게 중요하게 다뤄지지는 않음


### 07. 전체 데이터 큰흐름
```
                 ┌──────────────────┐
                 │   Footprint Data │
                 └────────┬─────────┘
                          ↓
                    M_Footprint
                          │
                          ↓
                   RT_Intermediate
                          ↑
                          │
                    M_CustomTrail
                          ↑
                          │
                      CaptureRT
                          ↑
                          │
                    SceneCapture
                          │
                          │
              ┌───────────┴───────────┐
              │                       │
              │     C++ SnowGenerator │
              │                       │
              └───────────┬───────────┘
                          │
                          ↓
                    M_Merge
                          ↑
              ┌───────────┴───────────┐
              │                       │
        RT_SnowHistory         RT_Intermediate
              │
              ↓
           합성 / 감쇠
              │
              ↓
           RT_Snow
              │
              ↓
        CopyToHistory()
              │
              ↓
      RT_SnowHistory
              │
          ┌───┴───────────────┐
          │                   │
          ↓                   ↓
  M_SnowSideHeap            M_Snow
          │                   │
          │                   ↓
          │          MF_DeformationProcess
          │                   │
          │                   ↓
          │         SnowDeformationOffset
          │                   │
          │                   ↓
          │                  WPO
          │                   │
          └───────────────────┴──→ Snow Mesh

```

입력 → RT에 기록 → Material 합성 →RT에 다시 기록 → 다음 Material RT를 읽음  
→ `WPO`계산 → Mesh Deformation


결국 Material은 크게 네 단계로 이루어져있다.

1. 데이터를 읽는다
	`RT_SnowHistory`, `RT_Intermediate`, `CaptureRT`(ParameterTexture) ,`MPC`, `MID Parameter`
2. GPU에서 데이터를 계산한다
	 Material Node : Multiply, Max, Offset
	 Parameter Value : Attenuation, Depth
3. 결과를 다시 RT에 기록한다
	`DrawMaterialToRenderTarget()`
4. 최종 Material에서는 그 결과를 Mesh에 적용한다
	`RT_SnowHistory` -> `MF_DeformationProcess`->`WPO` ->`Mesh`

---
이전의 RenderTarget 관련한 내용은 [Snow Deformation 1- RenderTarget](https://piusai.github.io/engine/2026/09/30/SnowDeformation1.html)에서 자세히 볼 수있다.

이제 Screen / UV Mapping으로, Character Offset만 정확하게 데이터 흐름만 이해하면 전반적인 SnowDeformation을 이해 완료할 수있겠다.