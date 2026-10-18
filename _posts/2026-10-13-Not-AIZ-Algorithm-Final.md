---
layout: post
published: true
title: "AIZ Algorithgm Ver9 : ComputeShader Terrain 알고리즘"
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-13 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
>**상태** : 최종 알고리즘 완료
>**구현** **범위**: 엔진 통합 계층(Component → SceneProxy)까지

**아키텍쳐 흐름**:
1. sceneProxy로 GT->RT로 복사하도록
2. usf 쉐이더로 

**최적화 방식** :
  1. 계속 Update 되지 않도록
     - Visited 범위를 두어 움직이는 범위
     - 가장 최근에 방문하는 위치를 범위로 지정해서 stamp형식으로 찍도록(계속 누적 x)
  2. 최대한 HighResolution -> low으로 두도록
     - RenderTarget 사이즈(1800cm)와 같이 RT size으로 reset될떄 tessellation Low로
  3. Priority_queue로 High Resolusion으로 update되는 우선순위를 player주변으로 둠

**연구 목적** : 
1. 1개의 바닥과 비교하기 떄문에 정적 메시와 완전 같은 


**수식 정리** :


**Future Study**:
1. interaction에만 vtx
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://www.youtube.com/watch?v=5quB66ROCn0" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/5quB66ROCn0/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>Final AIZ Teslleation</strong>
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

### 6. 플러그인 파일 구조 (Ver.Final)

```
AIZTessellationShader
├── AIZTessellationShader.uplugin
├── Source/
│   └── AIZTessellationShader/
│       ├── AIZTessellationShader.Build.cs
│       ├── Public/
│       │   ├── IZGridComponent_v09_02.h
│       │   ├── IZGridSceneProxy_v09_02.h
│       │   ├── IZGridComponent_v05.h                 # 상속 컴포넌트
│       │   ├── IZGridSceneProxy_v05.h                # 상속 씬 프록시
│       │   ├── FIZLEB_v08_1_1.h                      # v09_02에서 사용하는 LEB
│       │   ├── FIZLEB_v05.h                           # v05 프록시 기반 타입
│       │   ├── FIZCBTSumTree.h
│       │   ├── IZTypes.h
│       │   ├── IZTemporalTessellationCompensation.h
│       │   ├── IZRefineMetric.h
│       │   ├── AIZComponent.h                         # 상호작용 위치·접촉 데이터
│       │   └── IZCVars.h
│       └── Private/
│           ├── IZGridComponent_v09_02.cpp
│           ├── IZGridSceneProxy_v09_02.cpp
│           ├── IZGridComponent_v05.cpp                # 상속 Tick 및 상호작용 수집
│           ├── IZGridSceneProxy_v05.cpp               # 기반 프록시 생성·기본 동작
│           ├── AIZComponent.cpp
│           ├── IZCVars.cpp
│           └── AIZTessellationShader.cpp              # 셰이더 경로 등록
└── Shaders/
    └── Private/
        └── AIZGenerateGridMesh_v09_02.usf
```

**.usf 활용**:
1. AIZTessellation Shader: Shader Directory 가상 경로 매핑
2. IZGridSceneProxy_09_02:
   -> 2.1. Shader Class Declare + Parameter Struct 매크로 등록
   -> 2.2 `IMPLEMENT_GLOBAL_SHADER` : .usf를 Compute Shader로 등록
3. IZGridSceneProxy::GenerateMesh_RenderThread()(RT):  `AddPass()`로 RDGPass ComputeShader 실행 요청


  

|단계|실행 위치|하는 일|CPU에 두는 이유 / GPU에 두는 이유|
|---|---|---|---|
|입력·설정 수집|게임 스레드 (CPU)|플레이어/액터 위치, 속도, 접촉 상태, 파라미터, Transform 복사 후 `ENQUEUE_RENDER_COMMAND`로 렌더 스레드에 전달|게임 로직과 연결된 데이터라 CPU가 자연스러움|
|위치 스무딩·예측|렌더 스레드 (CPU)|상호작용 위치 스무딩과 예측 입력 계산|소량의 상태 기반 계산|
|재방문 판정·footprint 감쇠·컬링|렌더 스레드 (CPU)|셀 캐시 확인, 시간 감쇠, 거리 컬링|이력을 유지하며 분기가 많은 로직|
|갱신 임계값 판정|렌더 스레드 (CPU)|이번 프레임에 토폴로지를 갱신할지 결정|조건 판단 위주, 병렬화 이득 없음|
|LEB 분할·CBT 상태 관리|렌더 스레드 (CPU)|`FIZLEB_v08_1_1`, `FIZCBTSumTree`로 토폴로지와 활성 삼각형 확정|이웃·우선순위·이전 상태에 의존해 병렬화가 어려움|
|격자 좌표·색상 배열 준비|렌더 스레드 (CPU)|활성 삼각형의 `LeafGridVertices` 구성 후 GPU 버퍼로 업로드|GPU 입력을 만드는 단계|
|정점 XYZ 위치 생성|GPU (Compute Shader)|`AIZGenerateGridMesh_v09_02.usf`에서 정점별 로컬 위치 계산|정점끼리 독립적이고 연산이 동일해서 병렬 처리에 적합|
|삼각형 인덱스 생성|GPU (Compute Shader)|같은 셰이더에서 인덱스 순서 생성|위와 동일 (스레드 1개 = 정점 1개, `numthreads(64,1,1)`)|
|UV·접선·색상 스트림, Vertex Factory|렌더 스레드 (CPU)|보조 버텍스 스트림 구성, GPU 버퍼를 Vertex/Index Buffer로 연결|렌더링 리소스 구성 작업|
|Mesh Batch 제출|렌더 스레드 (CPU)|`GetDynamicMeshElements`에서 제출|렌더 명령 준비|
|최종 래스터화|GPU|제출된 메시를 화면에 그림|GPU 고유 역할|

**요약**

|구분|담당 작업의 성격|
|---|---|
|CPU (GT + RT)|분기·상태·이력이 많은 판단 로직, 토폴로지 확정, 리소스 구성과 제출|
|GPU|같은 연산을 수천 개 정점에 반복하는 위치·인덱스 계산, 래스터화|