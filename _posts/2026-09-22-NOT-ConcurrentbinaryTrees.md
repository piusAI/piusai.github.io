---
layout: post
published: false
title: Concurrent Binary Trees
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-21 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---

## Concurrent BinaryTrees
CPU에서 **DrawCall**을 보내는 권장 최대 Terrain Component수 : 1024
LOD보다, "Streaming"으로
- World를 Grid-Cell로 나누어서 camera 주변 Cell만 메모리에 Load/Unload
- 먼 셀은 HLOD(HierarchicalLOD)로 미리 구워둔 Mesh로 대체 - runtime이 아닌, Editor에서 **미리 Bake**해두는 방식
- RVT(Runtime Virtual Texture)로 텍스쳐도 필요한 영역만 메모리에 유지

## 전체 구조
```
                 UE5 Large World
                       │
               ┌───────┴────────┐
               │ World Partition│
               └───────┬────────┘
                       │
                Grid Cell Streaming
                       │
          ┌────────────┴─────────────┐
          │                          │
     Loaded Cell               Unloaded Cell
          │                          │
    Landscape Component             HLOD(Hierachical LOD)
          │
     ┌────┴────┐
     │         │
 Non-Nanite  Nanite
     │         │
 Landscape    Nanite
 LOD          Cluster/LOD
     │         │
     └────┬────┘
          ↓
        GPU
```


`UWorldPartitionRuntimeCellDataSpatialHash`로써, `Spatial Hash`방식.  
Quadtree처럼 Tree를 순회하는것이 아님.  
Tree traversal이 아니고, Flat Grid를 좌표 해싱을 직접 찾아서 더 단순함
- HLOD : 

### WorldPartition
```
              World Partition

┌────┬────┬────┬────┬────┐
│    │    │    │    │    │
├────┼────┼────┼────┼────┤
│    │LOAD│LOAD│    │    │
├────┼────┼────┼────┼────┤
│    │LOAD│CAM │    │    │
├────┼────┼────┼────┼────┤
│    │    │    │    │    │
└────┴────┴────┴────┴────┘
```
Sublevel -> Level Streaming으로 Player가 Landscape이동시, Loading, Unloading을 해야했다.


데이터 관리, 거리기반 level Streaming 시스템


---

#### HLOD


[HLOD UE Docs](https://dev.epicgames.com/documentation/unreal-engine/hierarchical-level-of-detail-in-unreal-engine) 계층형 LOD으로, SM actor 정리, 단일 proxy mesh Material로 결합 가능  
Rendering해야하는 actor 수 줄이는데 도움됨  
: Frame당 DrawCall 수 줄여서 퍼포먼스 높일 수 있다.

| HLOD 타입             | draw call 줄이는 방식                | Atlas 필요?                         |
| ------------------- | ------------------------------- | --------------------------------- |
| **Instancing**      | 동일 메시를 GPU instancing으로 한 번에 그림 | ❌ 필요 없음  <br>(텍스처가 이미 공유됨)        |
| **Merged Mesh**     | 여러 메시를 하나로 합침                   | ✅ 필요 <br>(Material 다르면 Atlas로 통일) |
| **Simplified Mesh** | 합친 뒤 폴리곤 수까지 줄임(decimation)     | ✅ 필요                              |
| **Approximation**   | 아예 새로 리메싱(remesh)해서 근사          | ✅ 필요                              |

01 Draw Call을 줄여야한다
02 Mesh Merge + Texture Atlas를 해준다

