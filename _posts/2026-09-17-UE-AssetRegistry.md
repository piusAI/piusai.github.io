---
layout: post
published: true
title: AssetRegistry
subtitle: Constraint
date: 2026-09-17 19:37:00 +0900
description: Unreal AssetRegistry?
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
AssetRegistry는 UE editor가 Asset을 찾기 쉽게 관리하는 **Asset 메타데이터 검색 시스템** ! !

[UE Docs : Asset Registry](https://dev.epicgames.com/documentation/unreal-engine/asset-registry-in-unreal-engine)  
위 Docs 참고!

## AssetRegistry와 Module의 활용

```
#include "AssetToolsModule.h"
#include "AssetRegistry/AssetRegistryModule.h"


	FAssetRegistryModule& AssetRegistryModule=
	FModuleManager::Get().LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");
	
	FARFilter filter;
	filter.bRecursivePaths = true;
	filter.PackagePaths.Emplace("/Game");
	filter.ClassPaths.Emplace(UObjectRedirector::StaticClass()->GetClassPathName());
	filter.ClassPaths.Emplace(UParticleSystem::StaticClass()->GetClassPathName());
	TArray<FAssetData>OutAssetDatas;
	AssetRegistryModule.Get().GetAssets(filter, OutAssetDatas);
```

AssetRegistoryModule을 만들어 Asset검색을 할 수있다.

### Filter?
검색 시스템에서 조건을 넣어 찾을 수 있다. "검색 조건에 맞는 에셋을 찾아줘"  
[FARFilter](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/CoreUObject/AssetRegistry/FARFilter?application_version=5.5) 타입의 `Struct`로 
- PackageName 
- PackagePath 
- bRecursiveClasses ; 상속을 재귀적으로!
- bRecursivePaths ; 경로를 재귀적으로


```
FARFilter filter;
filter.bRecursivePaths = true;
filter.PackagePaths.Emplace("/Game");

//ObjectRedirector의 클래스찾기
filter.ClassPaths.Emplace(UObjectRedirector::StaticClass()->GetClassPathName());
//Mateiral Instance 클래스도
filter.ClassPaths.Emplace(UMaterialInstance::StaticClass()->GetClassPathName());
filter.bRecursiveClasses=true; //클래스 상속 재귀
```
`bRecursiveClassess`를 켜야 `UMaterialInstance`상속받는 `UMaterialInstanceConstant`같은 자식 클래스도 검색 결과에 포함!


배열인 `ClassPaths`이어서 Emplace로 들어가면 `OR` 로 검색 가능!

```
TArray<FTopLevelAssetPath> ClassPaths; //이렇게 ClassPaths가 들어감
...
struct FTopLevelAssetPath
{
	...
	private:
		FName PackageName;
		FName AssetName;
}
```
