---
layout: post
published: true
title: Snow Deformation ③ - Mapping 집중
subtitle: Constraint
date: 2026-10-02 21:51:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
[Snow Deformation 1 - RenderTarget](https://piusai.github.io/engine/2026/09/30/SnowDeformation1.html)과 [SnowDeformation 2- Material]([Snow Deformation ② - Material 집중 | PIUS](https://piusai.github.io/engine/2026/10/01/SnowDeformation2.html))에 이어서
Screen Position, World Position, UV Position에 집중해서 알아보자.

눈 RT는 플레이어를 따라 움직이는 **Scrolling RT**이다.  
다시말해 RT에 어떤식으로 **Draw material**하는지도 중요하지만, 결국 Actor의 `FootprintInfo`, `CustomTrace`의 정보를 다시 Mapping해야하는 지점에 봉착하게 된다.

<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MF_OffsetResult.png" alt="MF_OffsetResult" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RT_Snow</strong> </td>  </tr> </table>

Actor 위치를 Raw하게 RT에 그대로 받은 그리는 형태가 아니기에 동적으로 관리하기 용이하다. 반면 `WorldpostoUV` 와같은 여러 함수들이 존재하고, `Material`자체에도 수학적인 위치 계산이 뒤엉켜서 어느지점에서 눈(Snow)을 `WorldPositionOffset.z`로 `Vertex Displacement`시켰는지 정확히 알기 힘들다.

Mapping은 아래 크게 세덩어리로 나누어 본다.
1. Variable에서의 Mapping : MPC, Material Parameter, Struct
2. C++ Function의 Mapping
3. Material에서의 Mapping

간단한 Material Parameter Collection를 알아보고, Player 위치를 도당체 어떻게 Grid Mesh의 정확한 위치를 **Mapping**하여 Displacement까지 했는지를 로그를 찍어보도록 하겠다.

(완전히 독립적이지는 못하여서 항목이 겹쳐있을 수있음)

---
### 01 MPC, Material Parameter : Variable Mapping
Material ParameterCollection, Material ParameterMapping
가장 쉽게 이해 할 수있는 Variable에서의 Mapping을 알아보입시다~

#### 01-01 Variable

`ASnowGenerator` : `(FVector2D)SnowFieldSize`, `(FVector2D)RTResolution`

**SnowFieldSize**:  
Snow FIeld의 크기이다. Default값으로는 (3000x3000)으로 설정했다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/SnowFieldSize.jpg" alt="SnowFieldSize" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SnowFieldSize : 1200x1200</strong> </td>  </tr> </table>
위와 같이 1200x1200으로 SnowField를 설정하면 짤린다.  
`PixelAlign()` (02-03 참고)에서 맞춰줘야한다.

**RT_Resolution** :
`RT_Snow`, `RT_SnowHistory`, `RT_Intermediate` - 2k(2048) 고정!

#### 01-02 MPC : Material Parameter Collection

<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MPC.png" alt="MPC" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>MPC</strong> </td>  </tr> </table>

`Material Parameter Collection`이라 하더라도, 구조체와 별반 다르지 않다.
``` cpp
//MPC_Snow
float SnowHeight = 0.0f;
Fvector4 PivotAndSize = Fvector4(860.f, 1035.f, 1000.f, 1000f);
// PivotAndSize : RG = Pivot(XY), BA = Size(XY)
```
이 값들의 흐름은 아래와 같다


>    ↓  Write ↓                                            ↓   Read  ↓
`cpp_SnowGenerator` -> **`MPC_Snow`** -> `M_Snow`, `M_CustomTrail`, `M_SnowSideHeap`

`ASnow_Generator`에서 Set/Write해주고,  `M_Snow`, `M_CustomTrail`, `M_SnowSideHeap`에서 Get(Read)한다.

**SnowHeight**:  
정적으로 BP에서 Initialize 값으로 설정할 수있다.
``` cpp
SetScalarParameterValue(this, MPC_Snow, FName("SnowHeight"), SnowHeight);
```

**PivotAndSize**:  
RT가 Scrolling되며 픽셀 정렬을 해야한다 했었다.  
`PixelAlign`(02-03 참고)을 꼭 해줘야 RT 중심이 항상 **Texel**경계 위에서만 이동한다.
```cpp
// ASnowGenerator::UpdatePos()
const FVector TrackingLocation = IsValid(Pawn) ? Pawn->GetActorLocation() : FVector(-777.f,-777.f, -777.f );
Pos = PixelAlign(TrackingLocation);
UKismetMaterialLibrary::SetVectorParameterValue(this, MPC_Snow, FName("PivotAndSize"), FLinearColor(Pos.X, Pos.Y, SnowFieldSize.X, SnowFieldSize.Y));
```
- **pivot(RG)** : `Pos.X`, `PosY`로 Pixel 정렬된 값들로 중앙 위치를 잡는다.
- **Size(BA)** :  `SnowFieldSize`가 그대로 들어감.


---

### 02 C++ Function
#### 02-01 `ASnowGenerator::` **WorldPosToUV**
: WorldPos vector정보를 UV에 맞춰주는 Function
``` cpp
FVector2D ASnowGenerator::WorldPosToUV(const FVector& WorldPos)
{
	return ( FVector2D(WorldPos.X, WorldPos.Y) -Pos ) / SnowFieldSize + FVector2D(0.5, 0.5);
}
```
**코드 분석** :  
> `PixelAligned`된 Pos를 SnowFieldSize 비율로 나누고 원점을 0.5, 0.5 offset해줌

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/WorldPosToUV.png" alt="WorldPosToUV" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>WorldPosToUV - without (Pos기준 뺄셈)</strong> </td>  </tr> </table>
Pos기준 뺄셈을 하지 않으면 위와 같이 위치 이격이 일어남.  
Pos를 기준으로 꼭 빼줘야함!

- `WorldPos - Pos` : Pos를 원점으로 한 `WorldPos` 좌표 : **RT 중심 기준 상대 좌표** 변환
-  `/SnowFieldSize` snow 필드 크기로 나누어 (-0.5~0.5)범위로 **Normalize**
-  `+0.5` 원점 중심으로 (0~1) 범위로 만든다.

##### 활용

```cpp
WorldPosToUV(Trace.Location); //FCustomTrace Struct 내부
WorldPosToUV(footprint.Location); //FFootprint Struct 내부
```

#### 02-02 `ASnowGenerator::` **ComputeScreenPosAndSize**
: Pos와 Size를 RT의 크기에 맞게 해주는 함수.

``` cpp
void ComputeScreenPosAndSize(FVector2D UV, float TexSizeScale, float ContactRadius, FVector2D& inPos, FVector2D& inSize)
{
	float Temp_Size = TexSizeScale * ContactRadius ; //World기준 footprint
	inSize = ComputeSizeForRT(Temp_Size, Temp_Size); // -> RT 픽셀 크기
	inPos = UV * Resolution - (inSize / 2.0f);      // ->RT 픽셀좌표의 좌상단
}
```
- `inSize` : World 크기(Temp_SIze)를 `ComputeSizeForRT()` RT 픽셀크기(Pixel수)로 환산
- `inPos`: RT Resolution 로 픽셀 좌표 중심을 구한뒤
- `- (inSize / 2.f)` : 중심 기준 위치를 **좌상단 기준**으로 이동 
  -> insize(RT 픽셀 크기)의 절반만큼 다시 Pos를 이동시킨다
	`DrawMaterial`의 `ScreenPosition`이 좌상단이 기준!
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/ComputeScreenPosAndSize.png" alt="ComputeScreenPosAndSize" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ComputeScreenPosAndSize</strong> </td>  </tr> </table>
 `inPos = UV * Resolution`이렇게, 중심 기준 **좌상단**으로 이동하지 않으면 발이 조금 밀린다
 
##### ::ComputeSizeForRT

``` cpp
FVector2D ComputeSizeForRT(float x, float y)
{
	return FVector2D(x, y) / SnowFieldSize * RTResolution;
}
```

- ` / SnowFieldSize` : 월드 크기를 Field 대비 비율(0~1)로 만들고
- ` * RTResolution` : 그 비율을 Pixel수로 환산! 

##### 활용
``` cpp
const FVector2D UV = WorldPosToUV(footprint.Location);
ComputeScreenPosAndSize(UV, 2.0f, footprint.ContactRadius, InPos, InSize);
//Inpos, InSize -> DrawMaterial(Canvas)에 넘길 RT 픽셀 좌표계 값
```
구조체 내의 정보를 가져와서 `ScreenPosAndSize`맞춤

> `WorldPosToUv`(World->UV) ->`ComputeScreenPosAndSize`(UV/World ->RT 픽셀) 

위와 같이 이어지는 **변환 체인**

#### 02-03 `ASnowGenerator::` **PixelAlign**

```cpp 
FVector2D ASnowGenerator::PixelAlign(const FVector& PixelPos)
{
	FVector2D retv = {PixelPos.x, PixelPos.y}; 
	const FVector2D res = RTResolution / SnowFieldSize; // 2048 / 3000 
	
	FVector2D AlignedPos = {floor(retv.X * res.X) / res.X, floor(retv.Y * res.Y) / res.Y};
	return AlignedPos;
}
```
`res`:  
- **단위 변환 계수(Pixel/WorldUnit)**.
- 1 unit(Cm)당 RT pixel 몇개인가?
- `1/res` : RT 픽셀 1개가 월드에서 차지하는 크기(Texel size)

**코드 분석** :  
- `retv.X * res.X` : World->pixel단위 좌표
- `floor`로 Pixel 격자에 Snap (정수 pixel 위치)
- ` / res.x` : 다시 **Pixel->World**로 환산

이 PixelAligned된 `AlignedPos`가 MPC_Snow의 `PivotAndSize`로 들어감.


---
### 03. Material

#### 03-01. RT_SnowHistory UVs Offset 데이터 흐름

: `UpdatePos-DeltaOffset`→ `M_Merge` →`MF_OffsetUV`→ `RT_SnowHistory UVs`

**DeltaOffset**:
```cpp
//ASnowGenerator::UpdatePos()
DeltaOffset = Pos - OldPos;
```
`DeltaOffset` : 1Frame만큼의 Offset을 저장

**Offset Value**:
``` cpp
//ASnowGenerator::MergeRT()
FVector2D ratio = DeltaOffset / SnowFieldSize
M_MegeMID ->SetVectorParameterValue(FName("Offset")),
	FLinearColor(ratio.X, ratio.Y, 0.f, 1.f)
);

```
`Offset` Parameter Set : SnowFieldSize당 DeltaOffset만큼의 비율을 `M_Merge`내에서 찾아서 할당

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/MF_OffsetUV.png"  alt="ShouldWorkOff" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>MF_OffsetUV</strong> </td> <td width="34%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/M_MergeRT.png" alt="M_MergeRT.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_Merge</strong> </td> </tr> </table>
`M_Merge`내부에 있는 `MF_OffsetUV` parameter Offset으로 들어가고, 내부에서는 `MF_OffsetUV`가 이 값을 이용해 UV를 이동시킨다.

```
DeltaOffset → Offset Parameter → MF_OffsetUV → RT_SnowHistory Sampling UV 이동
```

이 덕에 Snow Field가 플레이어를 따라 이동해도 기존 Snow 데이터가 새로운 영역에 맞춰 이동할 수 있다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/OffsetNo.png" alt="NotValid Offset" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Offset NotValid</strong> </td>  </tr> </table>
Offset을 제대로 인식하지못한다면 위와같이 `RT_SnowHistory`를 Offset 시키지못하여서  
그 흐름이 `RT_Snow` Draw까지 받아와서 플레이어 주변에만 그려진다.


#### 03-02. MF_WorldPosToUV

`M_Snow` ∋ `MF_WorldPosToUV` ∋ `MF_WorldPosToUV`
`MF_DeformationProcess` 내부에서 UV 로 Mapping하는 곳은 `MF_WorldPosToUV`하나뿐.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MF_WorldPosTOUV.png" alt="MF_WorldPosTOUV" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>MF_WorldPosUV</strong> </td>  </tr> </table>
노드 흐름은 다음과 같다

```
WorldPosition -> WorldPos(Input) -> ComponentMask(RG) ──┐
                                                        ├─> Subtract
MPC_Snow.PivotAndSize -> ComponentMask(RG) = Pivot ─────┘
                                                        ↓
MPC_Snow.PivotAndSize -> ComponentMask(BA) = Size ──> Divide (A / Size)
                                                        ↓
                                                   Add (+0.5)
                                                        ↓
                                               FunctionOutput "uv"
```

`PivotAndSize`의 RG = Pivot, BA = Size를 가져와서
- `WorldPosition.XY - Pivot` :`WorldPosition`을 pivot기준으로 맞춰주고,
- `/size`: `Size`크기에 맞춘뒤 (0.5, 0.5)만큼 더해준다.
눈 필드의 중심(Pivot)이 UV(0.5, 0.5)에 오도록 Mapping하는 구조이다.

여기서 재미있는건 Material node이지만, C++ `WorldPosToUV`(02-01)와 같은 컨셉이라는 것이다.
```cpp
//WorldPosToUV
FVector2D RetVal = (FVector2D(WorldPos.X, WorldPos.Y) - Pos) / SnowFieldSize +FVector2D(.5f, .5f);
```
같은 개념, 설명은 생략한다.


<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MF_DeformationProcess.png" alt="MF_DeformationProcess" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> MF_DeformationProcess</strong> </td>  </tr> </table>
`MF_DeformationProcess` 내부 art(Normal 재계산, SelfShadow)는 넘어가고 중요한 `MF_WorldPosUV`만 집중했다.  
`MF_WorldPosToUV`의 결과는 `RT_UV`로 **Node Reroute**로 선언 되고, 이 하나의 UV Coordinate 가 들어가는 아래 4군데 다 들어간다.

| 용도                 | 설명                                                               |
| ------------------ | ---------------------------------------------------------------- |
| RT_SnowHistory 샘플링 | R / G / B (Trail / Sunken / Ridge) 마스크 읽기                        |
| RT_AreaMask        | `UV - 0.5` -> abs -> x2 -> OneMinus -> SmoothStep 으로 RT 가장자리 페이드 |
| 노말 재계산             | `MF_NormalFormHeightMapSinglePass` 의 Coordinates                 |
| 셀프 섀도우             | Custom 노드 레이마칭 시작 UV                                             |

그래서 이 함수의 결과가 틀어지면 **RT와 World의 정렬이 전부 깨진다.**  
이것이 `MF_WorldPosToUV`에만 집중한 이유다.

--- 
### 최종정리

Mapping은 생각보다 별거 없다. 핵심은 `Pos`이다.
생각보다 Mapping이라는것은 별게 없다.
1. **공식** : `UV = (WorldPos - Pos) / SnowFieldSIze + 0.5`  
   `Pos`는 Cpp에서 `PixelAlign()`으로`MPC_Snow`의 `pivot`으로 넘기고,  
   Material과 Cpp(`WorldPosToUV`)가 같은 공식을 공유한다.
2. **Scroll** : 
   Player가 움직여 `Pos`가 바뀌면, `DeltaOffset`(정수 Texel단위)만큼 RT내용을 밀어 기존 자국이 World에 고정되도록 한다.
3. **Draw**:
   Footprint / Customtrace는 `WorldPosToUV`로 UV를 구해 DrawMaterial로 RT에 그린다.

> Pos가 RT의 중심(UV 0.5, 0.5)을 정의하고, 그 Pos를 정렬해서 Mapping을 안정시키고, Pos가 바뀔때마다 RT 내용을 DeltaOffset만큼 따라 밀어준다.

한번 더 마지막 정리로 큰 덩어리로 본다면 다음과 같다

> RT를 그대로 그리는것이 아닌 Player 위치를 기준으로 Pivot점 잡고(C++ Function), 다시 Pivot 기준으로 Offset을 해주어야한다
- **C++ Function** : `WorldPositionToUV`, `ComputeScreenPosAndSize`, `AlignedPixel`로 Player가 이동된 지점을 Pivot으로 맞춰주는 역할을 한다
- **Material** : `MF_OffsetUV` 에서 이동한만큼(`DeltaOffset`) 다시 `offset`을 해준다.
