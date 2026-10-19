---
layout: post
published: true
title: "AIZ Algorithgm Ver9 : ComputeShader Terrain 알고리즘"
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-13 21:22:00 +0900
description: AIZ Algorithm
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
> **상태** : 최종 알고리즘 완료 (v09_02)
> **구현 범위** : 엔진 통합 계층(Component → SceneProxy → RDG Compute Shader → FMeshBatch)까지

**아키텍쳐 흐름**:
1. Game Thread(GT)에서 상호작용 (위치/속도/접촉) 수집, `FPrimitiveSceneProxy`로 `ENQUEUE_RENDER_COMMAND`를 통해 GT→RT로 복사
2. RenderThread(RT)에서 필터링(smoothing 재방문 footprint)와 LEB 토폴로지 확정을 수행
3. 확정된 Leaf 정점의 격자 좌표를 업로드, `.usf` ComputeShader가 정점 위치와 index를 GPU pooling buffer에 기록
4. `GetDynamicMeshElements`에서 `FMeshBatch`제출 후 래스터화

**최적화 방식** :
  1. 계속 Update 되지 않도록
     - Visited 셀(100cm 양자화)을 두어 이미 고해상도로 만든 Cell 재방문해서 Tessellation 다시 하지않도록
     - 가장 최근 방문 위치를 Stamp(footprint) 형태로 찍고 시간 감쇠(계속 누적하지 않음 최대 64 FIFO) 
     - 이동한 임계값(50cm) / 접촉 변화 / 설정 변경 / footprint 개수 변화 / 0.25s 감쇠 틱 중 하나일때만 `Refine`
  2. 최대한 HighResolution → Low로 되돌리기
     - RenderTarget 크기(`FootprintCullDistance` :1800cm)를 벗어난 footprint는 시간감쇠를 기다리지 않고 즉시 제거 해당 영역은 기저 Tessellation으로 복귀
  3. PriorityQueue로 High Resolution 갱신 우선순위를 Player 주변에 둠
     - 우선순위 = `deficit * 1e5 -dist` (가까울수록, 부족한 깊이가 클수록 먼저 분할)

**연구 목적** : 
1. 정적인 단일 바닥 메시와 시각적으로는 유사한 결과를 내면서 삼각형 수와 비용을 줄이기.



**Ideation**:
1. Interaction한 영역에에만 vtx 생성되도록
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





### 1. 큰 그림

#### 1-1. 표 요약

