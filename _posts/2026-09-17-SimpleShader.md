---
layout: post
published: false
title: UE .usf의 가장 기본단위
subtitle: Constraint
date: 2026-09-17 19:37:00 +0900
description: Unreal SimpleShader HLSL?
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
쉐이더 프로그래밍의 기초, 가장 간단한 HLSL과 shader 연동을 해보자.

기본 Material에만 활용해본다

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/ShaderPrograming/SimpleShader/ShaderMaterial.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>BBox</strong> </td>  </tr> </table>

#### 폴더 구조
```
SimpleShader/
├── SimpleShader.uplugin
├── Source/
│   └── SimpleShader/
│       ├── SimpleShader.Build.cs
│       ├── Public/
│       │   └── SimpleShaderModule.h
│       └── Private/
│           └── SimpleShaderModule.cpp
└── Shaders/
    └── Public/
        └── SimpleTint.ush
```

PostCinfigInit으로 shaderplugin을 수정해준다
#### SimpleShader.uplugin

```
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "SimpleShader",
	"Description": "SimpleShaderHLSL",
	"Category": "Rendering",
	"CreatedBy": "pius",
	...
	"Modules": [
		{
			"Name": "SimpleShader",
			"Type": "Runtime",
			"LoadingPhase": "PostConfigInit"
		}
	]
}
```


AssetRegistoryModule을 만들어 Asset검색을 할 수있다.

#### SimpleShader.Build.CS

``` cpp
...
	PrivateDependencyModuleNames.AddRange(
		new string[]
		{
			"CoreUObject",
			"Engine",
			"Slate",
			"SlateCore",
			"RenderCore",
			"RHI",
			"Projects" //IPluginManager 사용!...	
		});
...
```


#### SimpleShader.h
``` cpp
#include "Modules/ModuleManager.h"

class FSimpleShaderModule : public IModuleInterface
{
public:

	/** IModuleInterface implementation */
	virtual void StartupModule() override;
	virtual void ShutdownModule() override;
};

```

#### SimpleShader.cpp

``` cpp
#include "SimpleShader.h"
#include "Interfaces/IPluginManager.h"
#include "ShaderCore.h"

void FSimpleShaderModule::StartupModule()
{
	// SimpleShader Plugin내의 Shader 폴더를
	// Material Graph에서 쓸 가상경로 "/Plugin/SimpleShader/"에 매핑
	FString PluginManagerDir =
		FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("SimpleShader"))->GetBaseDir(), TEXT("Shader"));
	AddShaderSourceDirectoryMapping(TEXT("/Plugin/SimpleShader"), PluginManagerDir);
}
...	
IMPLEMENT_MODULE(FSimpleShaderModule, SimpleShader)

```
가상 매핑

### SimpleTint.ush

``` .ush
//#include "/Engine/Public/Platform.ush"
// → hlsl instric이라 안씀!

float3 SimpleTint(float3 InColor, float3 TintColor, float TintAmount)
{
    return lerp(InColor, InColor * TintColor, saturate(TintAmount));
}
```



### Custom Material
- **Output Type**: `CMOT Float3`
- **Inputs**: `InColor` (float3), `TintColor` (float3), `TintAmount` (float)
- **Code**:
```
	return SimpleTint(InColor, TintColor, TintAmount);
```

Include File Path 추가:

```
/Plugin/SimpleShader/Private/SimpleTint.ush
```