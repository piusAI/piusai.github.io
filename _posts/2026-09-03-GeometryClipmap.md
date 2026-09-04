---
layout: post
published: true
title: GeometryClipmap 최적화?
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-01 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
**Terrain**에 맞는 Vertex 최적화 방법중 가장 기초인
Geometry Clipmap에 대해 알아보자

## Geometry ClipMap

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/TerrainOpti/clipmaps_01.jpg" alt="PS001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Geometry Clipmap</strong> </td> <td width="30%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/TerrainOpti/clipmaps_02.jpg" alt="PS002.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Terrian Render Geo Clipmap</strong> </td> </tr> </table>
Main 위치를 가운데 기준으로, LOD형태를 뿌린것이다.  
경계 주변의 Artifacts를 방지!  

Resolution간의 경계의 Articact 방지가 중요.
-> 2의 배수로 vtx matching 시키기위해 vertex index를 맞춰야함.

#### 2.1 데이터 구조
- constant buffer : Vertex, Index buffer 세트만ㅇㄹ 미리 정의, 틀 고정
- Texture Map : 지형 높낮이, 표면 방향 정보 Buffer가 아닌, Clip-map Level마다 2D texture로 따로 저장
- VRAM : 모든 Buffer, Data를 system memori가 아닌, VRAM에 올려두고 GPU가 곧바로 가져다 쓰도록 설계

#### 2.2 Clipmap 크기
- n = 2^k  -1로, Level에 대해 Low-High 정확히 중앙에 위치하지 않는다는 이점.
- High Level들이 low level에 대해 중앙 벗어날 수밖에 없도록!

- `n*n` $m = \frac{n + 1}{4}$ 으로, n=255의 경우 64x64개의 격자 간격으로 이루어져있음.
-> 이 `m*m`크기로 vtx grid위에서 삼각형들을 채우는(rasterization)것.

#### 2.3 Vertex / index Buffer
- Ring을 일정한 블록으로 쪼개면, View Frustrum culling이 가능해짐.
- 똑같은 하나의 Vtx/Index buffer 메모리에 딱 하나만 올려두고, VtxShader에서 Scale, Offset Transform parameter만 다르게 주어 terrain block을 우려먹음. 

#### 2.3.3 Draw Primitive + View Frustrum culling


```
		[Top-Left]   [Top-Center]   [Top-Right]
    +-------------+--------------+-------------+
    |  블록 (1)   |   블록 (2)   |  블록 (3)   |
    +-------------+--------------+-------------+
[L] |  블록 (4)   |   (여기는     |  블록 (5)   |     [R]
    +-------------+   안쪽 구멍   +-------------+     i
    |  블록 (6)   |   뚫려있음)  |  블록 (7)   |        g
    +-------------+--------------+-------------+     h
    |  블록 (8)   |   블록 (9)   |  블록 (10)  |        t
    +-------------+--------------+-------------+
       [Bottom-Left] [Bottom-Center] [Bottom-Right]
         (여기에 블록 11, 12 등 추가 배치)
```
위와같은 상황에서 카메라 컬링이 들어가면 4개에 내부 2개,

```
	+-------------------+-------------------+
    |                   |                   |
    |     블록 (1)      |     블록 (2)      |
    |                   |                   |
    +-------------------+---------+---------+
    |                   |         | L자형   |
    |     블록 (3)      | 블록(4)   | 스트립  |
    |                   |         | (5번 조각)|
    +-------------------+---------+---------+
```
위와같이 내부 구멍을 채우기위한 5번 호출
평균 총 6L + 5번의 DP가 발생

렌더 파이프라인을 이해한 이후,  RenderTarget이라는 것이 단순한 화면 출력용을 넘어,  
그래픽스 메모리를 다루는 유연한 C++의 템플릿같은 도구라는 것을 알게 되었다.  


##### Compute Alpha, Blend Elevation 수식
$$\alpha_x = \text{clamp}\left(\left(x - v_x - \left(\frac{n - 1}{2} - w - 1\right)\right) / w, 0, 1\right)$$

0~1로써 Clamp!

$$x - v_x $$
-> 카메라와의 거리 : 현재 정점 위치 - camera 위치 상대거리

