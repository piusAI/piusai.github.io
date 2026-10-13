---
layout: post
published: true
title: "AIZ Algorithgm Ver3 : CPU 적응형 Terrain 알고리즘"
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-13 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
>**상태**: 진행 중  
>**Ver3 : CPU 레퍼런스 구현** **범위**: 엔진 통합 계층(Component → SceneProxy)까지

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://www.youtube.com/watch?v=bIsvpShIB-k&feature=youtu.be" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/bIsvpShIB-k/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>Version03 Done</strong>
    </td>
  </tr>
</table>


> 

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://www.youtube.com/watch?v=Hv6IKS_8Sp8&feature=youtu.be" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/Hv6IKS_8Sp8/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>SphereTerrain CPU Test</strong>
    </td>
  </tr>
</table>
위 Ideation으로 시작했었다



### 1. 큰 그림
#### 1-1. 한 줄 요약

CPU에서 CBT/LEB 분할 → 메시 생성 → GPU 업로드 후 렌더링.
- `FIZLEB`가 CPU에서 분할하고 메시를 생성/수정한다.
- `Proxy::GetDynamicMeshElements()`가 그 결과를 GPU 렌더링용 메시로 구성한다.
- 흐름: **CBT/LEB 분할(CPU) → Vertex Index 생성(CPU) → GPU 업로드 → 렌더링**


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/AIZ/V3Arhitecture.png" alt="SO002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="40%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/AIZ/IZV3Algorithm.png" alt="SO003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>

- `FLEB`가 CPU에서 분할, 메시 생성 수정후, `Proxy::GetDynamicMeshElements()`가 결과를 GPU 렌더링용 메시를 구성하는 구조
-
- 추후버전 :
  1. GPU ComputeShader로 RT 샘플링후 CBT/LEB 분할 -> 메시생성 
  2. GPU Buffer ->Indeirect Draw..?

#### 1-2. 계층 구분
구현은 **엔진 통합 계층**과 **알고리즘 계층**으로 나뉨.

| 계층        | 클래스                                                                  |
| --------- | -------------------------------------------------------------------- |
| **UE 통합** | `UAIZComponent`, `UIZGridComponent(_v03)`, `FIZGridSceneProxy(_v03)` |
| **알고리즘**  | `FIZCBT`, `FIZLEB`, `FIZRefineMetric`                                |


#### 1-3. 현재 한계 & 다음 버전

- 한계 : Ver3은 **CPU 레퍼런스**다. RT에서 `Refine`을 CPU로 돌리고 Mesh를 **매 프레임 업로드**한다.
- 다음 버전  (GPU버전 이전)
	1. GPU ComputeShader로 RT Sampling -> CBT/LEB분할 -> 메시생성(CBT BitField -> GPU Buffer, Refine -> Compute)
	2. GPU Buffer -> Indirect Draw 검토


### 2. Data 흐름

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/AIZ/V3DataFlow.png" alt="V3DataFlow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>V3DataFlow</strong> </td>  </tr> </table>
- `FSceneProxy::GetDynamicMeshElements()`
  - RT에서 호출되는 CPU 함수(GPU 실행아님).
  - 렌더링 할 메시 요소 수집해 `MeshDraw`명령으로 넘기는 역할만.
  - 참고 : [MeshSceneProxy::GetDynamicMeshElements](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Engine/FStaticMeshSceneProxy/GetDynamicMeshElements)
