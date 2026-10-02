---
layout: post
published: false
title: Snow Deformation ③ - Mapping 집중
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

눈 RT는 플레이어를 따라 움직이는 **Scrolling RT**이다. 다시말해 RT에 어떤식으로 Draw material하는지도 중요하지만, 결국에는 Actor의 `FootprintInfo`, `CustomTrace`의 정보를 다시 Mapping해야하는 지점에 봉착하게 된다.

<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MF_OffsetResult.png" alt="MF_OffsetResult" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RT_Snow</strong> </td>  </tr> </table>

Actor 위치를 Raw하게 RT에 그대로 받은 그리는 형태가 아니기에 동적으로 관리하기 용이하다.
`WorldpostoUV` 와같은 여러 함수들이 존재하고, material자체에도 수학적인 위치 가 뒤엉켜서 어디지점이 눈을 `WorldPositionOffset.z`로 Vertex를 Displacement시켰는지 정확히 알기 힘들다.


1. Variable에서의 Mapping : MPC, Material Parameter, Struct
2. C++ Function의 Mapping
3. Material에서의 Mapping

간단한 Material Parameter Collection를 알아보고, Player 위치를 도대체 어떻게 Grid Mesh의 정확한 위치를 **Mapping**하여 displacement까지 했는지를 로그를 찍어보도록 하겠다.

(완전히 독립적이지는 못하여서 항목이 겹쳐있을 수있음)

---
### 01 MPC, Material Parameter : Variable Mapping
Material ParameterCollection, Material ParameterMapping
가장 쉽게 이해 할 수있는 Variable에서의 Mapping을 알아보려한다.

#### 01-01 Variable

`ASnowGenerator` : (FVector2D)SnowFieldSize, (FVector2D)RTResolution

**SnowFieldSize**:  
Snow FIeld의 크기이다. Default값으로는 (3000x3000)으로 설정했다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/SnowFieldSize.jpg" alt="SnowFieldSize" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SnowFieldSize : 1200x1200</strong> </td>  </tr> </table>
위와 같이 1200x1200으로 SnowField를 설정하면 짤린다.  
`PixelAlign()` (02-03확인)에서 맞춰줘야한다.

**RT_Resolution** :
`RT_Snow`, `RT_SnowHistory`, `RT_Intermediate` - 2k(2048) 고정!

#### 01-02 MPC : Material Parameter Collection

<table width="50%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/MPC.png" alt="MPC" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>MPC</strong> </td>  </tr> </table>

`Material Parameter Collection`이라 하더라도, 구조체와 별반 다르지 않다.
``` cpp
//MPC_Snow
float SnowHeight = 0.0f;
Fvector4 PivotAndSize = Fvector4(860.f, 1035.f, 1000.f, 1000f);
```
이 값들을 활용하는데, 아래와 같은 흐름으로 이어진다


>    ↓  Write ↓                                     ↓   Read  ↓
`cpp_SnowGenerator` -> **`MPC_Snow`** -> `M_Snow`, `M_CustomTrail`, `M_SnowSideHeap`

`Snow_Generator`에서 Set/Write해주고,  
`M_Snow`, `M_CustomTrail`, `M_SnowSideHeap`에다가 Get하여 Read한다


**SnowHeight**:  
정적으로 BP에서 Initialize 값으로 설정할 수있다.
``` cpp
SetScalarParameterValue(this, MPC_Snow, FName("SnowHeight"), SnowHeight);
```

