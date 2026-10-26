---
layout: post
published: false
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

#### 1-3. CPU/GPU 역할 분담

| 단계                           | 실행 위치                     | 하는 일                                                                             | CPU에 두는 이유 / GPU에 두는 이유                      |
| ---------------------------- | ------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------- |
| 입력·설정 수집                     | GT (CPU)                  | 플레이어/액터 위치, 속도, 접촉 상태, 파라미터, Transform 복사 후 `ENQUEUE_RENDER_COMMAND`로 렌더 스레드에 전달 | 게임 로직과 연결된 데이터라 CPU가 자연스러움                   |
| 위치 스무딩·예측                    | RT (CPU)                  | 상호작용 위치 스무딩과 예측 입력 계산                                                            | 소량의 상태 기반 계산                                 |
| 재방문 판정·footprint 감쇠·컬링       | RT (CPU)                  | 셀 캐시 확인, 시간 감쇠, 거리 컬링                                                            | 이력을 유지하며 분기가 많은 로직                           |
| 갱신 임계값 판정                    | RT (CPU)                  | 이번 프레임에 토폴로지를 갱신할지 결정                                                            | 조건 판단 위주, 병렬화 이득 없음                          |
| LEB 분할·CBT 상태 관리             | RT (CPU)                  | `FIZLEB_v08_1_1`, `FIZCBTSumTree`로 토폴로지와 활성 삼각형 확정                               | 이웃·우선순위·이전 상태에 의존해 병렬화가 어려움                  |
| 격자 좌표·색상 배열 준비               | RT (CPU)                  | 활성 삼각형의 `LeafGridVertices` 구성 후 GPU 버퍼로 업로드                                      | GPU 입력을 만드는 단계                               |
| 정점 XYZ 위치 생성                 | GPU (CS)                  | `AIZGenerateGridMesh_v09_02.usf`에서 정점별 로컬 위치 계산                                  | 정점끼리 독립적이고 연산이 동일해서 병렬 처리에 적합                |
| 삼각형 인덱스 생성                   | GPU (CS)                  | 같은 셰이더에서 인덱스 순서 생성                                                               | 위와 동일 (스레드 1개 = 정점 1개, `numthreads(64,1,1)`) |
| UV·접선·색상 스트림, Vertex Factory | 렌더 스레드 (CPU)              | 보조 버텍스 스트림 구성, GPU 버퍼를 Vertex/Index Buffer로 연결                                   | 렌더링 리소스 구성 작업                                |
| Mesh Batch 제출                | 렌더 스레드 (CPU)              | `GetDynamicMeshElements`에서 제출                                                    | 렌더 명령 준비                                     |
| 최종 래스터화                      | GPU                       | 제출된 메시를 화면에 그림                                                                   | GPU 고유 역할                                    |
| 버퍼 재할당 판단                    | 렌더 스레드 (CPU)              | `MaxTriangles×3`으로 사전 할당, 용량 부족 시에만 재할당                                          |                                              |
| RDG 그래프 구성·실행                | 렌더 스레드 (CPU)가 구성, 실행은 GPU | 업로드 → CS 디스패치 → 버퍼 추출                                                            |                                              |
- CPU(GT + RT) : 분기 상태 이력이 많은 판단, 토폴로지 확정, 리소스 구성과 제출
- GPU와 같은 연산을 수천 개 정점에 반복하는 위치 인덱스 계산, 래스터화
- `FIZLEB_V08_1_1`이 `Triangles`배열과 `Edge` map으로 topology 확정하고,`FIZCBTSumTree`는 Split에 맞춰 합 트리만 동기 갱신한다(현재 렌더링 경로에서는 조회 X)



### 2. Main Pipeline

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/AIZ/V09Algorithm.png" alt="V9Algorithm" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>V9Algorithm</strong> </td>  </tr> </table>

#### 2.1 `UpdateInteractionTelemetry()`
- Class : `UAIZComponent`(Grid보다 먼저 실행되어 같은 프레임 값 전달)
- 역할 :  
  1. 위치 / 속도 수집 : SkeletalMesh가 물리 시뮬레이션 중이면 무게중심, 물리 선속도를, 아니면 Actor 위치/속도를 사용. 속도가 0이라면 `(current-last )/dt`로 추정
  2. 접지 판정 : 아래방향 `LineTraceSingleByChannel`. Simulation Mesh는 trace 거리를 `BoxExtent.Z+25`까지 확장
  3. **Coyote 유예** : 마지막 접지후 `GroundContactCoyoteDuration`(기본 0.25)이내면 접촉 유지, 지형 요철로 trace가 순간 빗나가도 `bIsContacting`이 깜빡이지 않는다.
  4. `CurrentInteractionData`구조체(`FIZInteractionData`) 채우기 : Location, Velocity, bIsContacting, ContactRadius, Timestamp

