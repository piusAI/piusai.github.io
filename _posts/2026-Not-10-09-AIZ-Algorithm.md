---
layout: post
published: false
title: AIZ Algorithgm LoadMap
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-09 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
알고리즘 하나씩 뜯기위한 로드맵
## AIZ + Adaptive Tessellation 구현 로드맵 -UE5.7

> 목표: 플레이어 주변의 기본 Grid(눈 표면)를 CBT/LEB로 적응 분할하고, 이후 Snow Deformation RT, AIZ 계층까지 단계별로 연결

| 단계     | 내용                                                                                                 | 완료 기준 (AI에게 같이 전달)                                    | 주의                                                                                |
| ------ | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | --------------------------------------------------------------------------------- |
| **준비** | 플러그인 정리: 옛 `TessellationShader` 비활성화, 폴더명 오타(`Tessellaiton`) 수정, 모듈명·`_API` 매크로 통일                 | 빈 모듈이 에디터에서 빌드·로드됨                                    | 이름 충돌(`FAIZParams` 등)이 여기서 모두 정리돼야 함                                              |
| **0**  | Grid Component + SceneProxy 골격, CVar (루트는 사각형 = 삼각형 2개)                                            | 사각형이 보이고 `IZ.Enable 0`으로 사라짐                          | UE 5.7 API 기준으로 작성                                                                |
| **1**  | AIZComponent: 위치, 속도, 접촉 이벤트 GT→RT 전달                                                              | `IZ.Debug.PrintProxyState 1`에서 Proxy가 받은 위치가 플레이어와 일치 | Compensation 파라미터는 선언만                                                            |
| **2**  | CBT/LEB CPU 구현 + `IZRefineMetric` (거리 기반 베이스라인)                                                    | 플레이어 주변만 촘촘해짐, leaf 수가 상한 안에서 안정                      | CBT를 `uint32` 평탄 배열로 (GPU 이식 대비)                                                  |
| **3**  | Crack 검증, 레벨 컬러 디버그, 노드 카운트 (T-junction 검사 포함)                                                     | T-junction 자동 검사 0건                                   | 이 CPU 결과가 4단계의 **정답 데이터**                                                         |
| **4a** | 더미 Compute Shader + RDG Pass 뚫기                                                                    | readback으로 더미 값 확인                                    | 셰이더 디렉토리 매핑, `FGlobalShader` 선언                                                   |
| **4b** | CBT GPU 버퍼 업로드 + sum reduction (깊이마다 1 pass)                                                       | GPU 합 == CPU 합 (루트 값 = leaf 수)                        |                                                                                   |
| **4c** | Split → Merge (별도 dispatch)                                                                        | 같은 입력에서 CPU와 leaf 집합 일치                               | Split/Merge를 같은 pass에서 처리하지 않음                                                    |
| **4d** | Indirect Args 생성                                                                                   | leaf 수 → DrawIndirect 인자 정상                           |                                                                                   |
| **4e** | Indirect Draw (VS에서 leaf 디코드)                                                                      | 3단계와 같은 화면, CPU/GPU leaf 수 일치                         | Vertex Factory vs SceneViewExtension 방식을 **먼저 결정** (Depth Prepass/그림자 참여 여부가 달라짐) |
| **5**  | Snow RT 연동: VS displacement, PS normal, min/max mip, 삼각형 단위 displacement error                     | 발자국 위치만 변형·분할 증가, crack 0건                            | RT 갱신 패스가 tessellation 패스보다 먼저 실행되도록 RDG 의존성 확인                                   |
| **6**  | AIZ 계층: QuadTree, SSE, Importance Score, Target Level, Restricted QuadTree, Hysteresis, Predictive | 거리 베이스라인 대비 leaf 수·프레임 시간 비교 가능                       | 논문 핵심. 항목마다 CVar on/off (ablation용)                                               |
| **7**  | Compensation: Time-Sliced 보정, Lazy Stamp Replay, 예산 제어                                             | 갱신 건너뛴 영역이 매 프레임 갱신한 정답과 허용 오차 내 일치                   | 알고리즘 문서 3-B                                                                       |
| **8**  | Bounds/Culling, 프로파일링 (`RDG_GPU_STAT`, RenderDoc)                                                  | 패스별 ms 측정표 완성                                         | `CalcBounds`를 displacement min/max 기반으로 확장                                        |
1. 2번까지 진행

