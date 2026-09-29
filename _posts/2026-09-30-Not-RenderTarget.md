---
layout: post
published: false
title: RenderTarget for Shader
subtitle: Constraint
date: 2026-09-30 19:37:00 +0900
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

Houdini에서는 WetMap이라고 표현하는 것은 CPU에서 처리하기때문에 실시간성이 없고,
`SopSolver`를 활용한 houdini 작업은 항상 swapchain이슈가 많이있어 0Frame으로 꼭 돌려야지 되는 Determine simulation이다.  

그래픽스의 내가 생각하는 꽃은 최적화와 맞물리는 아트, 기술을 만난 예술이다.
### RenderTarget가 필요한 이유

RenderTarget은 [HeightField](https://piusai.github.io/graphics/2026/09/27/animationSandMud.html)를 저장하는 GPU 데이터 버퍼이다.
Texture2D로 이미지로써 보이지만 여기에서의 정확한 역할은 플레이어의 정확한 위치를 받아서 Shader에서 호환하기 위함에 있다.  


### RenderTarget 네 가지 Keyword

`FootPrint` / `Brush` : 
```
0 0 0 0 0
0 1 1 1 0
0 1 1 1 0
0 1 1 1 0
0 0 0 0 0
```
와 같이 밟았음을 Heightfield에 저장해야하는데 이를 `RT_Capture`로 처리한다.


`Accumulation` : `RT_SnowHistory`가 필요하겠다

이번 RenderTarget은 [DeformableSnowSystem](https://www.fab.com/listings/e5fbb0e8-d234-414d-9599-b8d65e6d0517?lang=en)의 RenderTarget 4가지에 집중해보려 한다.
### RenderTarget의 이해
1. RT4개 역할 - PingPong 누적 매커니즘(시간 지남에 따른 누적/ 감쇠)
		RenderTarget의 역할
		ping-pong 순환구조란?




Player의 발자국을 찍어야할때, 4가지나 되는 RenderTarget을 활용한다.
`RT_Snow`, `RT_Capture`, `RT_Immediate`, `RT_SnowHistory`


### 물려있는 RT

`BP_SnowGenerator`

`M_Snow` : `RT_SnowHistory`, 
`M_Merge` : `RT_Snowhistory`, `RT_interMeditate`가 들어가있음

### RenderTarget 매 프레임 단위 데이터 흐름

>**핑퐁(Ping-Pong) 순환 구조**: `RT_SnowHistory` + `RT_Capture` $\rightarrow$ `M_Merge` 연산 $\rightarrow$ `RT_Snow`에 출력 $\rightarrow$ 다음 프레임에서는 `RT_Snow`가 새로운 `RT_SnowHistory`가 됨.


```

```



#### Future Study
`RWTexture2D<float>`과 같은 것을 활용을 하면 UAV로 병렬 처리 가능

---
## Reference
SnowDeformable : https://illu.tistory.com/1496#footnote_link_1496_4
Deformable Snow System : https://www.fab.com/listings/e5fbb0e8-d234-414d-9599-b8d65e6d0517?lang=en