$$\frac{n - 1}{2} - w - 1$$
-> 기준점으로 Clipmap 전체크기 n, 블랜딩 폭 w고려해서 어느지점부터 blending 할것인지의 고정상수

(W)로 나누어주는 이유는 Linear Interpolation으로 증가시키기 위함

```
float2 alpha = clamp((abs(worldPos - ViewerPos) – AlphaOffset) * OneOverWidth, 0, 1);
```
위 수식의 코드


``` hlsl
float zf = floor(zf_zd);
float zd = frac(zf_zd) * 512 - 256; // (zd = zc - zf)
```
텍스쳐 드로우 콜을 줄일려고 한번에 패킹한뒤
floor : 정수부, frac : 소수부로 끄집어내어서 활용

``` hlsl
flaot z = zf + alpha.x * zd;
z = z * ZScaleFactor;
output.pos = mul(float4(worldPos.x, worldPos.y, z, 1), WorldViewProjMatrix);
```
VS에서 Projection 행렬로가기전에 들어감
-> blending시켜 자연스레 곱해줌


#### 2.3.6 Pixel Shader

``` hlsl


..

//001 VS에서 계산한 Alpha를 활용한 Normal Blending
float3 normal = float3((1-alpha) * normal_fc.xy + alpha * normal_fc.zw ,1) ;

//002 [0,1] ~> [-1, 1] 정규화 수정
normal = normalize(normal*2 -1);

// 003
float s = clamp(dot(normal, LightDirection), 0, 1);
return s * tex1D(ZBasedColorSampler,z);
// elevation 기반으로 terrain 색상 assign

```

- `normal map sample` : normalMap가진 2dTexture
- 001) `normal_fc.xy` 현재 Level의 normal, `normal_fc.zw`는 거친 Level의 Normal
- 002) Texture format [0, 1]범위를 3D Vector 범위 [-1, 1]fh wjdrbghk
- 003) 지형 색상 매핑하는 `ZbasedColorSampler`에 최종 색상 뱉어냄
- `ZBasedColorSampler` : 지형 높이`z`를 이용해, 모래, 풀, 눈 등 색으로 1D gradient colormap


##### Clip-map Level 2가지 TExture 저장
- 1channel float `Height map`
- 4channel (8bit per channel) `Normal Map`

##### Toroidal Addressing? (wrapping Addressing)
- Modulo(`%`)와 같이 순환구조인 Torus
  UV [0~1]이상이 될때, 예로, 1.2 ->0.2로의 Modulo
- Texture의 물리적 위치 그대로두고, Shader 바라보는 offset만을 순환 구조로 swap!

##### L자형?
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/TerrainOpti/L_torus.jpg" alt="PS001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Toroidal update</strong>  </td> </tr> </table>
Toroidal 업데이트로 texture 경계 넘어가는경우 새로 갱신은 새로 드러난 L자영역!

##### Residual 잔차?

``` hlsl
//업샘플링 뼈대 구축
float z_predicted Upsample(uv);

// 잔차 가져오기
float residual tex2D(residualSampler, p_uv * Scale);

// 세밀한 높이 합성
float zf = z_predicted+ residual;

//패킹 트릭
float zd = zc-zf;
float zf_zd = zf+(zd+256) / 512;
```

- 세밀한 높이(`zf`)는 부동소수, 
- `zd` = 낮은 Res level 높이(`zc`) - 높은 Resolution level 높이(`zf`)


#### Keyword
`torus` texture 좌표 메모리는 그대로두고, `Wrap`와 같이 Modulo처럼 offset 순환시켜 메모리 아낌
`residual` 을 통한 계층적 갱신 : Low Level을 Upsample해 뼈대 예측, 진짜 고해상도와의 오차인 잔차`Residual`를 더해 완벽한 `zf`를 합성하는 알고리즘!




[참고문헌]
Asirvatham, Arul, and Hugues Hoppe. 2005. "Terrain Rendering Using GPU-Based Geometry Clipmaps." In *GPU Gems 2: Programming Techniques for High-Performance Graphics and General-Purpose Computation*, edited by Matt Pharr, Chapter 2. Addison-Wesley.
https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-2-terrain-rendering-using-gpu-based-geometry