Plugin Module형식으로 붙힘
```
AIZTessellaitonShader/
├── AIZTessellaitonShader.uplugin
├── Source/
│   └── AIZTessellaitonShader/
│       ├── TessellationShader.Build.cs
│       ├── Public/
│       │   └── AIZTessellationCompoent.h
│       └── Private/
│           └── AIZTessellationCompoent.cpp
└── Shaders/
    └── .ush, .usf
```


- CVar로 토글 가능하도록 올리기

---
### 0. 작업 규칙

1. **CPU 레퍼런스 → GPU 이식** 순서를 지킨다. GPU 결과가 이상할 때 비교할 정답이 CPU에 있어야 한다.
2. 모든 기능은 **CVar로 토글**한다 (`IZ.Enable`, `IZ.Debug.LevelColor`, `IZ.Debug.Wireframe` 등). 
   문제 생기면 바로 끄고 비교 가능해야 한다.

---

### 1. 현재 위치 / 전체 지도

- [x] Player GameMode + 이동 구현 (실험용)
- [ ] 0 ~ 8단계 (아래)

```
Step0 Grid+Proxy 골격
  └ Step1 AIZComponent (GT→RT 데이터)
      └ Step2 CBT/LEB CPU + 거리 베이스라인
          └ Step3 Crack 검증 / 디버그 시각화
              └ Step4 RDG Compute 이식 (4a~4e)
                  └ Step5 Snow RT 연동 (Displacement/Normal/MinMax)
                      └ Step6 AIZ 계층 (논문 핵심)
                          └ Step7 Compensation
                              └ Step8 Bounds/Culling/Profiling
```

**모듈 구조 제안**

| 구분       | 클래스                                        | 역할                         |
| -------- | ------------------------------------------ | -------------------------- |
| Player 쪽 | `UAIZComponent : UActorComponent`          | 위치/속도/접촉/파라미터 수집, GT→RT 전달 |
| Grid 쪽   | `UIZGridComponent : UPrimitiveComponent`   | 그리드 소유, SceneProxy 생성      |
| 렌더 쪽     | `FIZGridSceneProxy : FPrimitiveSceneProxy` | RT 상태, 렌더                  |
| 알고리즘     | `FIZCBT`, `FIZLEB`, `IZRefineMetric.h`     | 엔진 의존 없는 순수 C++ (테스트 쉬움)   |
| 셰이더      | `Shaders/Private/IZ*.usf, *.ush`           | GPU 이식                     |

> `UActorComponent`는 SceneProxy를 가질 수 없다.
> 그래서 AIZComponent(데이터)와 GridComponent(렌더)를 분리한다.

---

### Step v3. Grid Component + SceneProxy 골격


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://www.youtube.com/watch?v=Hv6IKS_8Sp8&feature=youtu.be" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/Hv6IKS_8Sp8/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>AIZ V3</strong>
    </td>
  </tr>
</table>

**목표**: 평평한 사각형 하나가 `UPrimitiveComponent`로 화면에 그려진다.
**파일**: `IZGridComponent.h/.cpp`, `IZGridSceneProxy.h/.cpp`, `IZ.Build.cs`(모듈 의존성), CVar 정의 파일

``` Aiz Algorithm FrameWork
Player(Actor)
└ UAIZComponent ← 플레이어 위치/속도/접촉 이벤트만 수집 (데이터 공급)

Grid(Actor)
└ UIZGridComponent ← 그리드 소유 + 렌더링 담당
	└ FIZGridSceneProxy ← 렌더 스레드 쪽 객체
		├ FIZCBT ← 그냥 C++ 클래스 (컴포넌트 아님)
		├ FIZLEB ← 그냥 C++ 클래스
		└RefineMetric ← 함수

```

**할 일**
- [ ] `UIZGridComponent` 작성: `CreateSceneProxy()`, `CalcBounds()` (일단 고정 크기)
- [ ] `FIZGridSceneProxy::GetDynamicMeshElements()`에서 `FDynamicMeshBuilder`로 삼각형 2개(사각형) 출력
- [ ] 월드에 GridComponent를 가진 Actor 배치
- [ ] CVar `IZ.Enable` 추가
**공부 포인트**: GT 객체(Component)와 RT 객체(SceneProxy)가 왜 분리되는가. `FPrimitiveSceneProxy` 수명과 스레드 규칙.
**완료 기준**: 에디터/PIE에서 사각형이 보이고, `IZ.Enable 0`으로 사라진다.

---

