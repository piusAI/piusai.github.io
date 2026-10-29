---
layout: post
published: true
title: "GPU Gems 2 : Generic Refraction Simulation"
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
굴절의 기본 쉐이더를 이해 해본다.

참고 : [GPU Gem2, ch.19 : Refraction Simulation](https://developer.nvidia.com/gpugems/gpugems2/part-ii-shading-lighting-and-shadows/chapter-19-generic-refraction-simulation)

### 01. 배경:  
굴절을 정확히 이해하는 방법은 두가지가 있다.

- 빛이 튀는 환경을 런타임에 모두 계산 → 비용이 크다
- 환경 맵을 미리 구워서 사용 → 텍스처 메모리 사용량이 커서 성능이 떨어진다

그래서 Scene내 굴절되는 **객체의 색(Colo)을 직접 구하지 않고, Texture Coordinate(Uv 텍스쳐 좌표)를 변위 시켜** 이미 렌더링된 Scene 이미지를 다른 위치에서 읽는 방식으로 굴절을 흉내 낸다.


### 02. 기본 굴절 구현:   

핵심 아이디어 : 현재 Back buffer를 굴절 맵으로 사용, UV coordinate에 변위 더해 굴절 효과를Simulation한다.

#### 렌더링 순서
1. 굴절 메쉬를 **제외**한 Scene을 Texture `S`로 렌더링 (굴절 객체 뒤에 보이는 것들)
2. 굴절 메쉬를 그릴 때 `S`를 조회하되, 노멀맵 값으로 **UV를 교란(perturbation)**


#### DX 굴절(기본) 구현 :  

``` cpp
//dx11 .hlsl
cbuffer RefractionCB : register(b0)
{
	float4 vScale; //xy : uv distortion 세기
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
	float4 bumpTex = 2.0 * tex0.Sample(samp, input.bumpUV) - 1.0;
	
	// Texture 좌표 displace
	float2 newUV = (input.screenPos.xy / input.screenPos.w) + bumpTex.xy * vScale.xy;
	
	// refraction map fetch
	return tex1.Sample(samp, newUV);
}
```

#### binding
1. 굴절되는 물체(물,유리) 를 제외한 Scene을 offscreen - RenderTarget에 먼저 렌더링
2. 그 RT의 SRV(ShaderResourceView)를 `t1`슬롯에 바인딩 (refraction Map)
3. 물/ 유리 mesh 그릴 때 이 shader 사용 : `screenPos`에 `normalmap offset`을 더해서 2번에서 만든 Scene texture의 다른 위치를 읽어옴

#### 참고

- `screenPos.xy / screenPos.w` : perspective divide로
  Clip space Coordinate$\rightarrow$ NDC로 바꿈
- Shader 안에서 `tex1`이 그냥 Texture slot

- [TextureObject::Sample](https://learn.microsoft.com/ko-kr/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-sample)로 Uv Texture Coordinate 넣어서 바꿔줌



### 03. DX 굴절 Mask 처리:

UV를 왜곡하면 굴절 객체 **앞에 있는 물체**가 굴절 영역 안으로 끌려 들어오는 문제가 생긴다.
이를 막기위해 Scene texture의 **Alpha Channel을 mask**로 사용
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/RefractionMask.png" alt="RefractionMask" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RefractionMask</strong> </td>  </tr> </table>
앞 Geometry의 Perturbation을 받지 않도록 위와 가운데 이미지와 같이 FrameBuffer alphachannel로 잘라서 그 영역의 경우만 Refraction적용되도록 함

####  (Alpha blending) 구현:

``` cpp
//dx11 - .hlsl
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
	return vRefrB * vRefrA.w + vRefrA * ( 1 - vRefrA.w);
}
```

- `vRefrA` : 왜곡된 UV로 읽는 값
- `vRefrB` : 왜곡 없는 원래 UV로 읽는 값
- `vRefrA.w`가 크면(왜곡된 위치에 앞 물체 걸림) : 원래 위치의 `vRefrB`사용


---
### 04. 물 굴절 구현:   

#### 기존 방식과의 차이
- 기존 방식 : Re-Rendering, 수면 위 Geometry Cliping 추가 단계 거쳐 Refraction Map 생성
- 이 방식 : Refraction Mask로, **수면(WaterPlane)을 $S$ Alpha에 함께 기록**하므로, 굴절 맵을 위한 **별도 Pass가 필요 없다**.

#### 렌더링 단계
1. Water Plane을 굴절 Mask용 BackBuffer alpha에 기록 - (굴절 마스크)
2. Water plane통해 반사된 Scene, 반사 Map에 기록
3. 수면 아래 Geometry 잘라내 물 반사 맵 생성
4. $S$와 굴절 Mask를 alpha 채널 하나로 같이 쓴다.


#### Bump layer
1. 4개의 Bump layer사용
2. 각 레이어의 Tx 좌표는 **VtxShader**에서 layer마다 Scaling(타일링)과 scroll 값을 다르게 주어서 계산
   - 큰 Scale : Noise 넓고 느린 물결 (저주파)
   - 작은 Scale : Noise 좁고 빠른 물결(고주파)
   - 이를 적절히 섞어 반복감 줄이고 Detail 만듦( Niagara curl Noise value와 유사)

``` cpp
//dx11 .hlsl
float Fresnel(float NdotL, float fresnelBias, float fresnelPow)
{
	float facing = (1.0 - NdotL);
	return max(fresnelBias + (1.0 - fresnelBias) * pow(facing, fresnelPow), 0.0);
}

```



```cpp
//dx11 .hlsl


Texture2D tex0 : register(t0);  // bump (normal map)
Texture2D tex1 : register(t1);  // refraction
Texture2D tex2 : register(t2); // reflection
SamplerState samp : register(s0);

struct PS_Input{
	float3 Eye : TEXCOORD0;
	float4 Wave0 : TEXCOORD1; //xy = uv, w = distance
	float2 Wave1 : TEXCOORD2;
	float2 Wave2 : TEXCOORD3;
	float2 Wave3 : TEXCOORD4;
	float4 ScreenPos : TEXCOORD5;
};

float4 main(PS_Input input) : SV_Target
{
	float3 vEye = normalize(input.Eye);
	
	// 1. Bump Layer 4개 합성
	float3 vBumpTexA = tex0.Sample(samp, input.Wave0.xy).xyz;
	float3 vBumpTexB = tex0.Sample(samp, input.Wave1).xyz;
	float3 vBumpTexC = tex0.Sample(samp, input.Wave2).xyz;
	float3 vBumpTexD = tex0.Sample(samp, input.Wave3).xyz;
	
	float3 vBumpTex = normalize( 2.0 * (vBumpTexA + vBumpTexB + vBumpTexC + vBumpTexD) - 4.0);
	
	// 2. Refraction/Reflection 개별 bump scale
	float3 vRefrBump = vBumpTex * float3(0.075, 0.075, 1.0);
	float3 vReflBump = vBumpTex* float3(0.02, 0.02, 1.0);
	
	// 3. Projected Coordinates - 투영 좌표
	float2 vProj = input.ScreenPos.xy / input.ScreenPos.w;
	
	//4. 반사/굴절 샘플
	float4 vReflection = tex2.Sample(samp, vProj + vReflBump.xy);
	float4 vRefrA = tex1.Sample(samp, vProj + vRefrBump .xy);
	float4 vRefrB = tex1.Sample(samp, vProj);

	// 5. Refraction map Mask Occluders - 차폐 mask 보정
	float4 vRefraction = vRefrB * vRefrA.w + vRefrA * (1.0 - vRefrA.w);
	
	// 6. Fresnel
	float NdotL = max(dot(vEye, vReflBump), 0.0);
	float facing = 1.0 - NdotL;
	float fresnel = Fresnel(NdotL, 0.2, 5.0);
	
	
	// 7. 거리 기반 굴절 색 깊은 물 색보간
	// Refraction / deep water color와 Distance-based Lerp!
	float fDistScale = saturate(10.0 / input.Wave0.w);
	float3 deepColor = float3(0.0, 0.15, 0.115);
	float3 WaterDeepColor = vRefraction.xyz * fDistScale + (1.0 - fDistScale) * deepColor;

	// 8. View angle 기반 물 색상 보간
	// Lerp Between Water Color and Deep water color
	float3 WaterColor = deepColor;
	float3 waterColor = WaterColor * facing + WaterDeepColor * (1.0 - facing);
	
	//9. 최종 반사 * fresnel + 물 색
	float3 cReflect = fresnel * vReflection.xyz;
	return float4(cReflect + waterColor, 1.0);
	
	
}


```

일단 간단하게 Bump합성 -> Uv Offset -> Refl / Refr sample -> mask 차폐 보정 ->Fresnel, 거리로 색보간

유리 시뮬레이션도 이와 같은 셰이더로 color 들어간다 : 패스

---
### 05. 결론
- 물리적인 원리에 완벽히 기반한것은 아니지만, 효율적이고 우수한 결과 제공
- 서로 다른 색상의 굴절 표면이 **겹치는 부분에서 결과는 틀어짐.**

- 해결책
  - Refraction mesh를 뒤에서부터 앞으로 정렬, Mesh 그릴 때마다 굴절 맵 갱신 (정확 하지만 비용 큼)
  - 정렬 후 **Alpha blending** 사용하는 방법도 있음 (정확도 떨어지지만 가볍다.)