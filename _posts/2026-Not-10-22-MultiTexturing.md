---
layout: post
published: false
title: Phong Illumination, Phong Shading
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-08 21:22:00 +0900
description: AnimationSandMud
categories:
  - Graphics
tags:
  - Graphics
use_math: true
author: PIUS
---
Multi Texturing


--- 
### 1. Normal Mapping
- Bump & Dent : 울퉁 불퉁한 표면 표현
- RGB -> XYZ : Position정보 색상정보
이미지들끼리 합성하기 좋기 떄문에 NM  Texture 이미지로 저장



RGB Packing
``` cpp
#include <cstdint>

// 패킹: 0xAARRGGBB
inline uint32_t PackARGB(uint8_t a, uint8_t r, uint8_t g, uint8_t b)
{
    return (uint32_t(a) << 24) |
           (uint32_t(r) << 16) |
           (uint32_t(g) << 8)  |
            uint32_t(b);
}

// RGB만 (알파 없음)
inline uint32_t PackRGB(uint8_t r, uint8_t g, uint8_t b)
{
    return (uint32_t(r) << 16) |
           (uint32_t(g) << 8)  |
            uint32_t(b);
}

// 언패킹
inline uint8_t GetR(uint32_t c) { return (c >> 16) & 0xFF; }
inline uint8_t GetG(uint32_t c) { return (c >> 8)  & 0xFF; }
inline uint8_t GetB(uint32_t c) { return  c        & 0xFF; }
inline uint8_t GetA(uint32_t c) { return (c >> 24) & 0xFF; }
```


### 02. Tangent Space의 이유와 순서
1. Normal Map(Texture)에서의 Coordinate 는 "저장된 공간기준"
2. World내 Tangent Coordinate 재정의
3. Normal 적용

4. point마다의 @Up, @N, @Right

| 기호  | name      | Houdini | 의미                |
| --- | --------- | ------- | ----------------- |
| T   | Tangent   | @Up     | 표면의 접선 방향         |
| B   | Bitangent | @Right  | 표면을 따라가는 다른 접선 방향 |
| N   | Normal    | @N      | 표면 바깥쪽을 향하는 법선 방향 |


$$N_{TS}=2RGB-1$$



<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Shader/SphereVector.png" alt="SphereVector" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SphereVector</strong> </td> <td width="48%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Shader/SphereMeshVector.png" alt="SphereMesh" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>SphereMeshVector</strong> </td> </tr> </table>

- Normal : 푸른색
- Right Vector : x Axis - 녹색
- Up Vector : y Axis - 빨간색
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Shader/TangentSpace.png" alt="TangentSpace" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>TangentSpace</strong> </td>  </tr> </table>




---
### Animation

시간에 따라 Object의 Space상의 이동

1. Computer Assisted Animation - determine
2. Computer Generated Animation - undetermine

Physics-based Animation


- Rigid body animation

$𝑉_{World} = (1− 𝑎) × 𝑉_1 + 𝑎 × 𝑉_2,  (0 ≤ 𝑎 ≤ 1)$

- Articulated body:
  graph 구조로 표현 가능

$M_{transform} = M_{local} × M_{animation} × M_{parent}$

graph의 이점 : 제어, 검색이 빠르겠다.



$M_{boneTransform} = M_{boneLocal} × M_{boneAnimation} × M_{boneParent}$

$M_{World} = M_{local} × M_{meshLocal} × M_{boneTransform}$



