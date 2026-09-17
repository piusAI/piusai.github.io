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

### FModuleManager?
Manager이므로 싱글톤패턴으로 Module들을 관리하는 클래스  
[FModuleManager](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Core/FModuleManager?lang=en-US)

```cpp
// 로드되어있지 않으면 자동 로드
FModuleManager::LoadModuleChecked<FPiusJoonModule>("PiusJoonModule");

// 이미 로드된 모듈만 가져오기
FModuleManager::GetModulePtr<FPiusJoonModule>("PiusJoonModule");

bool bLoaded = FModuleManager::Get().IsModuleLoaded("PiusJoonModule");
```
엔진 전체에서 모듈 관련 작업을 한곳에서 처리하기에 싱글톤.

``` cpp
// 명시적으로 singletone 얻고 호출
FModuleManager::Get().LoadModuleChecked<FPiusJoonModule>("PiusJoonModule");
//static 함수가 내부 Get()
FModuleManager::LoadModuleChecked<FPiusJoonModule>("PiusJoonModule");
```
`Get()`을 활용하는지 안하는지 큰 차이가 없다.  
Static을 활용한 코드는, `Warpper`일 뿐이고, 내부적으로 결국 `Get()`호출해서 instance 접근은 동일

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
