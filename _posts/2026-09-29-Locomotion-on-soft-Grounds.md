---
layout: post
published: true
title: 『Locomotion on Soft Grounds With Dynamic Footprints (2022)』
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-27 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
[Locomotion On Soft Grounds With Dynamic Footprints]([https://arxiv.org/abs/2209.10215](https://www.frontiersin.org/journals/virtual-reality/articles/10.3389/frvir.2022.801856/full)는
캐릭터가 눈, 모래, 진흙위를 걸을때, kinematic Animation만으로 **발**이 땅에 **가하는** 힘을 추정하고, 그 힘으로 **HeightField**를 Compression, 융기시켜 발자국을 만들고, 그 결과를 다시 **보행 자세**에 되먹임 하는 양방향 모델.

3장까지는 Character 애니메이션 관련이라, 4~5장 Deformation Model을 집중 조명.

Virtual Character의 운동학적

발이 지면에 닿을떄 가하는힘을 갖고 놀았음

Keyword : 
Subdivide Terrain patch proc 변형으로 효율적으로 지형에 적용

 최근 딥러닝 강화학습으로 애니메이션 개선하더라도 고정된 지면 위에서만 animation이 발생한다.

Future Works
고정 해상도 $256^2$ Grid로 하여서, Mesh 밀도 배분과 Tessellation을 지적