---
layout: post
published: true
title: Unreal DataAsset vs PrimaryDataAsset
subtitle: Constraint
date: 2026-06-23 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Unreal
tags:
  - UnrealClass
  - Unreal
---
DataAsset : 참조자가 경로를 직접 알고있어야 접근 가능
PrimaryDataAsset : 엔진이 실행될때 이름을 통해 즉시 접근 가능, **이름 기반 지연 로딩 레지스트리**

내부적으로는 느낌상 ***HashMap +Asset Registry***에 더 가깝다!
``` cpp
TMap<FPrimaryAssetId, FAssetData>
```
와 비슷한 느낌으로의 테이블을 엔진이 관리하는 느낌.

---

## 1. 구조 : DataAsset vs PrimaryDataAsset

#### Asset

``` cpp
class UMyData : public UDataAsset
{
	UPROPERTY()
	int32 Damage;
}
```
데이터 묶음 파일, 별 다른 기능 없음

#### PrimaryAsset

``` cpp
class UMyData : public UPrimaryDataAsset
{
	UPROPERTY()
	int32 Damage;
	
public:
	//요놈 핵심
	virtual FPrimaryAssetId GetPrimaryAssetId() const override; 
}

```

| 기능                  | DataAsset | Primary Data Asset |
| ------------------- | --------- | ------------------ |
| 데이터 저장              | ✅         | ✅                  |
| 에디터에서 편집            | ✅         | ✅                  |
| 다른 에셋이 직접 참조        | ✅         | ✅                  |
| **AssetManager 등록** | ❌         | ✅                  |
| **이름으로 검색/로드**      | ❌         | ✅                  |
| **번들 단위 로드/언로드**    | ❌         | ✅                  |
| **쿠킹/패키징 규칙 지정**    | ❌         | ✅                  |

## 2. 접근 : DataAsset vs PrimaryDataAsset
#### Asset
``` cpp
UMyData* Data = LoadObject<UMyData>(
	nullptr,TEXT("/Game/Asset/Weapons/DA_Sword.DA_Sword") // 경로로 직접 찾기!
	);
		
```
반드시 직접 하드코딩 / 다른 에셋이 참조하고 있어야함!

#### Primary Asset
``` cpp
FPrimaryAssetId Id( "MyDataType", "MyData");

UAssetManager::Get().LoadPrimaryAsset(
	Id, TArray<FName>(), FStreamableDelegate::CreateLambda([Id]()
	{
		UMyData* Data = UAssetManager::Get().GetPrimaryAssetObject<UMyData>(Id);
	})
);
```
AssetManager를 통해 ID로 요청 가능!

> AssetManager 조회-> 실제 패키지 경로 획득 -> 로드

: Game Ability System에서의 Ability, GameplayEffect, CharacterClassData, ItemData같은 것들이
Primary Data Asset을 많이 쓰는 이유!



## 3. 선택 : DataAsset vs PrimaryDataAsset

#### DataAsset
- 다른 BP / Asset이 직접 들고 있을때
- 작은 규모의 프로젝트
- EX) Character BP가 멤버 변수를 들고있는 스탯 데이터

#### PrimaryDataAsset
- 런타임에 동적으로 Load/UnLoad
- 어떤 Asset이 존재하는지 목록 검색해야할때
- EX) 아이템 100종류를 필요할 때만 꺼내 쓰는 경우