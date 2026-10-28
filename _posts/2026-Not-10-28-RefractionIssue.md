---
layout: post
published: false
title: Refraction
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-08 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
hlsl, ComputeShader 를 활용해서 Refraction의 Issue 대응 / 계산 속도 개선 등을 Reserach해보겠다.


--- 
### 1. UE - Material 속성 : RefractionDepthBias
- UnrealEngine Material 자체에서 굴절 Bias를 주는 방식

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RefractionDepthBias.png" alt="RefractionDepthBias" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RefractionDepthBias : 20 / 50 / 100</strong> </td>  </tr> </table>
이렇게 깊이 편향을 조절해서 전경 누출을 줄일 수 있따.
다만 고정된 Bias조절이라, Scene마다의 값을 직접 맞춰야하기에 depth차이에 따라 적응 불가능하다.


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RefractionMaterial.png" alt="![[Pasted image 20261028151951.png]]" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Refraction Material setting</strong> </td>  </tr> </table>
- RefractionDepthBias수정으로 가능



#### Refraction Keyword
Refraction은 화면 공간(Screen-Space) 왜곡, 투명체 RayMarching/RayTracing 과정에서 극심한 고주파 노이즈와 깜빡임(Shimmering)을 유발한다. 아래의 작동 원리 파악하기!

#### 01 TAA(Temporal Anti-Aliasing | 시간적 안티 에얼리싱)
TAA는 현재 프레임, 이전 프레임들의 정보(History)를 시간적으로 누적(Accumulation)하여 계단 현상(Aliasing)과 픽셀 깜빡임을 부드럽게 상쇄 시키는 기법이다.

- 작동 원리:
1. Jittering : 매 프레임마다, Camera Projection Matrix를 미세히 흔들어(Sub-Pixel Jitter) 매번 다른 샘플링 위치를 갖게 한다.
2. Motion Vector : Object - Camera 움직임 정보를 담은 Motion Vector Buffer를 이용해, 이전 Frame의 어느 픽셀이 현재 Frame의 어느 위치에 대응 되는지를 추적
3. History Blending : 현재 프레임의 Sample, 이전 프레임의 복원된(Reprojected) 색상을 Blending해서 AntiAliasing 효과를 극대화 한다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/TAAChain.png" alt="![[Pasted image 20261028151951.png]]" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Refraction Material setting</strong> </td>  </tr> </table>
[Anti Aliasing/Upscaling](https://dev.epicgames.com/documentation/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine)에서 알 수 있듯, 아래와 같은 Rendering Chain을 가진다.
Motion Blue 이전에 TAA가 박힘.

#### 02 TSR(Temporal Super Resolution) : 




#### Refraction Rendering 흐름

|시기|흐름|핵심|
|---|---|---|
|1980|Whitted 레이 트레이싱|굴절/반사 레이를 재귀로 쏨. 정확하지만 느림|
|1996|포톤 매핑 (Jensen)|커스틱스까지 가능한 오프라인 GI|
|2000년대 초|환경맵 기반 굴절|레이 추적 없이 환경맵 조회. 빠르지만 자기 가림, 주변 물체 반영 불가|
|2005|Wyman 이미지 공간 굴절|깊이 두 장으로 탈출점을 근사. 이 논문(Kim & Hong)의 계보|
|2005~2007|Distance impostor, Caustics mapping, Eikonal|GPU 근사 방법들이 쏟아진 시기|
|2010년대|스크린 스페이스 레이 마칭(SSR 등)|깊이 버퍼를 레이 마칭. 게임 엔진에 정착|
|2018~|하드웨어 레이 트레이싱 (RTX)|실시간에서 정확한 굴절이 가능해졌지만 비용이 큼|


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Shader/SphereVector.png" alt="SphereVector" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SphereVector</strong> </td> <td width="48%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Shader/SphereMeshVector.png" alt="SphereMesh" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SphereMeshVector</strong> </td> </tr> </table>

- Normal : 푸른색
- Right Vector : x Axis - 녹색
- Up Vector : y Axis - 빨간색

