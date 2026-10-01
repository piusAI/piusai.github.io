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




Actor 위치를 Raw하게 그대로 받은 형태가 아니기에
`WorldpostoUV`, 와같은 여러 함수들이 존재하고, material자체에도 수학적인 위치 가 뒤엉켜서 어디지점이 눈을 `WorldPositionOffset.z`로 Vertex를 Displacement시켰는지 정확히 알기 힘들다.

, 간단한 Material Parameter Collection를 알아보고, Player 위치를 도대체 어떻게 Grid Mesh의 정확한 위치를 **Mapping**하여 displacement까지 했는지를 로그를 찍어보도록 하겠다.
## 02 Mapping

### Variable
`ASnowGenerator::` SnowFieldSize, 

### Function
`ASnowGenerator::` **WorldPosToUV**
`ASnowGenerator::` **ComputeScreenPosAndSize**




## Offset  - MFOffset추가

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



### MF_DefomrationProcess

## World Position → RT UV

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