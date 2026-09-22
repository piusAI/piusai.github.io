---
layout: post
published: false
title: GPU Tessellation Terrain
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-18 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---

CPU에서 **Coarse Spatial Hierarchy**와 visibility결정, GPU에서는 patch 내부의 fine-grained geometric refinement를 Tessellation으로 수행, 이때 서로다른 LOD Patch사이의 T-Junction / Crack을 방지하기 위한 **Edge consistency 조건** 추가.



## 전체 구조
```
                    CPU
                     │
          ┌──────────┴──────────┐
          │                     │
       Quadtree             Frustum Culling
          │                     │
          └──────────┬──────────┘
                     ↓
                Terrain Patch
                     ↓
              ┌──────────────┐
              │ Vertex Shader│
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │ Hull Shader  │  ← Tessellation Factor
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │ Tessellator  │  ← GPU fixed function
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │ Domain Shader│  ← Heightmap → 위치 변위
              └──────────────┘
```
- **CPU** :큰 덩어리의 의사결정
  QuadTree로 지형을 Patch단위로 관리 ( coarse- grained) ,frustum Culling
- **GPU** 세밀한 병렬작업
  VS → `Hull Shader` → `Tessellator` →`Domain Shader`

DX11  `Rasterizer RenderPipeline`

기본적인 개념은


`HullShader` : Tessellation Factor 계산 - 거리, SSE, Height Variance 기반



---
- Indirect Draw;
  Gpu계산결과를 다시 CPU로 보내 읽어와야할텐데, GPU-driven의미가 사라지기에
- GPU 메모리상의 버퍼를 가리키기만 한다.
#### Low - high LOD Patch

01 Tessellation Factor Matching
02 Edge Stitching
03 Skirt
04 Vertex Morphing
05 Power - of - Tow Tessellation