**PivotAndSize**:  
RT가 Scrolling되며 픽셀 정렬을 해야한다 했었다.  
PixelAlign(02-03 확인)을 꼭 해줘야 RT 중심이 항상 **Texel**경계 위에서만 이동한다.
```cpp
// ASnowGenerator::UpdatePos()
const FVector TrackingLocation = IsValid(Pawn) ? Pawn->GetActorLocation() : FVector(-777.f,-777.f, -777.f );
Pos = PixelAlign(TrackingLocation);
UKismetMaterialLibrary::SetVectorParameterValue(this, MPC_Snow, FName("PivotAndSize"), FLinearColor(Pos.X, Pos.Y, SnowFieldSize.X, SnowFieldSize.Y));
```
**Size**:  SnowFieldSize로 바로 들어감
**pivot** : Pos.X, PosY로 Pixel 정렬된 값들로 중앙 위치를 잡는다.


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
pos기준으로 하지 않는다면 이렇게 위치 이격이 일어남  
pos를 기준으로 꼭 빼줘야함!

- `WorldPos - Pos` : Pos를 원점으로 한 `WorldPos` 좌표 : **RT 중심 기준 상대 좌표** 변환
-  `/SnowFieldSize` snow 필드 크기로 나누어 (-0.5~0.5)범위로 Normalize
-  +0.5 원점 중심으로 (0~1) 범위로 만든다.

##### 활용

```cpp
// 사용하는 곳
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

ComputeScreenPosAndSize의 TexsizeScale
`ComputeSizeForRT`는
- ` / SnowFieldSize` : 월드 크기 를 Field 대비 비율(0~1)로 만들고
- ` * RTResolution` : 그 비율을 Pixel수로 환산! 

##### 활용
``` cpp
const FVector2D UV = WorldPosToUV(footprint.Location);
ComputeScreenPosAndSize(UV, 2.0f, footprint.ContactRadius, InPos, InSize);
//Inpos, InSize -> DrawMaterial(Canvas)에 넘길 RT 픽셀 좌표계 값
```
구조체 내의 정보를 가져와서 `ScreenPosAndSize`맞춤

`WorldPosToUv`(World->UV) ->`ComputeScreenPosAndSize`(UV/World ->RT 픽셀)로 이어지는 변환 체인


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
- `floor`로 Pixel 격자에 Snap - 정수 pixel 위치
- ` / res.x` : 다시 **Pixel->World**로 환산

이 PixelAligned된 `AlignedPos`가 MPC_Snow의 `PivotAndSize`로 들어감.


---
### 03. Material

#### 01-01. Offset  - MFOffset추가

`UpdatePos()`에서 현재 Snow 영역의 위치를 계산하고,

```
DeltaOffset = Pos - OldPos;
```

이를 이용하여

```
FVector2D ratio = DeltaOffset / SnowFieldSize;
```

를 만든다.

이 값이 `M_Merge`의

```
Offset
```

Parameter로 들어간다.

Material 내부에서는 `MF_OffsetUV`가 이 값을 이용해 UV를 이동시킨다.

```
DeltaOffset
   ↓
Offset Parameter
   ↓
MF_OffsetUV
   ↓
RT_SnowHistory Sampling UV 이동
```

이 때문에 Snow Field가 플레이어를 따라 이동해도 기존 Snow 데이터가 새로운 영역에 맞춰 이동할 수 있다.



#### 01-02 MF_DefomrationProcess

#### World Position → RT UV

Snow Material에서 현재 Vertex의 World Position을 RT 공간으로 변환한 뒤,

`RT_SnowHistory`에서 해당 위치의 Snow 상태를 Sample한다.

즉

```
3D World Position
      ↓
2D RT UV
      ↓
RT_SnowHistory
      ↓
해당 위치의 Snow 정보
```

라는 구조다.

이 과정 때문에 3D Mesh 위에서 Snow가 변형되는 것처럼 보이지만, 실제 Snow 상태 데이터는 **2D RenderTarget에 저장되어 있다.**




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
   Footprint / customtrace는 `WorldPosToUV`로 UV를 구해 DrawMaterial로 RT에 그린다.

> Pos가 RT의 중심(UV 0.5, 0.5)을 정의하고, 그 Pos를 정렬해서 Mapping을 안정시키고, Pos가 바뀔때마다 RT 내용을 DeltaOffset만큼 따라 밀어준다.