| #   | 단계                          | 실행 위치                     | 하는 일                                                                                          | 실행 조건 / 비고                                              |
| --- | --------------------------- | ------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| 1   | 입력·설정 수집                    | 게임 스레드 (CPU)              | 위치, 속도, 접촉 상태, 파라미터, Transform을 복사해 `ENQUEUE_RENDER_COMMAND`로 렌더 스레드에 전달                      | 매 프레임 (`GFrameNumber` 중복 방지)                            |
| 2   | 위치 스무딩·예측                   | 렌더 스레드 (CPU)              | `VInterpTo`로 위치 스무딩, 속도에 look-ahead 비율 적용                                                     | 매 프레임                                                   |
| 3   | 재방문 판정                      | 렌더 스레드 (CPU)              | 셀 키 계산, `VisitedCells` 조회, FIFO 캐시 관리                                                         | 매 프레임                                                   |
| 4   | footprint 추가·거리 컬링·시간 감쇠 제거 | 렌더 스레드 (CPU)              | 발자국 샘플 추가, `FootprintCullDistance` 밖 제거, hold-then-exp 감쇠로 만료된 샘플 제거                          | 매 프레임                                                   |
| 5   | 갱신 여부 판정                    | 렌더 스레드 (CPU)              | 이동량, 접촉 변화, 설정 변경, footprint 개수 변화, 0.25초 감쇠 틱 중 하나라도 해당하면 갱신. 아니면 이전 GPU 버퍼를 그대로 사용하고 종료     | 여기서 걸러지면 6~11단계 전부 생략                                   |
| 6   | LEB 분할                      | 렌더 스레드 (CPU)              | `FIZLEB_v08_1_1::Refine`: `Reset()` 후 균일 선분할, 우선순위 힙 기반 분할, `SplitConforming`으로 T-junction 방지 | 5단계 통과 시. 매번 처음부터 재구성 (증분 아님)                           |
| 7   | CBT 합 트리 갱신                 | 렌더 스레드 (CPU)              | `SplitOne`마다 `SumTree.SplitLeaf` 호출로 합 트리 동기 갱신                                               | 현재 렌더링 경로에서는 조회되지 않음. 활성 삼각형은 `Triangles`의 `bAlive`로 판단 |
| 8   | 토폴로지 변경 감지                  | 렌더 스레드 (CPU)              | 활성 리프 NodeID 목록을 이전 목록과 비교                                                                    | 같으면 9~13단계 생략                                           |
| 9   | 격자 좌표·색상 배열 준비              | 렌더 스레드 (CPU)              | 활성 삼각형마다 정점 3개의 정수 격자 좌표와 깊이 색상을 `Scratch` 배열에 기록                                             | 토폴로지 또는 설정이 바뀐 경우만                                      |
| 10  | 버퍼 용량 판단                    | 렌더 스레드 (CPU)              | `MaxTriangles×3`으로 사전 할당, 용량 부족 시에만 재할당                                                       | 평소에는 재할당 없음                                             |
| 11  | RDG 구성·업로드                  | 렌더 스레드 (CPU)가 구성, 실행은 GPU | 격자 좌표 업로드, CS 디스패치 등록, 풀링 버퍼 추출                                                               | 토폴로지 변경 시만                                              |
| 12  | 정점 위치 변환                    | GPU (Compute Shader)      | 격자 좌표를 로컬 좌표로 변환: `XY = (GridPoint / G − 0.5) × TerrainSize`, Z = 1.0                         | 스레드 1개 = 정점 1개 (`numthreads(64,1,1)`)                   |
| 13  | 인덱스 기록                      | GPU (Compute Shader)      | 정점 3개마다 `(0,2,1)` 순서로 순차 인덱스 기록 (정점 공유 없음)                                                    | 같은 셰이더, 와인딩 순서만 결정                                      |
| 14  | UV·접선·색상 스트림 구성             | 렌더 스레드 (CPU)              | UV는 `GridPoint × InvGridSize`, 접선은 상수 (1,0,0)/(0,1,0)/(0,0,1), 색상 복사                          | GPU로 옮길 수 있는 부분 (향후 과제 후보)                              |
| 15  | Vertex Factory 구성           | 렌더 스레드 (CPU)              | GPU 풀링 버퍼를 Position/Index 버퍼로 연결, 보조 스트림 바인딩                                                  | 토폴로지 변경 시만                                              |
| 16  | Mesh Batch 제출               | 렌더 스레드 (CPU)              | `GetDynamicMeshElements`에서 `FMeshBatch` 구성 후 `Collector`에 제출                                  | 매 프레임, 뷰마다                                              |
| 17  | 최종 래스터화                     | GPU                       | 제출된 메시를 화면에 그림                                                                                | 매 프레임                                                   |


### 2. Main Pipeline

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/AIZ/V09Algorithm.png" alt="V9Algorithm" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>V9Algorithm</strong> </td>  </tr> </table>

#### 2.1 `UpdateInteractionTelemetry()`
- Class : `UAIZComponent`
- 역할 :  
  1. 미세한 Geomtry Jitter 방지 : `LineTraceSingleByChannel`이 빗나가지 않도록
  2. 상태 Flickering 차단 : 매 프레임마다 깜빡이지 않도록
  3. 유예 : `GroundContactCoyoteDuration`이 0.1안이라면, 땅에 붙어있는것으로 간주
  4. `CurrentInteractionData`구조체(`FIZInteractionData`) 채우기

#### 2.2 `UpdateRenderThreadState()`
- Class : `UIZGridComponent_v05`
- 역할 :  
  1. ResolveInteractionData() -> 3장의 우선순위에 따라 `FIZInteractionData`를 만들어 반환
  2. 위 1번의 `ResolveInteractionData()` output등 RT파라미터로 복사.
     `Enable Temporal Tessellation Compensation`, `Tessellation Detail Decay Rate`, `Tessellation Detail Hold Duration`, `Min Tessellation Detail Weight`

- 주요 : `FPrimitiveSceneProxy*`를 `GT->RT`로 복사해서 Proxy에서 RenderThread로 활용할 수있도록함
#### 2.3 `SetOptimizeParams_RenderThread()`
- Class : `UIZGridComponent_v09`
- 역할 :  
  1. 2.2와 동일하게 `UIZGridProxyComponent_v09`로 `GT->RT`복사
  2. 추가 optimize - 아래 Variable
