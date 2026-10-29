---
layout: post
published: true
title: "GPU Gems 2 : Generic Refraction Simulation -작성중"
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-10-29 17:22:00 +0900
description: Refraction
categories:
  - Graphics
tags:
  - Graphics
use_math: true
author: PIUS
---


### Generic Refraction Simulation

빛이 튀는 모든 환경을 실행중에 계산하는 방식, 환경 맵을 미리 구워 Runtime에 사용하는 방식이 있지만, 
$\rightarrow$ 텍스쳐 메모리 사용량이 많아 성능 저하가 발생한다.
그렇기 때문에 

Scene내 굴절되는 객체 이미지를 Color로 가져오는것이 아닌, Texture Coordinate를 변경하는 방식으로 표현!
### 01. Concepts:  



현재 Back buffer를 굴절 맵으로 사용, Texture coordinate에 변위 더해 굴절 효과 Simulation하는 기본 기법


기본 굴절 기법 단계
1. 모든 굴절 Mesh 건너 뛰고 Scene geometry를 Texture로 렌더링
   후속 pass에서 Rendering될 굴절 객체 뒤에 보이는 객체를 결정하는데 사용 가능
   : 텍스쳐$S$.
2. 굴절 메쉬 Rendering, $S$에서 값 조회, 해당값에 Perturbation(교란) 적용

굴절(기본) :  

``` cpp hlsl
//dx11
cbuffer RefractionCB : register(b0)
{
	float4 vScale;
}

//Sampler2D = Texture2D + SamplerState 두개로 분리
Texture2D tex0 : register(t0);    // bump(normal) map
Texture2D tex1 : register(t1);    // refraction map - Scene Render Tx
SamplerState samp : register(s0);

struct PS_INPUT
{
	float4 pos : SV_POSITION;
	float2 bumpUV : TEXCOORD0;
	float4 screenPos : TEXCOORD1;
};

float4 main(PS_INPUT input) : SV_Target
{
	//bump 텍스쳐 Fetch 후 [0 .. 1] -> [-1 .. 1] unpack
	float4 bumpTex = 2.0 * tex2D(tex0, input.bumpUV) - 1.0;
	
	// Texture 좌표 displace
	float2 newUV = (input.screenPos.xy / input.screenPos.w) + bumpTex.xy * vScale.xy;
	
	// refraction map fetch
	return tex1.Sample(samp, newUV);
}

```

1. 굴절되는 물체(물,유리) 를 제외한 Scene을 offscreen - RenderTarget에 먼저 렌더링
2. 그 RT의 SRV(ShaderResourceView)를 `t1`슬롯에 바인딩 (refraction Map)
3. 물/ 유리 mesh 그릴 때 이 shader 사용 : `screenPos`에 `normalmap offset`을 더해서 2번에서 만든 Scene texture의 다른 위치를 읽어옴

- `screenPos.xy / screenPos.w` : perspective divide로, Clip space Coordinate$\rightarrow$ NDC로 바꿈
- Shader 안에서 `tex1`이 그냥 Texture slot

- [TextureObject::Sample](https://learn.microsoft.com/ko-kr/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-sample)로 Uv Texture Coordinate 넣어서 바꿔줌

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RefractionMask.png" alt="RefractionMask" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RefractionMask</strong> </td>  </tr> </table>
앞 Geometry의 Perturbation을 받지 않도록 위와 가운데 이미지와 같이 FrameBuffer alphachannel로 잘라서 그 영역의 경우만 Refraction적용되도록 함


굴절 (Alpha blending):  

``` cpp hlsl
//dx11
cbuffer RefractionCB : register(b0)
{
	float4 vScale;
}

//Sampler2D = Texture2D + SamplerState 두개로 분리
Texture2D tex0 : register(t0);    // bump(normal) map
Texture2D tex1 : register(t1);    // refraction map - Scene Render Tx
SamplerState samp : register(s0);

struct PS_INPUT
{
	float4 pos : SV_POSITION;
	float2 bumpUV : TEXCOORD0;
	float4 screenPos : TEXCOORD1;
};

float4 main(PS_INPUT input) : SV_Target
{
	//bump 텍스쳐 Fetch 후 [0 .. 1] -> [-1 .. 1] unpack
	float4 bumpTex = 2.0 * tex2D(tex0, input.bumpUV) - 1.0;
	
	float2 vProj = (input.screenPos.xy / input.screenPos.w);
	
	// refraction map fetch
	float4 vRefrA = tex1.Sample(samp, vProj.xy + bumpTex.xy * vScale.xy); //distortion UV
	float4 vRefrB = tex1.Sample(samp, vProj.xy); //original uV
	return vRefrB * vRefrA.w + vRefrA( 1 - vRefrA.w);
}

```
