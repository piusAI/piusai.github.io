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

### 7. **수식** 
### 7.1 격자, LEB, 리프 수

격자 한 변의 셀 수 $G = 2^{K}$, 지형 한 변 길이 $L$, 셀 크기 $s = L/G$. 정수 격자 좌표 $\mathbf g \in \{0,\dots,G\}^2$는 로컬 좌표로

$$\mathbf x(\mathbf g) = \left(\mathbf g - \tfrac{G}{2}\right)s = \left(\tfrac{\mathbf g}{G}-\tfrac12\right)L, \qquad \mathbf{uv}=\tfrac{\mathbf g}{G}$$

로 변환된다. (CPU 빌더와 Compute Shader가 같은 식을 사용)

**LEB 분할.** 삼각형 $T=(v_0,v_1,v_2)$ (빗변 $v_0v_2$)를 중점 $m=\tfrac{v_0+v_2}{2}$로 나누면 자식은

$$T_L=(v_1,\,m,\,v_0),\qquad T_R=(v_2,\,m,\,v_1)$$

NodeID $k$의 자식은 $2k,\,2k+1$이고, LEB 깊이는 $d(k)=\lfloor\log_2 k\rfloor-1$ (기저 삼각형 NodeID 2, 3 → $d=0$).

**깊이와 리프 수.** 균일하게 깊이 $d$까지 분할하면

$$N(d)=2^{d+1},\qquad A(d)=\frac{L^2}{2^{d+1}},\qquad \ell(d)=L\,2^{-d/2}\ \ (\text{직각변 길이})$$

최대 깊이 $d_{\max}=2K-1$에서 $N=2^{2K}=G^2$, 직각변 $\sqrt2\,s$. 비교 기준이 되는 **정적 균일 메시**는 같은 해상도의 $2G^2$ 삼각형(셀당 2개)이다. 정수 중점이 보장되는 한계가 $2K-1$이다.

**합 트리(CBT Sum Tree).**

$$c(k)=c(2k)+c(2k+1),\qquad c(\text{leaf})\in\{0,1\},\qquad N=c(1)$$

$r$번째 활성 리프 탐색 ($r=0,\dots,N-1$, 루트에서 시작):

$$\text{if } r<c(2k):\ k\leftarrow 2k;\quad \text{else } r\leftarrow r-c(2k),\ k\leftarrow 2k+1 \qquad (O(d))$$

### 7.2 깊이 범위와 예산

$$d_{\text{eff}}=\min(2K_{\max}-1,\ 2K-1),\qquad d_{\min}=\min(2\hat{l}-1,\ d_{\text{eff}})$$

$\hat l$은 $l_{\min}$에서 시작해 예산 조건을 만족할 때까지 줄인 값이다.

$$\hat l=\max\{\,l\le l_{\min}:\ 4^{l}\le N_{\max}\,\},\qquad N(d_{\min})=4^{\hat l}$$

(구현은 좌변에 계수 2를 곱해 2배 보수적, 4.5 M2.) 기본값 $l_{\min}=4$ → $d_{\min}=7$, 기저 리프 $N=256$.

예산 대비 압축률(정적 균일 메시 대비)

$$\rho=\frac{N_{\text{adaptive}}}{2G^2}\ \le\ \frac{N_{\max}}{2G^2}$$

기본값 $K=8$에서 $2G^2=131072$이고 $N_{\max}=8192$이면 $\rho\le 6.25\%$. ($d_{\max}$까지 균일 분할 시 $G^2=65536$ 리프이므로 LEB 기준으로는 12.5%.)

### 7.3 스무딩과 예측

프레임 $t$의 스무딩 위치 (UE `VInterpTo`, $k$ = SmoothingSpeed, $\Delta t\in[0,0.1]$):

$$\mathbf p_s^{t}=\mathbf p_s^{t-1}+\operatorname{clamp}(k\,\Delta t,0,1)\,\big(\mathbf p^{t}-\mathbf p_s^{t-1}\big)$$

예측 위치. 의도한 식은 $\tilde{\mathbf p}=\mathbf p_s+\lambda\mathbf v$ 이나, **현재 구현은 룩어헤드가 두 번 곱해져서**

$$\tilde{\mathbf p}_{\text{impl}}=\mathbf p_s+\frac{\lambda^2}{0.1}\,\mathbf v=\mathbf p_s+0.064\,\mathbf v\qquad(\lambda=0.08)$$

### 7.4 점-삼각형 거리

질의점 $\mathbf q$, 삼각형 $T=(a,b,c)$. 내부(경계 포함)면 0이고, 외부면 세 변까지 거리의 최솟값.

$$d(\mathbf q,T)=\begin{cases}0 & \mathbf q\in T\\ \min\limits_{(u,w)\in\{(a,b),(b,c),(c,a)\}}\ \|\mathbf q-\pi_{uw}(\mathbf q)\|& \text{otherwise}\end{cases}$$ $$\pi_{uw}(\mathbf q)=u+\tau\,(w-u),\qquad \tau=\operatorname{clamp}\!\left(\frac{(\mathbf q-u)\cdot(w-u)}{\|w-u\|^2},0,1\right)$$

내부 판정은 세 변에 대한 2D 외적 부호가 모두 같음 (허용오차 $10^{-4}$).

### 7.5 연속 목표 깊이

$D_{lo}=\max(10,\text{MinDistance})$, $D_{hi}=\max(D_{lo}+10,\text{RefineDistance})$, 샤프니스 $\gamma$, 깊이 한계 $d_{\text{eff}}$.