#### 2.2 `UpdateRenderThreadState()`
- Class : `UIZGridComponent_v05`
- 역할 :  
  1. ResolveInteractionData() -> [3장](https://piusai.github.io/graphics/2026/10/13/AIZ-Algorithm-v3.html)의 우선순위에 따라 `FIZInteractionData`를 만들어 반환
  2. 위 1번의 `ResolveInteractionData()` output등 RT로 전달
     `Enable Temporal Tessellation Compensation`, `Tessellation Detail Decay Rate`, `Tessellation Detail Hold Duration`, `Min Tessellation Detail Weight`
  3. RT 작업이 끝나면 `AsyncTask(GameThread)`로 `CachedTriangleCount` 등 통계를 GT에 되돌린다.

- 주요 : `FPrimitiveSceneProxy*`를 `GT->RT`로 복사해서 RT는 GT객체를 직접 읽지 않는다(thread safe)
#### 2.3 `SetOptimizeParams_RenderThread()`
- call : `UIZGridComponent_v09_02::TickComponent`
- 구현 : `FIZGridSceneProxy_v09_02`(Proxy)
- 역할 :  2.2와 동일하게 `GT->RT`복사. v09_02 전용 최적화 값만 전달

- Variable : 
  - `FootprintCulldistance`: RTSize Culling 
  - `bEnableFootprintCullDistance`, `bEnableRevisitSuppression`
  - `RevisitCacheCellSize` : Cell cache Size
  - `MaxCachedVisitedCells` : 방문 셀 FIFO용량(128~65536)

#### 2.4 `UpdateInteractionState_RenderThread()` ★★
- Class : `FIZGridSceneProxy_v09_02`∈ `FIZGridSceneProxy_v05`
- Call Method : `RefineAndBuildGPUInput_RenderThread`

① 위치 스무딩, 예측
-  `SmoothedInteractionLocation = VinterpTo(SmoothedInteractionLocation, InData.Location, dt, SmoothingSpeed)` (dt는 0~0.1 clamp)
- `FilterData.Velocity  =InData.Velocity *(PredictiveLookAheadSeconds/0.1f);`
② 재방문 Cell 판정,  Local Cell 양자화
 -  `MakeCellKey()` : 컴포넌트 로컬 좌표로 변환 후 `FloorToInt(Local/CellSize)
 - `ResolveCurrentCell_RT()`:
	 - 접촉중 아니라면 :false
	 - 직전 접촉 셀과 같으면 : true
	 - 다른 셀ㄹ인데 `visitedcells`에 없으면 : `visitedCells.Add`, `InsersionOrder.Add`용량 초과시 가장 오래된 셀부터 제거 : true
	 - 다른 셀인데 이미 있으면 : true(재방문 억제 : live 가중치 0)
	
③ 발자국 추가, 거리 컬링, 시간 감쇠 제거
  - `FIZTimedDetailSample` = {Interaction, Last TouchTime}
  - `TArray<FIZTimedDetailSample>TimedDetailSample`
  - `TemporalCompensation == false`면 `TimedDetailSample`Reset 하고 현재 위치 1개만 유지
  - true면, 접촉 중이고, `bAllowInteraction`이고, 해당 셀이 `FootPrintCells` 에 없을떄에 한해 sample 추가

④ FootPrint CullDistance 밖 즉시 제거 : smoothing된 플레이어 위치에서 2D 거리가 `FootprintCulldistance`넘는 샘플 제거 -> 그 영역 기저 tessellation으로 복귀
⑤ `Compute Weight`로 시간 감쇠 : 가중치 0이된 샘플 제거, 남은 sample중 최대 가중치를 `DetailCompensationWeight`로 기록
⑥ Effective Params 확정: `CVar`를 반영, `Gridsize` 바뀌면 `LEB.Init`
⑦ `bShouldRefine?`
```
!bHasEverRefined
|| MoveDist >= SpatialUpdateThresholld
|| bMeshSettingsChanged
|| bFootprintCountChanged
|| 샘플 존재 && 마지막 refine 이후 0.25
```
아니면 이전 GPU버퍼 그대로 쓰고 ㄱeturn

⑧ 통과한다면 `LEB.Refine()`Leaf 목록 -> 정점 배열 -> GPU 업로드 ->CS execute -> Draw


#### 2.5 `Refine()` ★★
- Class : `FIZLEB_v_08_1_1`

① Reset : 삼각형 2개( `Edges`, `SumTree`를 초기화)
- LEB방식으로 빗변을 기준으로 반 쪼개는 연산 반복해 세분화

② 균일 BFS 선 분할 : `MinimumDepth`까지 `SplitConforming`
- 거리와 상관없이 모든 삼각형 `MinimumDepth`까지 BFS 순서로 먼저 쪼갠다  
  (`BaseQueue.Add(0)`, `BaseQueue.Add(1)` 두 root 삼각형에서 시작)
- 효과 : 거대한 Coarse 삼각형을 가로지르는 가늘고 긴 needle분할 없애고, base Grid를 대칭으로 만듦
- 예산(`MaxTriangles`)이 부족하면 `MinSubdivLevel`을 자동으로 낮춘다 (`BudgetSafeMinLevel`)

③ Queue 초기화 - 활성 리프마다 `EvaluateTriangleMetrics`
- `ComputeDistanceToTriangle2D` : 플레이어와 삼각형 사이의 2D 거리를 구함
- `ComputeContinuousTargetDepth`:거리 가까울수록 높은 target depth 반환
- `ComputeWeight` : footprint 샘플마다 같은 방식으로 거리 구하고 시간 가중치 곱함
- 최종 `DesiredDepth = clamp(MinDepth + roundToInt(max(가중 deficit)), MinDepth, MaxDepth)
- `DesiredDepth>Depth`인 리프만 `prirority = deficit* 1e5 -dist`로 heapPush

④ HeapPop -> 분할 -> 재평가
- Pop : 목표 깊이에 가장 모자라고 (deficit 큼) 플레이어와 가장 가까운 삼각형이 먼저 나오도록 설계
- 검증(`lazy Delation`) : `splitConforming`이 이웃을 연쇄적으로 쪼개니, queue에 있던 삼각형이 이미 쪼개져서 죽었을수있어서, 큐 중간 항목을 지우지않고, 꺼낼때 `bAlive`를 확인해 죽었으면 버리기
- 재평가 : Push지점의 `priority`는 스냅샷으로, pop한 뒤 `EvaluateTriangleMetrics`를 다시 돌려 최신 최신 `desiredDepth`로 판단(footprint가 바뀌거나 이웃 분할로 상황 변했을 수 있다)
- 분할 조건: `T.Depth<DesiredDepth`일 때만 쪼개기
- `SplitConforming()` 
	- 삼각형 쪼개면 부모 죽고 자식 두개 생긴다.
	- T-junction 생기지 않도록 빗변 공유하는 이웃까지 연쇄로 쪼갠다.
		- `NewLeaves`에 담기고 `NeghborIndex==INDEX_NONE` : 단독 분할,
		- `NeighborHypKey==HypKey` : 둘다 분할
		- 그게 아니면, 재귀로 분할 후 다시 판정
- 새 리프는 `newLeaves`에 담기고, 자식마다 재 평가후, deficit있으면`HeapPush(FIZRefineCandidate(NewLeafIdx, ChildPriority));`
- `LeafCount >= Maxtriangles`가 되면 중단


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
- Bit 연산과 Shift로 방향 없는 Edge를 64Bit 고유키로 변환하는 패턴.
- `TMap<uint64, FEdgeUsers>`로 이 Edge를 사용하는 삼각형 2개를 O(1)에 찾는다


#### 2.6 `GenerateMesh_RenderThread()` ★
- class : IZGridSceneProxy_v09_02

① Scratch 배열에 Grid 좌표 색상 수정

② 용량 판단 `MatrixTriangle*3`

③ FDGBuilder 업로드, 구조체 업로드, Dispatch, Buffer 추출

④ UV, 색상 CPU 루프, StaticMeshBuffer/ColorBuffer/ Vertex 재 초기화


---

### 3. Shader (.usf)활용 흐름

1. `AIZTessellationShader`: Shader Directory 가상 경로 매핑
2. IZGridSceneProxy_09_02:
   -> 2.1. Shader Class Declare + Parameter Struct 매크로 등록
   -> 2.2 `IMPLEMENT_GLOBAL_SHADER` : .usf를 Compute Shader로 등록
3. `IZGridSceneProxy::GenerateMesh_RenderThread()`(RT): `FRDGBuilder`로 업로드 -> `AddPass()`-> 버퍼 추출 ->`Execute`
4. 같은 함수 후반부 : pooling buffer를 `FVertexBuffer` / `FIndexBuffer`로 alias, UV/접선/색상 스트림과 `FLocalVertexFactory`구성
5. `IZGridSceneProxy::GetDynamicMeshElements()` :Batch 제출만 수행(RDG/Update 없다)

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



**요약**

|구분|담당 작업의 성격|
|---|---|
|CPU (GT + RT)|분기·상태·이력이 많은 판단 로직, 토폴로지 확정, 리소스 구성과 제출|
|GPU|같은 연산을 수천 개 정점에 반복하는 위치·인덱스 계산, 래스터화|

---

# AIZ 논문 수식 · 한 줄 의미 · 기호 · 코드 위치 (최종)



|약칭|파일|
|---|---|
|**Comp**|`AIZComponent.cpp` (`AIZComponent.h`)|
|**C09**|`IZGridComponent_v09_02.cpp`|
|**C05h / C05**|`IZGridComponent_v05.h` / `IZGridComponent_v05.cpp`|
|**P09**|`IZGridSceneProxy_v09_02.cpp` (`.h`)|
|**L08**|`FIZLEB_v08_1_1.h` ← v09_02가 쓰는 LEB|
|**TC**|`IZTemporalTessellationCompensation.h`|
|**USF**|`AIZGenerateGridMesh_v09_02.usf`|

---

## 0. 전체 흐름 한눈에

```
Game Thread                         Render Thread (CPU)                          RDG / GPU
상호작용 수집                       발자국 관리 → 목표 깊이 → 큐 분할 → 게이팅     정점·인덱스 버퍼 생성
(contact, q)                        (smoothing, cell, decay, E_K, D_want, P_K)    (컴퓨트 셰이더)
```

|단계|수식|한 줄|
|---|---|---|
|① 입력|contact, smoothing|캐릭터 위치·접촉을 안정적으로 얻는다|
|② 발자국|cell, $\sigma$|중복 없이 지나온 경로를 기록한다|
|③ 요구량|alpha, target, decay, weight, deficit, integrated, target_depth|삼각형마다 "얼마나 더 잘게 쪼개야 하는가"를 계산한다|
|④ 분할|base_level, priority, Algorithm 1|예산 안에서 급한 삼각형부터 쪼갠다|
|⑤ 갱신|update_gate|변화가 없으면 재계산을 건너뛴다|

---

## 1. 최종 파라미터

|기호|에디터 항목|값|기호|에디터 항목|값|
|---|---|---|---|---|---|
|$L_{\max}$|Max Subdiv Level|10|$\lambda$|Decay Rate|0.1 s⁻¹|
|$L_{\min}$|Min Subdiv Level|4|$T_{\mathrm{hold}}$|Hold Duration|6 s|
|$d_{\max}$|Refine Distance|90 cm|$w_{\min}$|Min Detail Weight|0.018187|
|$d_{\min}$|Min Distance|50 cm|$\delta_{\mathrm{move}}$|Spatial Update Threshold|50 cm|
|$B$|Max Triangles|26,000|$R_{\mathrm{cull}}$|Footprint Cull Distance|1800 cm|
|$k_{\mathrm{sm}}$|Smoothing Speed|50 s⁻¹|$h$|Revisit Cache Cell Size|175 cm|
|$\gamma$|Radial Falloff Sharpness|1.0|$M$|Max Cached Visited Cells|8192|
|$N_{\mathcal F}$|Max Footprint Samples|64|$T_{\mathrm{coy}}$|Coyote Duration|0.25 s ⚠️|
|$\kappa$|(코드 상수)|$10^5$|$r_c$|Contact Radius|100 cm ⚠️|

⚠️ $T_{\mathrm{coy}}$, $r_c$는 캐릭터 AIZ Component 값입니다. 기본값을 가정했으니 확인하세요.

파생값: $L_{\mathrm{safe}}=4$, $D_{\mathrm{base}}=7$, $D_{\lim}=19$, $\sigma=50$ cm, $T_{\mathrm{life}}\approx46.1$ s, 렌더링 복원 ≈ 37.8 s.

---

## 2. 4.1 상호작용 입력 · 발자국 관리

### 2-1. 접촉 판정 `eq:contact`

$$b_{\mathrm{contact}}(t)=\begin{cases}1,&\text{CastGroundHit or } t-t_{\mathrm{hit}}\le T_{\mathrm{coy}}\0,&\text{otherwise}\end{cases}$$

**한 줄 의미**: 아래로 쏜 line trace가 지면에 닿았거나, 마지막으로 닿은 뒤 유예 시간 이내면 "접촉 중"으로 본다.

|기호|의미|코드|위치|
|---|---|---|---|
|$b_{\mathrm{contact}}$|접촉 여부 (1/0)|`CurrentInteractionData.bIsContacting`|Comp:146-153|
|CastGroundHit|이번 프레임 line trace 적중|`LineTraceSingleByChannel` → `bDirectContact`|Comp:129-141|
|$t_{\mathrm{hit}}$|마지막 적중 시각|`LastContactTimestamp`|Comp:148|
|$T_{\mathrm{coy}}$|유예 시간|`GroundContactCoyoteDuration`|AIZComponent.h:51|

보조: 캐릭터 위치(물리 시뮬 중이면 무게중심)는 `GetAIZSourceLocation`(Comp:11-), 속도가 0에 가까우면 위치 차분 `(Cur-Prev)/DeltaTime`(Comp:123-125)으로 추정합니다.

### 2-2. 위치 평활화 `eq:smoothing`

$$\mathbf q_{\mathrm{sm}}\leftarrow\mathbf q_{\mathrm{sm}}+\operatorname{clamp}(\Delta t,k_{\mathrm{sm}},0,1),(\mathbf q-\mathbf q_{\mathrm{sm}})$$

**한 줄 의미**: 평활 위치를 실제 위치 쪽으로 프레임마다 일정 비율만큼 끌어당겨 떨림을 줄인다.

|기호|의미|코드|위치|
|---|---|---|---|
|$\mathbf q$|측정 위치|`InData.Location`|P09:247|
|$\mathbf q_{\mathrm{sm}}$|평활 위치|`SmoothedInteractionLocation` → `FilteredData.Location`|P09:249|
|$\Delta t$|프레임 시간 (0~0.1 s clamp)|`DeltaTime`|P09:235-237|
|$k_{\mathrm{sm}}$|평활 강도 (1~50 clamp)|`SmoothingSpeed`|P09:108|
|식 전체||`FMath::VInterpTo(...)`|P09:247|
|첫 프레임|$\mathbf q_{\mathrm{sm}}\leftarrow\mathbf q$|`Smoothed = InData.Location`|P09:240-244|

### 2-3. 예측 위치 $\mathbf q_p$ (식 번호 없음, 본문 서술)

$$\mathbf q_p=\mathbf q_{\mathrm{sm}}+\mathbf v\cdot\delta_{\mathrm{la}}$$

**한 줄 의미**: 갱신 지연을 보상하려고 현재 접촉의 요구량 계산에만 속도 방향으로 조금 앞선 위치를 쓴다.

|항목|코드|위치|
|---|---|---|
|1차 스케일 $\mathbf v\times(\text{LookAhead}/0.1)$|`FilteredData.Velocity = Velocity * (PredictiveLookAheadSeconds / 0.1f)`|P09:256|
|2차 $\mathbf q_{\mathrm{sm}}+\mathbf v'\times\text{LookAhead}$|`PredictedWorldSource = Location + Velocity * PredictiveLookAhead`|L08:289|
|발자국은 예측 미사용|`Sample.Interaction.Velocity = ZeroVector`|P09:272, 291, 300|

> 코드에서 look-ahead가 두 번 곱해져 실효값은 $0.21838\times0.021838\approx0.0048$ s입니다. 논문은 수치를 쓰지 않아 서술은 맞습니다.

### 2-4. 셀 양자화 `eq:cell`

$$c(\mathbf p)=\left(\lfloor x/h\rfloor,\ \lfloor y/h\rfloor\right)$$

**한 줄 의미**: 위치를 크기 $h$ 격자의 셀 번호로 바꿔, 같은 자리에 발자국이 중복 등록되는 것을 막는 키로 쓴다.

|기호|의미|코드|위치|
|---|---|---|---|
|$c(\cdot)$|셀 키|`MakeCellKey` (컴포넌트 로컬 좌표)|P09:147-154|
|$h$|셀 크기|`RevisitCacheCellSize` (하한 25)|P09:118, 150|

### 2-5. 현재 접촉 허용 $b_{\mathrm{allow}}$ (본문 서술)

**한 줄 의미**: 현재 접촉이 아직 방문하지 않은 셀이거나 직전과 같은 셀일 때만 허용하고, 오래전에 방문한 셀이면 막는다.

|조건|코드|위치|
|---|---|---|
|접촉 아니면 0|`bAllowCurrentInteraction = false`|P09:164-168|
|직전과 같은 셀이면 1|`LastContactCell == Cell`|P09:171-177|
|처음 방문 셀이면 1|`!VisitedCells.Contains(Cell)`|P09:180|
|방문 셀 캐시 최대 $M$개, 오래된 것부터 제거|`MaxCachedVisitedCells`, `InsertionOrder`|P09:185-191|

두 셀 집합의 차이:

|집합|용도|제거 시점|
|---|---|---|
|`FootprintCells`|발자국 중복 등록 방지|시간 만료(P09:336), 거리 컬링(P09:322)|
|`VisitedCells`|$b_{\mathrm{allow}}$ 판정|캐시 용량 초과(P09:188), 설정 변경 시 초기화(P09:357, 368)|

### 2-6. 발자국 간격 $\sigma$ (본문 서술)

$$\sigma=\max(25,\mathrm{cm},\ 0.5,r_c)$$

**한 줄 의미**: 새 발자국을 추가하는 최소 간격이며, 접촉 영향 반경의 절반(최소 25 cm)이다.

|기호|의미|코드|위치|
|---|---|---|---|
|$\sigma$|최소 등록 간격|`DetailSampleSpacing = Max(25, ContactRadius*0.5)`|C05:123|
|$r_c$|접촉 영향 반경|`ContactRadius`|AIZComponent.h:39|

|동작|코드|위치|
|---|---|---|
|새 셀에서 거리 ≥ $\sigma$ → 새 발자국 추가|`DistSquared2D(...) >= Square(Spacing)`|P09:283-295|
|아니면 최근 샘플 갱신|`TimedDetailSamples.Last()`|P09:296-302|
|발자국 위치 $\mathbf p_s$, 시각 $t_s$|`Sample.Interaction.Location`(평활 위치), `LastTouchTime`|P09:290-292|
|개수 상한 $N_{\mathcal F}$, 초과 시 가장 오래된 샘플 제거|`MaxFootprints`, `RemoveAt(0)`|C05h:57, C05:125, P09:285-288|
|용량 초과 제거 시 셀 키는 남음|(`FootprintCells.Remove` 없음)|P09:285-288|
|시간 만료 → 샘플·셀 키 제거|`FootprintCells.Remove(ExpiredCell)`|P09:333-338|
|$R_{\mathrm{cull}}$ 초과 → 샘플·셀 키 제거 (평활 위치 기준 2D)|`FootprintCells.Remove(CulledCell)`|P09:313-326|

---

## 3. 4.2 거리 기반 목표 깊이

### 3-1. 삼각형까지의 거리 $d(K,\mathbf q)$ (본문 정의)

**한 줄 의미**: 삼각형 $K$와 상호작용 위치 사이의 2D(투영) 최단 거리이며, 점이 삼각형 안에 있으면 0이다.

|항목|코드|위치|
|---|---|---|
|컴포넌트 로컬 2D 좌표로 변환|`InverseTransformPosition`|L08:289-291, 338-339|
|정확한 점-삼각형 거리|`ComputeDistanceToTriangle2D`|L08:112-151|
|`bUseExactTriangleDistance=true`일 때 사용||L08:301-304|

### 3-2. 정규화 거리 `eq:alpha`

$$\alpha(d)=\operatorname{clamp}!\left(\frac{d_{\max}-d}{d_{\max}-d_{\min}},,0,,1\right)$$

**한 줄 의미**: 거리 $d$를 $d_{\min}$ 이하면 1, $d_{\max}$ 이상이면 0이 되도록 [0,1]로 정규화한다.

|기호|의미|코드|위치|
|---|---|---|---|
|$d_{\min}$|최대 세분화 유지 반경|`MinDist = Max(10, MinDistance)`|L08:162|
|$d_{\max}$|영향 소멸 반경|`MaxDist = Max(MinDist+10, RefineDistance)`|L08:163|
|$\alpha$|정규화 값|`1 - (Dist-MinDist)/(MaxDist-MinDist)`|L08:174|
|clamp|경계 처리|early return (`Dist>=MaxDist→0`, `Dist<=MinDist→Limit`)|L08:165-172|

### 3-3. 목표 깊이 곡선 `eq:target`

$$T(d)=D_{\lim},S!\left(\alpha(d)^{\gamma}\right),\qquad S(x)=3x^2-2x^3,\qquad \gamma\in[0.1,5]$$

**한 줄 의미**: 정규화 거리에 $\gamma$ 곡선과 smoothstep을 적용해, 거리별 목표 분할 깊이(0~$D_{\lim}$)를 부드럽게 만든다.

|기호|의미|코드|위치|
|---|---|---|---|
|$T(d)$|거리 $d$의 목표 깊이|`ComputeContinuousTargetDepth`|L08:156-182|
|$\alpha^\gamma$|감쇠 형태 조절|`Pow(Alpha, RadialFalloffSharpness)`|L08:175-178|
|$S(x)$|smoothstep (끝점 기울기 0)|`Alpha*Alpha*(3 - 2*Alpha)`|L08:179|
|$\gamma$|곡선 지수 (0.1~5 clamp)|`RadialFalloffSharpness`|P09:107|
|$D_{\lim}$|최대 깊이|`DepthLimit` ← `EffectiveMaxDepth`|L08:181, 248|

성질: $d\le d_{\min}\Rightarrow T=D_{\lim}$, $d\ge d_{\max}\Rightarrow T=0$. 이 값은 최종 토폴로지가 아니라 "더 쪼갤 필요가 있는지" 평가용 목표치입니다.

---

## 4. 4.3 시간 감쇠 · 요구량 통합

### 4-1. 발자국 가중치 감쇠 `eq:decay`

$$\widetilde w_s=\exp!\Big[-\lambda,\max(0,\ \Delta t_s-T_{\mathrm{hold}})\Big],\qquad \Delta t_s=t_{\mathrm{now}}-t_s$$

**한 줄 의미**: 발자국 가중치는 $T_{\mathrm{hold}}$ 동안 1을 유지하다가 그 뒤부터 지수적으로 줄어든다.

|기호|의미|코드|위치|
|---|---|---|---|
|$\widetilde w_s$|감쇠 가중치|`ComputeWeight` 내부 `Weight`|TC:28|
|$\Delta t_s$|경과 시간|`CurrentTime - Sample.LastTouchTime`|L08:329, P09:331|
|$T_{\mathrm{hold}}$|유지 시간|`TessellationDetailHoldDuration`|TC:22-26|
|$\lambda$|감쇠율|`TessellationDetailDecayRate`|TC:28|

### 4-2. 제거 임계값 `eq:weight`

$$w_s=\begin{cases}\widetilde w_s,&\widetilde w_s>\bar w_{\min}\0,&\text{그 외}\end{cases},\qquad \bar w_{\min}=\operatorname{clamp}(w_{\min},0,0.25)$$

**한 줄 의미**: 가중치가 임계값 이하로 떨어지면 0으로 처리해 발자국의 영향을 끊는다.

|기호|의미|코드|위치|
|---|---|---|---|
|$w_s$|최종 발자국 가중치|`ComputeWeight` 반환값|TC:18-30|
|$\bar w_{\min}$|clamp된 임계값|`Clamp(MinWeight, 0, 0.25)`|TC:29|
|$w_s=0$ → 삭제|`RemoveAtSwap`||P09:333-338|

### 4-3. 현재 접촉 가중치 (본문 서술)

$$w_{\mathrm{cur}}=\mathbb I\big[b_{\mathrm{contact}}\land b_{\mathrm{allow}}\big]$$

**한 줄 의미**: 접촉 중이고 $b_{\mathrm{allow}}=1$일 때만 1이며, 현재 접촉에는 즉시 반응하고 과거 흔적에는 점진 복원을 적용한다.

|코드|위치|
|---|---|
|`CurrentInteractionWeight = (bIsContacting && bAllowInteraction) ? 1.0f : 0.0f`|P09:398|
|`Refine(..., DetailWeight = CurrentInteractionWeight, ...)` → `ClampedDetailWeight`|P09:399-400, L08:257|

### 4-4. 발자국 수명 `eq:lifetime`

$$T_{\mathrm{life}}=T_{\mathrm{hold}}+\frac{1}{\lambda}\ln\frac{1}{\bar w_{\min}}$$

**한 줄 의미**: 가중치가 제거 임계값에 도달해 발자국 자료가 사라지는 시점이다.

|항목|값|
|---|---|
|최종값|$6+10\ln(1/0.018187)\approx\mathbf{46.1}$ s|
|코드|별도 계산 없음. `ComputeWeight`가 0을 반환해 삭제되는 시점(P09:333)이 곧 $T_{\mathrm{life}}$|

반올림 때문에 세분화는 먼저 복원됩니다. 최대 깊이 영역은 $E_K=12w_s<0.5$, 즉 $6+10\ln24\approx37.8$ s에 베이스 깊이로 돌아옵니다.

### 4-5. 현재 접촉·발자국의 깊이 결핍량 `eq:current_deficit`, `eq:foot_deficit`

$$e_{\mathrm{cur},K}=w_{\mathrm{cur}}\max{0,\ T(d(K,\mathbf q_p))-D_{\mathrm{base}}}$$ $$e_{s,K}=w_s\max{0,\ T(d(K,\mathbf p_s))-D_{\mathrm{base}}}$$

**한 줄 의미**: 접촉 또는 발자국 $s$가 삼각형 $K$에 요구하는 목표 깊이 중, 베이스 깊이를 넘는 추가분에 가중치를 곱한 값이다.

|기호|의미|코드|위치|
|---|---|---|---|
|$D_{\mathrm{base}}$|베이스 깊이|`MinimumDepth`|L08:256|
|$e_{\mathrm{cur},K}$|현재 접촉 결핍량 ($\mathbf q_p$ 기준)|`Max(0, ContTarget - MinimumDepth) * ClampedDetailWeight`|L08:318-322|
|$e_{s,K}$|발자국 결핍량 ($\mathbf p_s$ 기준)|`Max(0, SampleTarget - MinimumDepth) * SampleWeight`|L08:324-349|
|함수||`EvaluateTriangleMetrics` 람다|L08:294-352|

### 4-6. 요구량 통합 `eq:integrated`

$$E_K=\max\Big(e_{\mathrm{cur},K},\ \max_{s\in\mathcal F}e_{s,K}\Big),\qquad \max\emptyset=0$$

**한 줄 의미**: 현재 접촉과 모든 발자국의 요구량 중 최댓값만 취해, 겹친 접촉이 깊이를 누적시키지 않게 한다.

|기호|코드|위치|
|---|---|---|
|$E_K$|`MaxScaledDeficit = Max(MaxScaledDeficit, ...)`|L08:317, 321, 347|
|$\mathcal F$|`TimedDetailSamples`|P09, L08:324-326|

### 4-7. 삼각형별 목표 깊이 `eq:target_depth`

$$D_K^{\mathrm{want}}=\operatorname{clamp}!\Big(D_{\mathrm{base}}+\operatorname{round}(E_K),\ D_{\mathrm{base}},\ D_{\lim}\Big)$$

**한 줄 의미**: 요구량을 정수로 반올림해 베이스 깊이에 더하고, 베이스와 최대 깊이 사이로 제한한 최종 목표 깊이다.

|코드|위치|
|---|---|
|`Clamp(MinimumDepth + RoundToInt(MaxScaledDeficit), MinimumDepth, EffectiveMaxDepth)`|L08:351|

성질: $E_K<0.5$이면 $D_K^{\mathrm{want}}=D_{\mathrm{base}}$이므로 추가 분할이 없고, 감쇠 중 목표 깊이는 연속이 아니라 단계적으로 내려갑니다.

### 4-8. AIZ 정의 `eq:aiz`

$$\mathcal A_{t_{\mathrm{now}}}={K\in\mathcal M\ \mid\ D_K^{\mathrm{want}}>D_{\mathrm{base}}}$$

**한 줄 의미**: 목표 깊이가 베이스를 넘는 삼각형들의 집합이 AIZ(적응형 상호작용 영역)이다.

|항목|코드|위치|
|---|---|---|
|별도 자료구조 없음|`Deficit = Desired - Depth > 0`일 때만 분할 후보|L08:374-375|

---

## 5. 5.1 예산 기반 세분화

### 5-1. 안전 베이스 레벨 `eq:base_level`

$$L_{\mathrm{safe}}=\max\Big{L\in\mathbb Z\ \Big|\ 1\le L\le\min(L_{\min},L_{\max}),\ \ 2\cdot4^{L}\le B\Big}$$

**한 줄 의미**: 균일 베이스 메시의 삼각형 수($2\cdot4^L$)가 예산 $B$ 안에 들어가는 가장 큰 레벨을 고른다.

|기호|코드|위치|
|---|---|---|
|$\min(L_{\min},L_{\max})$ 상한|`Clamp(MinSubdivLevel, 1, MaxSubdivLevel)`|L08:250|
|$2\cdot4^L\le B$|`while (2ll*(1ll<<(2*Level)) > Max(2,MaxTriangles)) --Level`|L08:251-255|
|조건 불만족 시 최소 레벨 1|`BudgetSafeMinLevel > 1`|L08:251|

### 5-2. 베이스·최대 깊이 (본문 정의)

$$D_{\lim}=2L_{\max}-1,\qquad D_{\mathrm{base}}=2L_{\mathrm{safe}}-1$$

**한 줄 의미**: 격자 레벨을 LEB 이진 트리 깊이로 바꾸는 관계식이다 (레벨 $L$ = 깊이 $2L-1$).

|기호|코드|위치|
|---|---|---|
|$D_{\lim}$|`Min(MaxSubdivLevel*2-1, MaxDepth())`|L08:248|
|$D_{\mathrm{base}}$|`Min(2*BudgetSafeMinLevel-1, EffectiveMaxDepth)`|L08:256|
|균일 베이스 메시 생성|BFS로 `MinimumDepth`까지 선분할|L08:262-286|

최종값: $D_{\lim}=19$, $D_{\mathrm{base}}=7$.

### 5-3. 우선순위 `eq:priority`

$$P_K=\kappa,(D_K^{\mathrm{want}}-D_K)-d(K,\mathbf q_p),\qquad \kappa=10^{5}$$

**한 줄 의미**: 깊이 부족량이 클수록, 부족량이 같으면 현재 접촉에 가까울수록 먼저 분할되게 하는 큐 우선순위다.

|기호|의미|코드|위치|
|---|---|---|---|
|$P_K$|우선순위|`Priority = Deficit * 100000.0f - Dist`|L08:378|
|$\kappa$|깊이 부족량 가중치|`100000.0f`|L08:378|
|$d(K,\mathbf q_p)$|현재 접촉까지 거리 (발자국만 있는 영역도 동일)|`Dist` (`LocalPlayerPos2D` 기준)|L08:303|
|max-priority queue||`HeapPush` / `HeapPop`, `FIZRefineCandidate_v08_1_1`|L08:25-35, 355, 379, 392|

### 5-4. Algorithm 1 대응 `alg:budget_refinement`

**한 줄 의미**: 균일 베이스 메시에서 시작해 깊이가 부족한 리프를 우선순위 순으로 conforming 분할하며, 리프 수가 예산에 닿으면 멈춘다.

|논문 단계|코드|위치|
|---|---|---|
|$L_{\mathrm{safe}}$, $D_{\mathrm{base}}$ 선택|`BudgetSafeMinLevel`, `MinimumDepth`|L08:250-256|
|UniformBaseMesh|BFS 선분할|L08:262-286|
|모든 리프의 $D^{\mathrm{want}}$ 평가, $>D_K$이면 큐에 삽입|큐 초기화 루프|L08:358-381|
|큐 비어있지 않고 LeafCount $<B$|`if (LeafCount >= MaxTriangles) break`|L08:386-389|
|최대 우선순위 pop|`HeapPop`|L08:392|
|not leaf 또는 $D_K\ge D_{\lim}$ → continue||L08:395-404|
|재평가 후 부족하면 분할|`EvaluateTriangleMetrics` → `if (T.Depth < DesiredDepth)`|L08:406-413|
|conforming LEB 분할|`SplitConforming`|L08:188-225|
|새 리프만 큐에 추가||L08:415-437|
|예산 초과 가능 (이웃 연쇄 분할)|한 번에 `SplitOne` 2회 이상|L08:216-223|

---

## 6. 5.2 메시 갱신 게이팅

### 6-1. 이동 조건 `eq:update_gate`

$$\left|\mathbf q_{\mathrm{sm},xy}-\mathbf q_{\mathrm{last},xy}\right|_2\ \ge\ \delta_{\mathrm{move}}$$

**한 줄 의미**: 마지막 재계산 위치에서 수평으로 $\delta_{\mathrm{move}}$ 이상 움직였을 때만 메시 재계산을 요청한다.

|기호|의미|코드|위치|
|---|---|---|---|
|$\mathbf q_{\mathrm{last}}$|마지막 재계산 위치|`LastRefinedLocation`|P09:374, 392|
|이동 조건||`MoveDistSq >= Square(SpatialUpdateThreshold)`|P09:374, 379|
|$\delta_{\mathrm{move}}$|이동 임계값|`SpatialUpdateThreshold`|P09:110|

### 6-2. 재계산을 요청하는 조건 전체 (본문 서술)

**한 줄 의미**: 최초 실행, 접촉·설정·발자국 개수 변경, 이동 임계 초과, 감쇠 반영 주기 경과 중 하나라도 해당할 때만 재계산한다.

|조건|코드|위치|
|---|---|---|
|최초 실행|`!bHasEverRefined`|P09:378|
|접촉 상태 변경|`bContactChanged`|P09:375|
|설정 변경|`bMeshSettingsChanged`|P09:363|
|발자국 개수 변경|`bFootprintCountChanged`|P09:345|
|감쇠 반영 주기 $T_{\mathrm{update}}=0.25$ s (고정값, 발자국 존재 시)|`Timestamp - LastRefineTimestamp >= 0.25f` (하드코딩)|P09:376|
|해당 없으면 생략|`if (!bShouldRefine) return;`|P09:385-390|
|재계산 시 베이스부터 재구성|`LEB.Refine` 내부 `Reset()`|P09:399, L08:245|
|리프 집합이 바뀔 때만 버퍼 재생성|`ScratchLeafNodeIDs != CachedLeafNodeIDs`|P09:416-434|
|발자국 등록·제거는 매 프레임|게이팅 앞에서 수행|P09:263-343|

---

## 7. 5.3 Unreal Engine 실행 경로 · RDG

**한 줄 의미**: CPU(Render Thread)가 토폴로지를 결정하고, RDG 컴퓨트 패스가 정점·인덱스 버퍼만 만든다. GPU에서 분할을 판단하지 않는다.

|논문|코드|위치|
|---|---|---|
|Game Thread: 접촉·위치·속도 수집|`UAIZComponent::UpdateInteractionTelemetry`|Comp:110-160|
|Render Thread로 전달|`ENQUEUE_RENDER_COMMAND`|C05:128-|
|Render Thread: 목표 깊이 + 토폴로지 (CPU)|`RefineAndBuildGPUInput_RenderThread` → `LEB.Refine`|P09:218, 399|
|RDG 구성|`FRDGBuilder GraphBuilder(RHICmdList)`|P09:480|
|버퍼 등록/생성|`RegisterExternalBuffer` / `CreateBuffer`|P09:486-503|
|리프 정점 업로드|`QueueBufferUpload`|P09:506-508|
|컴퓨트 패스|`FComputeShaderUtils::AddPass("AIZ_v09_02_LEBMeshGeneration")`|P09:520|
|버퍼 추출·실행|`QueueBufferExtraction`, `Execute()`|P09:522-524|
|셰이더 본문|격자 좌표 → `(UV-0.5)*TerrainSize`, z=1, 인덱스 winding 교정|USF:14-32|

---

## 8. 논문 표 ↔ 에디터 최종값 대조

|기호|논문 표|에디터|판정|
|---|---|---|---|
|$d_{\min},d_{\max},\gamma$|50, 90, 1.0|50, 90, 1.0|✓|
|$T_{\mathrm{hold}},\lambda,w_{\min}$|6 s, 0.1, 0.0182|6, 0.1, 0.018187|✓|
|$h,N_{\mathcal F}$|175, 64|175, 64|✓|
|$M$|8192|8192|✓|
|$k_{\mathrm{sm}}$|50|50|✓|
|$R_{\mathrm{cull}}$|1800|1800|✓|
|$\delta_{\mathrm{move}},T_{\mathrm{update}}$|50 cm, 0.25 s|50, 0.25(하드코딩)|✓|
|$B$, $L_{\min},L_{\max}$|26,000, 4, 10|26000, 4, 10|✓|
|$D_{\mathrm{base}},D_{\lim}$|7, 19|계산값 7, 19|✓|
|$\kappa$|$10^5$|100000|✓|
|$T_{\mathrm{coy}},r_c,\sigma$|0.25 s, 100 cm, 50 cm|캐릭터 BP 미확인|⚠️ 확인|

---

## 9. 마지막 확인 사항

1. **$T_{\mathrm{coy}}$, $r_c$**: 캐릭터 AIZ Component 값 확인. 다르면 표와 $\sigma$ 수정.
2. **look-ahead 이중 곱셈**: 논문 서술에는 문제 없음. 코드를 고치면 실험 재수행 필요.
3. **코드에서 쓰이지 않는 설정**: `Enable Hysteresis`, `Prioritize Current Interaction`(v08_1_1에서 미사용), `Enable Hierarchical Distance Culling`(확인한 소스에 사용처 없음). 논문에 언급하지 않으므로 문제 없음.
4. **기호 충돌 (선택)**: $M$(방문 셀 캐시)과 $\mathcal M$(메시), $T(d)$와 $T_{\mathrm{hold}}$, $\Delta t$와 $\Delta t_s$.