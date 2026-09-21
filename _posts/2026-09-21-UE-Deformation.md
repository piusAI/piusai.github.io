---
layout: post
published: true
title: Unreal Deformation 변형 방식 3가지
subtitle: Constraint
date: 2026-09-21 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
#### RenderTarget 기반 Snow Deformation 접근 방식?
눈과 같은 Deformation은 Terrain 지오메트리 만든 이후에 **RT에 저장된 변형값을 geometry에 반영하는 단계**이다, Unreal engine에서는 3가지로 크게 진행할 수 있는데 그 3가지를 알아보겠다.

---
### 공통구조
1. 충돌체(발, 차량 등) Deformation RT에 기록(Height offset, 깊이, 복원 상태)
2. Terrain Geometry 단계에서 그 RT를 샘플링해 Vtx 위치를 밀어낸다

세 방식은 **RT를 어떤 Geometry 표현에 적용하냐**에서 갈린다.

### 01 Render Target Vertex Offset Displacement ★★★
- Terrain Patch를 LOD Metric으로 선택, Tessellation(Hull/Domain Shader)로 세분화 한뒤, Domain Shader / Vertex Shader에서 Deformation RT를 샘플링하여 정점의 높이를 Offset 한다.
- 구현이 단순, 기존 Terrain Rendering pipeline에 그대로 끼워넣을 수 있다
- 한계
  - Deformation resolution이 **RT Resolution, Tessellation Resolution**에 둘다 묶임
  - World 전체를 덮기위해, RT 크기 폭증, 카메라 중심 Scrolling RT나,Clipmap이 필요
  - Tessellation 계수 낮은곳에서 detail 뭉개짐
  - LOD 경계에서의 Crack, Popping 생기기 쉬워 Normal은 별도 RT에서 재구성!

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/TerrainOpti/RenderTarget/RTSphere.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RenderTarget+VS</strong> </td>  </tr> </table>
ComputeShader를 활용해 Subsystem에 결합한 Sphere + Vertexshader

- 대표 사례
  01 Assassin's Creed III : SnowDeformation  
  02 Rise Of the Romb Radier : Snow / Mud Deformation  
  03 Batman;Arkham Origin : Snow
**내 논문**에서 채택한 방식


### 02 Virtual Heightfield Mesh ★★☆

- Runtime Virtual Texture에 **Height까지 함께 기록**하고, 전용 grid mesh가 높이 sampling 변형
- Terrain 기본 Geometry와 분리해서 **높이 정보를 Virtual Texture 계층**에서 관리
- Sparse하게 streaming되어서 큰 월드에 유리
- **한계**:  
  RVT 갱신 비용, 자주 바뀌는 Deformation data는 cache효율 떨어져서 실시간 동적 변형에서 부담

### 03 RenderTarget+Nanite ★★☆
Nanite는 VtxShader가 매 프레임 자유롭게 정점을 움직인는 구조가 아닌,  
미리 빌드된 **Cluster 계층**를 GPU-Driven으로 Culling, Rasterizer하는 구조  
→RT deformation 붙이는 방법이 제한적, 3갈래

- **Nanite Tessellation + Displacement**: 머티리얼의 Displacement 값으로 클러스터 단위 Tessellation과 변위를 적용. 이론상 RT를 샘플링해 변위에 넣을 수 있지만, Nanite 메시의 클러스터 바운드는 빌드 시점에 정해지므로 **RT에 의한 큰 변위는 바운드 확장을 미리 설정해야** 컬링이 깨지지 않는다.
- **WPO(World Position Offset) on Nanite**: 머티리얼에서 RT를 샘플링해 정점을 밀어내는 방식. 지원되지만 Nanite에서는 비용이 크고, 클러스터 바운드와 컬링, 그림자(VSM) 캐시 무효화 이슈가 따라옴. 변형 영역이 넓을수록 성능이 나빠짐.
- **Nanite Landscape / Heightfield Nanite화**: Terrain 자체를 Nanite로 만들면 Partitioning, Culling, LOD가 Nanite 안에서 처리되어 별도 LOD Metric이 필요 없어짐. 대신 **동적 변형은 정적 클러스터와 잘 맞지 않아서**, 변형 부분만 다른 경로(RT 기반 오버레이나 별도 메시)로 처리해야 하는 경우가 많다.

---

### 비교

| 방식                              | 지오메트리 표현                | RT 적용 지점          | 강점                  | 약점                   |
| ------------------------------- | ----------------------- | ----------------- | ------------------- | -------------------- |
| RT Vertex Offset (Tessellation) | 패치 + HW/SW Tessellation | Domain Shader     | 단순, 국소 변형 적합        | RT 해상도 한계, LOD crack |
| Virtual Heightfield Mesh        | 그리드 + RVT Height        | Vertex 샘플링        | 대규모 월드, Sparse 스트리밍 | 동적 갱신 시 캐시 비용        |
| RT + Nanite (Displacement/WPO)  | Cluster 계층              | Material 변위 / WPO | 초고밀도 지오메트리          | 바운드·컬링·그림자 캐시 문제     |

#### 정리

- 01번은 가볍고 기존 파이프라인에 잘 붙으므로, RT 해상도와 LOD Crack 문제만 잘 풀면 실시간 눈 변형에 가장 실용적
- 02, 03번은 HW Tessellation 없이도 고밀도 지오메트리를 확보하는 방식이라, Terrain 파이프라인의 Tessellation 단계 자체가 필요 없다. 대신 RVT 갱신 비용이나 Nanite의 정적 클러스터 제약이라는 다른 문제