- Variable : 
  `FootprintCulldistance`: RTSize Culling  
  `RevisitCacheCellSize` : Cell cache Size

#### 2.4 `UpdateInteractionState_RenderThread()` ★★
- Call Method : `RefineAndBuildGPUInput_RenderThread`
1. Class : `FIZGridSceneProxy_v09_02`∈ `FIZGridSceneProxy_v05`

- 역할 
① 위치 스무딩, 예측
-  `FMath::VinterpTo(SmoothedInteractionLocation, InData.Location, DeltaTime, SmoothingSpeed);
- `FilterData.Velocity  =InData.Velocity *(PredictiveLookAheadSeconds/0.1f);`
② 재방문 Cell 판정,  Local Cell 양자화
 -  `ResolveCurrentCell_RT()`:
	-> `VisitedCells.Add(cell);`로 더함, return true하면 interaction allow됨
  - `MakeCellKey()` : `FloorToInt`로 양자화

  ③ 발자국 추가, 거리 컬링, 시간 감쇠 제거
  - `TimedDetailSample`: FIZTimedDetailSample 배열로 Interaction, Last TouchTime가진 구조체
    `FootPrintCells` : 
  - `TemporalCompensation == false`면 `TimedDetailSample`Reset
  ④ FootPrint CullDistance 밖 Tessllation Lower로 만듦 (`TimedDetailSample` list 제거)
  ⑤ `Compute Weight`로 `TimedDetailSample`점점 사라지게
  ⑥ Effective Params에 `CVar`를 반영, Gridsize 바뀌면 `LEB.Init`
  ⑦ `bShouldRefine?`으로 이동임계값 / 접촉변화, 설정변경 footprint 개수 변화
  `LEB.Refine()`
  1. Leaf 목록 -> 정점 배열 -> GPU 업로드 ->CS execute -> Draw


#### 2.5 `Refine()` ★★
- Class : `FIZLEB_v_08_1_1`
- 역할
① Reset : 삼각형 2개
- `BaseQueue.Add(0)`, `BaseQueue.Add(1)`root 삼각형으로 시작,
  LEB방식으로 빗변을 기준으로 반 쪼개는것 반복후 세분화
② 균일 BFS 선 분할 : minimumDepth(SplitConforming)
- 거리와 상관없이 삼각형 `MinimumDepth`까지 BFS 순서로 먼저 조갠다
③ Queue 초기화 - 각 리프마다 평가
- `ComputeDistanceToTriangle2D` : 플레이어와 삼각형 사이의 2D 거리를 구함
- `ComputeContinuousTargetDepth`:거리 가까울수록 높은 target depth 반환
- `ComputeWeight` : 2-4-5와 같은 감쇠
④ HeapPop -> 분할 -> 재평가
- Pop : 3번에서 넣은 초기 후보로 target depth에서 가장 많이 coarse, player와 가장 가까운 삼각형이 먼저
- 검증 : `splitConforming`이 이웃을 연쇄적으로 쪼개니, queue의 삼각형 이미 쪼개져서 죽었을수있어서 확인후 죽었으면 버리기 : `lazy deletion` -queue 중간항목 지우지않고, 꺼낼때 무효 후 버림
- 재평가 : pop한 값의 `priority`는 Push 시점의 스냅샷, `evaluateTriangleMetrics`를 돌려 최신 `desiredDepth`로 판단
- 분할 조건: `T.Depth<DesiredDepth`일 때만 쪼개기
- `SplitConforming()` : 삼각형 쪼개면 부모 죽고 자식 두개 생긴다-> 이웃과 T-junction 생기지 않도록 이웃까지 연쇄로 쪼개고, `NewLeaves`에 담기고 `NeghborIndex==INDEX_NONE`이던지, `NeighborHypKey==HypKey`확인하고 그게 아니면, 재귀로 또 쪼개짐
- Priority_queue인 HeapPush를 활용해서 Functor로 childpriority활용  `HeapPush(FIZRefineCandidate(NewLeafIdx, ChildPriority));`로 


```cpp
FIZLEB.h
	static FORCEINLINE uint32 PackPoint(const FIntPoint& P)
	{	return ((static_cast<uint32>(P.X) & 0xFFFF) << 16) | (static_cast<uint32>(P.Y) & 0xFFFF);	}

	static FORCEINLINE uint64 MakeEdgeKey(const FIntPoint& P1, const FIntPoint& P2)	{
		uint32 A = PackPoint(P1);
		uint32 B = PackPoint(P2);
		if (A > B){	Swap(A, B);	}
		return (static_cast<uint64>(A) << 32) | static_cast<uint64>(B);
	}
}
```
- Bit연산, Shift로 Edge를 64Bit 고유키로 변환하는 패턴.
- 이런식으로 TMap으로 Edge 정보를 O(1)시간에 빠르게 관리 가능

---

### 3. Shader (.usf)활용 흐름

1. `AIZTessellationShader`: Shader Directory 가상 경로 매핑
2. IZGridSceneProxy_09_02:
   -> 2.1. Shader Class Declare + Parameter Struct 매크로 등록
   -> 2.2 `IMPLEMENT_GLOBAL_SHADER` : .usf를 Compute Shader로 등록
3. `IZGridSceneProxy::GenerateMesh_RenderThread()`(RT):  `AddPass()`로 RDGPass ComputeShader 실행 요청
4. `IZGridSceneProxy::GetDynamicMeshElemeht()` - 1. FDGBuilder. 업로드, 구조체 업로드, Dispatch, buffer 추출

#### 3.1 Shader Directory **가상 경로 매핑**:
``` cpp
AIZTessellationShader::StartupModule(){
const FString PluginShaderDir = FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("AIZTeslleationShader"))-> GetBaseDir(), TEXT("Shaders"));
AddShaderSourceDirectoryMapping(TEXET("/Plugin/AIZTessellationShader"), PluginShaderDir);
```

#### 3.2 GlobalShader 선언:

```cpp
class FAIZGenerateGridMeshCS : public FGlobalShader
{
	DECLARE_GLOBAL_SHADER(FAIZGenerateGridMeshCS);
	SHADER_USEPARAMTER_STRUCT(FAIZGenerateGridmeshCS, FGlobalShader);
public:
	BEGIN_SHADER_PARAMETER(FParameters, )
	... // shader binding to .usf
	END_SHADER_PARAMETER_STRUCT()
	
