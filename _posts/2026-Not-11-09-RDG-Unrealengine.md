---
layout: post
published: false
title: RDG 001
subtitle: GameThread, RenderThread
date: 2026-10-12 22:24:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
렌더 그래프, [RDG(Render Dependency Graph)](https://dev.epicgames.com/documentation/unreal-engine/render-dependency-graph-in-unreal-engine?utm_source=chatgpt.com&lang=ko)는 UE에서 렌더 명령을 컴파일, 실행할 수 있도록 그래프 데이터 Structure로 기록하는 즉시 모드 API이다.  


----
### RDG 톺아보기

RDG는 단순한 Render Pass를 순서대로 나열한List에서 끝나는것이 아닌, DAG(Directed Acyclic Graph)로 방향성 비순환 그래프를 그리며,  
Pass가 어떤 **Resource**를 **읽고 쓰는지에 따라 의존 관계를 연결한 그래프**이다.

Graph를 탐색해 메모리 사용 최적화하고, CPU, GPU의 RenderPass를 병렬화 한다.

1. Graph의 미사용 리소스 및 Pass Culling

UE에서 RenderPipeline을 수정하기 위해서, 엔진 자체를 뜯어고치기보다, Engine에서 열어둔 RDG graph에 넣어주는 형태로 한다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Graphics/RDG/RDGExample.png" alt="RDGExample" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RDGExample</strong> </td>  </tr> </table>

### 0. RDG에서의 렌더링 파이프라인


### 0. RDG를 사용하는 이유
DX에서 직접구현을 하면 RT에서만 손봐야할것들을 GT에서 수정가능하게 접근 가능토록 만든다면, 이슈가 생기기 마련이다.  
이와같이 UE의 Renderpipeline에서도 개발자가 수정하게 둔다면, 비트로 바꿔야하는 과정이나, 쉐이더 코드로 바인딩할때 등등도 쉽지 않다. 또, 크로스 플랫폼을 지원하는 UE의 특징인, DX, Vulkan같은 API위에서 렌더링 작업을 관리해야하는 시스템으로, 이러한 접근을 쉽게 막아두면 고통이 이만저만이 아니다. 그렇기 때문에, RDG를 이용하는 경우가 많다.


### RDG 실행 유형 4가지
``` cpp
enum class ERDGPassFlags : uint16
{
	/** Pass doesn't have any inputs or outputs tracked by the graph. This may only be used by the parameterless AddPass function. */
	None = 0,

	/** Pass uses rasterization on the graphics pipe. */
	Raster = 1 << 0,

	/** Pass uses compute on the graphics pipe. */
	Compute = 1 << 1,

	/** Pass uses compute on the async compute pipe. */
	AsyncCompute = 1 << 2,

	/** Pass uses copy commands on the graphics pipe. */
	Copy = 1 << 3,

	/** Pass (and its producers) will never be culled. Necessary if outputs cannot be tracked by the graph. */
	NeverCull = 1 << 4,

	/** Render pass begin / end is skipped and left to the user. Only valid when combined with 'Raster'. Disables render pass merging for the pass. */
	SkipRenderPass = 1 << 5,

	/** Pass will never have its render pass merged with other passes. */
	NeverMerge = 1 << 6,

	/** Pass will never run off the render thread. */
	NeverParallel = 1 << 7,

	/** Pass uses copy commands but writes to a staging resource. */
	Readback = Copy | NeverCull
};
```