$$\alpha(d)=1-\frac{d-D_{lo}}{D_{hi}-D_{lo}},\qquad \bar\alpha=\alpha^{\gamma},\qquad S(\bar\alpha)=\bar\alpha^2(3-2\bar\alpha)$$ $$d^{*}(d)=\begin{cases}0 & d\ge D_{hi}\\ d_{\text{eff}} & d\le D_{lo}\\ \operatorname{clamp}\big(S(\bar\alpha)\,d_{\text{eff}},\,0,\,d_{\text{eff}}\big) & \text{otherwise}\end{cases}$$

($\gamma=1$이면 순수 smoothstep.)

### 7.6 시간 가중치 (4.6)

$$w(t)=\begin{cases}1 & t\le h\\ e^{-k(t-h)} & t>h,\ e^{-k(t-h)}>w_{\min}\\ 0 & \text{otherwise}\end{cases},\qquad t_{exp}=h+\frac{\ln(1/w_{\min})}{k}$$

### 7.7 목표 깊이 결합 (플레이어 + footprint)

footprint 집합 $\mathcal S=\{(\mathbf c_i,\tau_i)\}$ (위치, 마지막 터치 시각). 현재 시각 $t$, 현재 상호작용 가중치

$$w_0=\mathbb 1[\text{접촉 중}\ \wedge\ \text{재방문 허용}],\qquad w_i=w(t-\tau_i)$$

삼각형 $T$의 깊이 부족량(deficit)과 목표 깊이:

$$\Delta(T)=\max\Big\{\,w_0\,\big[d^{*}(d(\tilde{\mathbf p},T))-d_{\min}\big]_+,\ \max_{i}\ w_i\,\big[d^{*}(d(\mathbf c_i,T))-d_{\min}\big]_+\Big\}$$ $$d_{\text{des}}(T)=\operatorname{clamp}\big(d_{\min}+\operatorname{round}(\Delta(T)),\ d_{\min},\ d_{\text{eff}}\big)$$

($[x]_+=\max(x,0)$.) 분할 조건: $d(T)<d_{\text{des}}(T)$.

### 7.8 우선순위 큐

$$P(T)=\big(d_{\text{des}}(T)-d(T)\big)\cdot 10^{5}-d(\tilde{\mathbf p},T)$$

의도한 pop 순서는 $P$ 내림차순 (**deficit이 큰 것 먼저, 같으면 플레이어에 가까운 것 먼저**). $10^5$ 계수는 deficit이 거리보다 항상 우선하도록 하는 사전식(lexicographic) 정렬 역할이다. 구현상 pop 후 $d_{\text{des}}$를 재평가(lazy), 분할 후 자식을 같은 식으로 push. (4.5 H1: 힙 방향 확인 필요)

### 7.9 재방문 억제와 footprint

셀 크기 $\ell$ (RevisitCacheCellSize), 컴포넌트 로컬 좌표 $(x_l,y_l)$:

$$\mathbf z(\mathbf p)=\big(\lfloor x_l/\ell\rfloor,\ \lfloor y_l/\ell\rfloor\big)$$

방문 집합 $\mathcal V$ (FIFO, $|\mathcal V|\le M$), 직전 접촉 셀 $\mathbf z_{last}$:

$$a=\begin{cases}1 & \mathbf z=\mathbf z_{last}\\ \mathbb 1[\mathbf z\notin\mathcal V] & \text{otherwise}\end{cases}\quad(\text{접촉 중일 때}),\qquad a=0\ \ (\text{비접촉})$$

$a=1$이고 $\mathbf z\notin\mathcal V$이면 $\mathcal V\leftarrow\mathcal V\cup\{\mathbf z\}$. 이 $a$가 7.7의 "재방문 허용"이다.

footprint 추가 간격 $\sigma=\max(25,\,0.5\,r_c)$ ($r_c$ = ContactRadius). 새 셀이고 마지막 샘플과의 2D 거리 $\ge\sigma$이면 추가(FIFO, 최대 $F=64$), 아니면 마지막 샘플을 갱신. **거리 컬링**:

$$\text{remove}\ (\mathbf c_i,\tau_i)\ \text{if}\ \big\|\mathbf p_{s,xy}-\mathbf c_{i,xy}\big\|>R_{cull}$$

### 7.10 갱신 게이팅

$$\text{refine}\iff \neg\text{init}\ \vee\ \|\Delta\mathbf p_{xy}\|\ge\theta\ \vee\ \Delta\text{contact}\ \vee\ \Delta\text{settings}\ \vee\ \Delta|\mathcal S|\ \vee\ \big(|\mathcal S|>0\wedge t-t_{last}\ge 0.25\big)$$

$\theta$ = SpatialUpdateThreshold. 서 있을 때 호출 빈도의 상한은 감쇠 틱 때문에 $\le 4\,\text{Hz}$.

### 7.11 비용 모델 (논문용, 근사)

- Refine : 평가 횟수를 $E$, 평균 샘플 수를 $\bar S$라 하면 $O\big(E\,(1+\bar S)+N\log N\big)$. $E\approx N_0+2N_{split}$ 규모. (M1 개선 시 샘플 변환 비용 제거)
- 메시 생성 : CS 그룹 수 $\lceil 3N/64\rceil$, 정점 $3N$ 개.
- 메모리(사전 할당) : 위치 $3N_{\max}\times12\,\text{B}$, 인덱스 $3N_{\max}\times4\,\text{B}$. $N_{\max}=8192$이면 위치 약 295KB, 인덱스 약 98KB.
- 합 트리 : $4\cdot2^{\,2K\,(\text{or}\,2K+1)}$ B (4.7 S3).