- Data 방향 : 기본적으로 [GT->RT](https://piusai.github.io/engine/2026/10/12/GameRenderThread.html) 단 방향이고, 통계(`DynamicMeshElements()`)에서만 `RT->GT`
- `ENQUEUE_RENDER_COMMAND`는 GT가 넣은 순서대로 RT에서 실행된다.

---

### 3. 상호작용 대상 결정

`IZGridComponent::ResolveInteractionData` - `FIZInteractionData` return

우선순위:
1. `trackedActor` - 직접 지정한 AActor (`TWeakObjectPtr<AActor> TrackedActor;`)
2. 첫 PlayerController의 Pawn -> 없으면 월드 내 아무 Pawn(템플릿 제외)
3. `ActiveAIZComponents` static에서 (같은 World내)
4. Default : GridComponent 위치


### 4. Code Review


#### 4.1 AIZTessellationShader.h/.cpp
- 모듈 Startup/Shutdown.

#### 4.2 IZCVars.h / .cpp

``` cpp
int32 CVarIZEnable;
int32 CVarIZDebugPrintProxyState;
int32 CVarIZDebugLevelColor;
int32 CVarIZDebugWireframe;
int32 CVarIZMaxDepth;
float CVarIZRefineDistance;
```

- CVar로 toggle가능
- 모두 `ECVF_RenderThreadSafe`: RT에서 읽는다.
- `IZ.Enable`, `IZ.Debug.LevelColor`, `IZ.Debug.Wireframe` 로 시각화

#### 4.3 AIZComponent - GT전용 데이터 공급
- Super Class : `UActorComponent` (Render/Physics없이 순수 Telemetry 수집용)
- **UpdateInteractionTelemetry** : 위치, 속도, 지면 LineTrace로 `bIsContacting` 판정
- `static TArray<TWeakObjectPtr<UAIZComponent>> ActiveAIZComponents` 에 등록해 Grid가 찾기 쉽도록 함.
  - `OnRegister / BeginPlay`에서 추가하고 `OnUnregister/EndPlay`에서 제거.

| 함수                               | 역할                        |
| -------------------------------- | ------------------------- |
| `OnRegister()`                   | Engine에 컴포넌트 등록 시 필요한 초기화 |
| `CreateRenderState_Concurrent()` | 렌더링 시스템이 사용할 상태 생성        |
| `OnCreatePhysicsState()`         | 물리 시뮬레이션에 필요한 상태 생성       |


#### 4.4 IZGridcomponent_v03.h/.cpp
- Super Class : `UMeshComponent`
- `OnRegister` : `SetMaterial`로 Proxy가 오래된 material 잡는것 방지
- `UPrimitiveComponent::PostEditChangeProperty` : Editor변경시 후처리 용도
- `CalcBounds`: `Terrain size`, `heightfielddisplacement`로 계산

##### `UIZGridComponent::UpdateRenderThreadState()`
- 역할 : `IZGridSceneProxy`인 RT에맞는 구조체로 보내기위함
- `PrimitiveComponent::FPrimitiveSceneProxy* SceneProxy`을 `FIZGridSceneProxy`로 casting해서 활용
- `ENEQUEUE_RENDER_COMMAND`로 RTThread로 복사 보냄

##### `UIZGridComponent::ResolveInteractionData()`
- 3장의 우선순위에 따라 `FIZInteractionData`를 만들어 반환



#### 4.5 IZTypes.h - 데이터 구조체


| 구조체                          | 만드는 곳                                                               | 쓰는 곳                                                                              | GT→RT 복사                  |
| ---------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------- |
| `FIZInteractionData`         | `UAIZComponent::GetInteractionData`), `ResolveInteractionData`의 대체값 | `FIZLEB::Refine` → `FIZRefineMetric::ComputeTargetDepth`, 프록시 `RTInteractionData` | O (`DataCopy`)            |
| `FIZRefineParams`            | `UIZGridComponent` 멤버 변수                                            | `Refine`, `BuildMeshData`, `ComputeTargetDepth`, 프록시                              | O (`EffectiveParamsCopy`) |
| `FIZMeshData`                | - `IZGridSceneProx` 멤버변수<br>`FIZLEB::BuildMeshData` (RT)            | 프록시 `RTMeshData` → `GetDynamicMeshElements`                                       | X (RT 내부)                 |
| `FIZCompensationParams`      | `UAIZComponent` UPROPERTY                                           | **현재 사용처 없음** (placeholder)                                                       | X                         |
| `FIZTimedDetailSample` (v03) | 프록시 `TimedDetailSamples`                                            | v03 `Refine`의 시간 보상                                                               | X (RT 내부)                 |

`FinteractionData`:  
1. `ResolveInteractionData`
2. `FIZLEB::Refine()`, `FIZLEB::BuildMeshData()`
3. `FIZRefineMetic::ComputeTargetDepth()`

`FIZCompensationParams`:  
- Compensation 버전 넣을때 추가 (현재 사용x)

`FIZRefineParams` :  
1. GT->RT로 Copy 보낸 후, proxy에서 Refinement setting정보 받기위함
2. `FIZLEB::Refine()`, `FIZLEB::BuildMeshData()`
3. `FIZRefineMetic::ComputeTargetDepth()`