	static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters) { return IsFeatureLevelSupported(Parameters.Platform, ERHIFearreLevel::SM5); }
	//2.1 
};
IMPLEMENT_GLOBAL_SHADER(FAIZGenerateGridMeshCS, "/Plugin/AIZTessellationShader/Private/AIZGenerateGridMesh.usf", "AIZGenerateGridMeshCS", SF_Compute);
```


#### 3.3 AddPass :

``` cpp
IZGridSceneProxy::GenerateMesh_RenderThread(){
FComputeShaderUtils::AddPass(GraphBuilder, RDG_EVENT_NAME("AIZ_LEBMeshGeneration"), ComputeShader, Parameter, FIntVector(FMath::DivideAndRoundup(VertexCount, 64u), 1, 1));
...
GraphBuilder.Execute();
}
```
- `"AIZ_LEBMeshGeneration"`으로 RDG_EVENT이름 설정
- GraphBuilder로 RDG ComputeShader 실행!


#### 3.4 .usf
```hlsl


```


| C++ `FParameters`  | `.usf` 선언                                    | 타입  | 역할                                       |
| ------------------ | -------------------------------------------- | --- | ---------------------------------------- |
| `InvGridSize`      | `float InvGridSize;`                         | 상수  | 사전 계산된 역격자 폭 (`1.0 / GridSize`), 나눗셈 제거용 |
| `VertexCount`      | `uint VertexCount;`                          | 상수  | 유효 정점 수                                  |
| `TerrainSize`      | `float TerrainSize;`                         | 상수  | 월드 메쉬 평면 크기                              |
| `LeafGridVertices` | `StructuredBuffer<float2> LeafGridVertices;` | SRV | CPU 0-할당 정점 격자 좌표                        |
| `RWPositions`      | `RWBuffer<float> RWPositions;`               | UAV | GPU 로컬 정점 좌표 출력                          |
| `RWIndices`        | `RWBuffer<uint> RWIndices;`                  | UAV | GPU 정점 인덱스 스트림 출력                        |

---




### 4. Code Review


#### 4.1 AIZTessellationShader.h/.cpp
- 모듈 Startup/Shutdown.

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
│       │   ├── FIZLEB_v05.h                          # v05 프록시 기반 타입
│       │   ├── FIZCBTSumTree.h
│       │   ├── IZTypes.h
│       │   ├── IZTemporalTessellationCompensation.h
│       │   ├── IZRefineMetric.h
│       │   ├── AIZComponent.h                        # 상호작용 위치·접촉 데이터
│       │   └── IZCVars.h
│       └── Private/
│           ├── IZGridComponent_v09_02.cpp
│           ├── IZGridSceneProxy_v09_02.cpp
│           ├── IZGridComponent_v05.cpp               # 상속 Tick 및 상호작용 수집
│           ├── IZGridSceneProxy_v05.cpp              # 기반 프록시 생성·기본 동작
│           ├── AIZComponent.cpp
│           ├── IZCVars.cpp
│           └── AIZTessellationShader.cpp             # 셰이더 경로 등록
└── Shaders/
    └── Private/
        └── AIZGenerateGridMesh_v09_02.usf
```

  

