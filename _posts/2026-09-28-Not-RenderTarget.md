---
layout: post
published: false
title: RenderTarget for Shader
subtitle: Constraint
date: 2026-09-21 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
RenderTaget은 GPU 메모리Buffer를 채우는 것이다.

이 단 한문장으로 모든 의미를 내포할수는 있다.  
하지만 "**그래서 뭔데?**"라고 한다면 점점 활용법에 맞는 고수준으로 가야한다.    

지금까지 내가 활용 해온 것만으로도 총 3가지가 있다.
1. Dx에서도 최종 BackBuffer와 SwapChain을 해서 RenderTarget으로 보내기
2. XR screen에 projection하기 위한 RenderTarget
3. RenderTarget Texture를 그려서 Shader에 보내어 수정


이번에는 3번에 집중해 보려한다.
### Main 이해
1. RT4개 역할 - PingPong 누적 매커니즘(시간 지남에 따른 누적/ 감쇠)
		RenderTarget의 역할
		ping-pong 순환구조란?
>**핑퐁(Ping-Pong) 순환 구조**: `RT_SnowHistory` + `RT_Capture` $\rightarrow$ `M_Merge` 연산 $\rightarrow$ `RT_Snow`에 출력 $\rightarrow$ 다음 프레임에서는 `RT_Snow`가 새로운 `RT_SnowHistory`가 됨.


RenderTarget은 [HeightField](https://piusai.github.io/graphics/2026/09/27/animationSandMud.html)를 저장하는 GPU 데이터 버퍼이다.


Player의 발자국을 찍어야할때, 4가지나 되는 RenderTarget을 활용한다.
`RT_Snow`, `RT_Capture`, `RT_Immediate`, `RT_SnowHistory`


### 물려있는 RT

`M_Snow` : `RT_SnowHistory`, 
`M_Merge` : `RT_Snowhistory`, `RT_interMeditate`가 들어가있음

### RenderTarget 프레임 단위 데이터 흐름
```

```