`FIZMeshData`:  
1. IZGridSceneProxy - Member Variable
2. `FIZLEB::BuildMeshData()`


#### 4.6 시간 보상 -`IZTemporalTessellationCompensation::ComputeWeight()`

상호작용이 지나간 뒤 디테일을 즉시 끄지 않고, hold 이후 지수 감쇠시킨다.

감쇠 수식

$$w=e^{k(h-t)}$$

($k$ = DecayRate, $t$ = Elapsed, $h$ = Hold)

```cpp
const float Weight = FMath::Exp(-FMath::Max(0.0f, DecayRate)* (Elapsed-Hold));
```



---
### 5. Terrain Tessellation 알고리즘

#### 5.1 IZRefineMetric.h -`ComputeTargetDepth`

- **Predictive AIZ**: 상호 작용 object 속도 벡터로 진행 방향 영역 사전 활성화
$$\tilde{\mathbf p}(\delta) = \mathbf p + \delta \mathbf v$$
``` cpp
const FVector PredicatedSource = Interaction.Location + Interaction.Velocity * PredictiveDeltaSeconds;
```


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/AIZ/ComputeTargetDepth.png" alt="ComputeTargetDepth" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ComputeTargetDepth</strong> </td>  </tr> </table>

| 조건                | 결과                                                                       |
| ----------------- | ------------------------------------------------------------------------ |
| `Dist >= MaxDist` | `0` (먼 곳: 기저 레벨)                                                         |
| `Dist <= MinDist` | `DepthLimit` (가까운 곳: 최대 디테일)                                             |
| 그 사이              | `round( smoothstep(1 - (Dist-MinDist)/(MaxDist-MinDist)) * DepthLimit )` |

**Depth 한계 계산** (`FIZLEB.h`, `FIZLEB::Refine()`에서 사용)

``` cpp
//FIZLEB.h
const int32 EffectiveMaxDepth = FMath::Min(Params.MaxSubdivLevel * 2 - 1, MaxDepth());

const int32 MinimumDepth = FMath::Min(2 * BudgetSafeMinLevel - 1, EffectiveMaxDepth);
```

- `MaxSubdivLevel = 11`, `MinSubdivLevel = 5` (default)일 때 각각 depth **15**, **9**가 나온다.


#### 5.2 FIZCBT - bitfield heap
- bBinary heap 구조 (Concurrent Binary Tree)
- `SplitNode` / `MergeNode`비트 켜고 끄고 `ChackedLeafCount`갱신
- 1-Based 노드 indexing으로 부모/자식 관계 간결히 계산

#### 5.3 FIZLEB.h - longest edge bisection

| 함수                             | 역할                              |
| ------------------------------ | ------------------------------- |
| `Reset()`                      | 사각형을 삼각형 2개(NodeID 2, 3)로 시작    |
| `Refine()`                     | 스택으로 노드 순회하며 목표 depth까지 분할/병합   |
| `SplitOne` / `SplitConforming` | 빗변 중점 분할. **T-junction 방지의 핵심** |
| `BuildMeshData`                | 리프 노드로 `FIZMeshData` 생성         |
| `ValidateTJunction`            | 결과 검증용                          |

---

### 6. 플러그인 파일 구조 (Ver3)

```
AIZTessellaitonShader/
├── AIZTessellaitonShader.uplugin
├── Source/
│   └── AIZTessellaitonShader/
│       ├── TessellationShader.Build.cs
│       ├── Public/
│       │   └── AIZTessellationCompoent.h
│       │   └── AIZComopnent.h
│       │   └── FIZCBT.h
│       │   └── FIZLEB.h
│       │   └── IZCVars.h
│       │   └── IZGridComponent.h
│       │   └── IZGridSceneProxy.h
│       │   └── IZRefineMetric.h
│       │   └── IZTemporalTessellationComponent.h
│       │   └── IZTypes.h
│       └── Private/
│           └── AIZComponent.cpp
│           └── AIZTessellationShader.cpp
│           └── IZCVars.cpp
│           └── IZGridComponent.cpp
│           └── IZGridSceneProxy.cpp
└── Shaders/
    └── .ush, .usf
```