| 단계                           | 실행 위치                     | 하는 일                                                                             | CPU에 두는 이유 / GPU에 두는 이유                      |
| ---------------------------- | ------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------- |
| 입력·설정 수집                     | 게임 스레드 (CPU)              | 플레이어/액터 위치, 속도, 접촉 상태, 파라미터, Transform 복사 후 `ENQUEUE_RENDER_COMMAND`로 렌더 스레드에 전달 | 게임 로직과 연결된 데이터라 CPU가 자연스러움                   |
| 위치 스무딩·예측                    | 렌더 스레드 (CPU)              | 상호작용 위치 스무딩과 예측 입력 계산                                                            | 소량의 상태 기반 계산                                 |
| 재방문 판정·footprint 감쇠·컬링       | 렌더 스레드 (CPU)              | 셀 캐시 확인, 시간 감쇠, 거리 컬링                                                            | 이력을 유지하며 분기가 많은 로직                           |
| 갱신 임계값 판정                    | 렌더 스레드 (CPU)              | 이번 프레임에 토폴로지를 갱신할지 결정                                                            | 조건 판단 위주, 병렬화 이득 없음                          |
| LEB 분할·CBT 상태 관리             | 렌더 스레드 (CPU)              | `FIZLEB_v08_1_1`, `FIZCBTSumTree`로 토폴로지와 활성 삼각형 확정                               | 이웃·우선순위·이전 상태에 의존해 병렬화가 어려움                  |
| 격자 좌표·색상 배열 준비               | 렌더 스레드 (CPU)              | 활성 삼각형의 `LeafGridVertices` 구성 후 GPU 버퍼로 업로드                                      | GPU 입력을 만드는 단계                               |
| 정점 XYZ 위치 생성                 | GPU (Compute Shader)      | `AIZGenerateGridMesh_v09_02.usf`에서 정점별 로컬 위치 계산                                  | 정점끼리 독립적이고 연산이 동일해서 병렬 처리에 적합                |
| 삼각형 인덱스 생성                   | GPU (Compute Shader)      | 같은 셰이더에서 인덱스 순서 생성                                                               | 위와 동일 (스레드 1개 = 정점 1개, `numthreads(64,1,1)`) |
| UV·접선·색상 스트림, Vertex Factory | 렌더 스레드 (CPU)              | 보조 버텍스 스트림 구성, GPU 버퍼를 Vertex/Index Buffer로 연결                                   | 렌더링 리소스 구성 작업                                |
| Mesh Batch 제출                | 렌더 스레드 (CPU)              | `GetDynamicMeshElements`에서 제출                                                    | 렌더 명령 준비                                     |
| 최종 래스터화                      | GPU                       | 제출된 메시를 화면에 그림                                                                   | GPU 고유 역할                                    |
| 버퍼 재할당 판단                    | 렌더 스레드 (CPU)              | `MaxTriangles×3`으로 사전 할당, 용량 부족 시에만 재할당                                          |                                              |
| RDG 그래프 구성·실행                | 렌더 스레드 (CPU)가 구성, 실행은 GPU | 업로드 → CS 디스패치 → 버퍼 추출                                                            |                                              |

`FIZLEB_v08_1_1`이 `Triangle`배열, `Edge`맵으로 topology 확정, `FIZCBTSumTree`는 Split에 맞춰 합트리 만 동기 갱신(현재 렌더링 경로에서는 조회 X)



**요약**

|구분|담당 작업의 성격|
|---|---|
|CPU (GT + RT)|분기·상태·이력이 많은 판단 로직, 토폴로지 확정, 리소스 구성과 제출|
|GPU|같은 연산을 수천 개 정점에 반복하는 위치·인덱스 계산, 래스터화|


---

### 7. **수식** :
