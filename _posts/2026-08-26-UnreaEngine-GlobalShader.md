---
layout: post
published: false
title: Unreal GlobalShader를 알아보자
subtitle: Constraint
date: 2026-08-26 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---

`PiusShader.usf` Custom Shader를 등록해놓은 이후  
Cpp에 Compile하고 싶다면 GlobalShader에 등록해야한다


``` cpp
#include "GlobalShader.h"

class FPiusShaderCS : public FGlobalShader
{

...

};
```
이런 식으로 `FGlobalShader`를 먼저 상속받았다

---

## Engine의 Shader System에 등록하는 MACRO


UE는 모든 Shader를 **전역 Registry**에 등록해서 관리함  
Shader Compile, Hot-reload, Caching, Reflection 등이 모두 등록 정보를 기반으로 동작.

:Macro 내부에 까지 들어가는것은 UE engine 프로그래머까지이니,
**매크로** ***인자***((How)와, 하려는 ***행위***(What) ***뜻하는바***(Why)에만 집중해보자

#### 01
``` CPP
	DECLARE_EXPORTED_SHADER_TYPE(FPiusShaderCS, Global, PIUSSHADERS_API)
```
- `FPiusShaderCS` : 이 Class 자기자신 타입
- `Global` : Shader 종류 카테고리
- `CHARACTERDOTSHADERS_API` :`PiusShaders`라는 Module로 묶어놓았지만
  다른 Module에서도 참조 할 수있도록 DLL Export 위함
--> Shader Type Declare하고, Shader Class 맞춰줌

#### 02 
```
SHADER_USE_PARAMETER_STRUCT(FPiusShaderCS, FGlobalShader)
```
- ShaderClass, ShaderParentClass
--> ShaderClass, ShaderParameterMetadata Binding


#### 03
```cpp
// ShaderParameter Structure
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters,)
	SHADER_PARAMETER(FVector2D, ViewportSize)

	SHADER_PARAMETER_UAV(RWTexture2D, SceneColorOutput)
END_SHADER_PARAMETER_STRUCT()
```
Parameter의 이름 잘 맞춰주기!

#### 04

```cpp

IMPLEMENT_GLOBAL_SHADER(FPiusShaderCS, "/CustomShaders/PiusShader.usf", "PiusShader", SF_Compute)

#define \IMPLEMENT_GLOBAL_SHADER(ShaderClass,SourceFilename,FunctionName,Frequency) \IMPLEMENT_SHADER_TYPE(,ShaderClass,TEXT(SourceFilename),TEXT(FunctionName)\
,Frequency)

```
- 가상주소에다가 PiusShader.usf를 올려놓은걸  Source FileName으로 지정
--> PiusShader Function을 Compute Shader에 올림


---
#### 01 
```cpp

RDG_EVENT_SCOPE(GraphBuilder, "PiusShader");
```
`RDG_EVENT_SCOPE_CONSTRUCT`에서 Emplace로 새로운  
"RDG Scope/Event 시작~"이라 등록


---

``` cpp

void FPiusShaderCSInterface::AddPass_RenderThreads(FRRGBuilder& GraphBuilder, FGlobalShaderMap* InShaderMap, uint32 InResolution, const ...)
{
	RDG_EVENT_SCOPE(GraphBuilder, "PiusShader");
	TShaderMapRef<FPiusShaderCS> ComputeShader(InShaderMap);

	
}
```

`IMPLEMENT_GLOBAL_SHADER(FPiusCS, "/CustomShaders/PiusShade.usf", "PiusCS", SF_Compute);`으로 등록했기떄문에  
Editer 부팅시, 체크해서 순열에 올라감.  
GPU에 올라갈 준비가 끝난 Compile된 Shader Byte code를 가리키는 핸들

``` cpp
FPiusShaderCS::FParameters* PassParameter = GraphBuilder.AllocParameters<FPiusShaderCS::FParameters>();
```
RDG 특징상, `AddPass`호출하는 시점에 이 패스가 실제 언제 **실행**될지 모름.  
`Execute()`시점까지 지연되기 때문, Pass Lambda안에서 



``` cpp
ENQUEUE_RENDER_COMMAND(PiusShader)(
[ ] (FRHICommandListImmediate& RHICmdList) {


	FRDGBuilder GraphBuilder(RHICmdList);
	...
	
	GraphBuilder.Execute();

});
```

`FRHICommandListImmediate& RHICmdList` : RT에서 GPU 명령 처리하는 최상위 RHI 커맨드 list 핸들



## First Shader Class?
```cpp
class FMyShaderCS : public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FMySHaderCS);
	SHADER_USE_PARAMETER_STRUCT(FMyShaderCS, FGlobalShader);
	
	//Engine Template(SetShaderParameters, AddPass 등)이
	// 이 클래스 내부의 parameter Type을 FParameters라는 통일된 이름으로
	// 참조 할 수 있도록 연결!
using FParameters = FMyShaderParameters;
};
```
### 외부에 using(별칭)을 쓰는 이유
1. Header 분리, 모듈화 : 
   Parameter가 점점 커질때, Structure를 Shader Class 외부 / 별도 파일로 분리해서 헤더 가독성 높혀야함
2. 코드 재사용성: VS, PS, ComputeShader가 하나의 Parameter Structure를 공유해야한다면, 외부에서 선언된 structure하나를 각 클래스에서 `using FParameters = 공유 struct`로 불러와 재사용 할 수있다.
3. 단독 사용시 단축 가능 :
   특정 Shader만 단독으로 쓰이는 parameter라면 굳이 외부로 만들지 않고, class 내부에 `BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )`로 즉시 선언하면 using 생략도 가능다른 directory로 Shader에 FParameters유형을 할당
