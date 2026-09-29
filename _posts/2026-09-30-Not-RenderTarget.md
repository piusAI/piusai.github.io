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
[BP로 RT Texture 생성하기](https://dev.epicgames.com/documentation/unreal-engine/creating-textures-using-blueprints-and-render-targets?application_version=4.27)

지금까지 내가 활용 해온 것만으로도 총 3가지가 있다.
1. Dx에서도 최종 BackBuffer와 SwapChain을 해서 RenderTarget(이하 RT)으로 보내기
2. RT로 Texture 생성하기
3. XR screen에 projection하기 위한 RenderTarget
4. RenderTarget Texture를 그려서 Shader에 보내어 수정

이번에는 4번에 집중해 보려한다.

### 오프라인 렌더링의 Interaction 흐름과 RT가 필요한 이유
오프라인 렌더링에서 발자국과 같은 VTX 이동을 어떤식으로 표현할까?
대략적인 흐름은 아래와 같다
```
1. 발(표면과 interaction할 영역) VDB volume 생성
2. Ground nearpoint 
3. @P.y -= @dist; //displace
```
이 정도 흐름을 생각할 수있을것이다.  
거기에 추가로 Accumulation을 해야한다하면 Houdini에서는 `WetMap`이라 표현하는데,  
이는 `SopSolver`단계에서 Simulaiton한다.  
항상 swapchain 이슈가 많이있어  

1. 0Frame으로 꼭 돌려야하거나
2. SopLevel왔다갔다 해야하거나
3. SceneView를 다시 열어줘야한다.

이 마저도 당연히 *Determine simulation*이고, CPU에서 처리하기때문에 **실시간성이 없**다.  
그래픽스의 꽃은 역시 최적화인데, 실시간에서 가능하다는것이 경이로웠다.

이러한 작업을 실시간이 가능한 엔진으로 가져온다면 어떻게 해야할까? 그 정답은 RT에 있다.

다시 한번 강조하자면 RenderTarget은 [HeightField](https://piusai.github.io/graphics/2026/09/27/animationSandMud.html)를 저장하는 GPU 데이터 버퍼이다.
그냥 간단하게 Texture2D로 이미지로써 보이지만 틀린 말도 아니다.  
이 Deformation HeightField와 같은 작업에서의 정확한 역할은  
(Interaction할)플레이어의 정확한 위치를 받아서 Shader에서 호환하기 위함에 있다.  


**현재 실험** :
`BP_SnowGenerator` : `SnowGenerator.cpp` CPP Conversion
`BP_CustomTraceComponent` : `HJCS_CustomTraceComponent` (BP)
`BP_FootprintComponent`: `HJCJ_FootTraceComponent`  (BP)
로 각각 재구현함


### 현재 물려있는 RT

`BP_SnowGenerator` : `RT_Snow`, `RT_SnowHistory`, `RT_Intermediate`
`BP_CustomTraceComponent` : `RT_Capture`
`M_Snow` : `RT_SnowHistory`, 
`M_Merge` : `RT_Snowhistory`, `RT_interMeditate`
`M_SnowSideHeap` : `RT_Snow`
`M_SnowDebug` : `RT_Snow`


제안하는 Concept:
HeightField만 변경되어야 하는 부분만 Tessellation Subdivision을 활용한다

### 최소 수준만 구현하는 RT
SnowGenerator에서의 RT, BP_CustomTraceComponent에서의 RT만을 먼저 뜯어 이해  
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/DeformationSnow.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> DeformationSnow</strong> </td>  </tr> </table>


<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/OnlyRenderTarget.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> RenderTarget - noMaterial</strong> </td>  </tr> </table>
Material까지 연동하지 않고 RenderTarget 흐름에만 먼저 집중해보자.


<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/OnlyMaterial.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Only RenderTarget </strong> </td>  </tr> </table>
지금 RenderTarget위치를 Screen Position과 UV를 기준으로 Step으로 나누어놓았기때문에 정확한 위치가 아닌 Texture (0~1)정규화되도록 나온다. 각각

### RenderTarget 네 가지 Keyword

##### RT_Snow - 현재 눈상태 (2k)
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


## 4. Render Target 4개

| RT                | 크기 / 포맷            | 역할                                           |
| ----------------- | ------------------ | -------------------------------------------- |
| `RT_Capture` ♬    | 256×256, R16f      | `E`키 누른 이후, **Top View**를 캡쳐한 Render Texture |
| `RT_Intermediate` | 2048×2048, RGBA16f | 합성·가공의 중간 작업장                                |
| `RT_Snow`         | 2048×2048          | **현재** 눈 변형 상태. 머티리얼이 읽는 최종 결과               |
| `RT_SnowHistory`  | 2048×2048          | **직전** 상태. 다음 프레임 합성의 기준                     |
##### RT_Capture

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/ShouldWorkOff.png"  alt="ShouldWorkOff" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ShouldWorkOff</strong> </td> <td width="25%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/ShouldWorkOn.png" alt="ShouldWorkOn.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ShouldWorkOn</strong> </td> </tr> </table>

왼쪽과 같이 RT_capture의 `ShouldWork` 멤버 초기화를 Off로 해야지 E로 죽었을때 RenderTarget이 올라옴

(Begin play)
`SnowGenerator`에서의 Actor 위치를 받아 저장해두고  
(Tick)
`Should Work`가 on일때만 `LineTraceForObject`로 AddCustom Trace로 받아 구조체에 넣어서 `SnowGenerator::CursomTrace` 함수 호출하여 `CustomTrace` 배열에 넣는다.

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/CustomTrace.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> CustomTrace </strong> </td>  </tr> </table>



**비유**:
- `RT_Capture` = 이번에 찍은 도장 하나
- `RT_Intermediate` = 도장을 종이에 찍기 전 작업대
- `RT_Snow` = 지금 종이(눈)에 찍혀 있는 상태
- `RT_SnowHistory` = 방금 전 종이의 사본

### 4.1 프레임 단위 데이터 흐름 (추정 포함)

```
① 요청 도착      FCustomTrace 목록이 Generator에 쌓임
② 캡처          SceneCaptureForEachActor → RT_Capture (자국 모양)
③ 합성 준비      RT_Snow를 Clear (MergeRT)
④ 합성          M_Merge로 [RT_Capture + RT_SnowHistory(+감쇠)] 를 그림
                  → RT_Intermediate 를 거칠 수 있음 🔶
⑤ 결과 확정      RT_Snow 갱신
⑥ 기록          CopyToHistory: RT_Snow → RT_SnowHistory (다음 프레임용)
⑦ 표시          M_Snow가 RT_Snow를 읽어 변형
```

- "시간이 지나면 눈이 복원된다"는 효과는 ④에서 History 값에 **감쇠**(`SetSnowAttenuation`)를 곱해 조금씩 약하게 만드는 방식으로 구현됐을 가능성이 높다. 🔶
- ③에서 `RT_Snow`를 지우고 ④에서 History + 새 자국으로 다시 채우는 **"지우고 다시 그리기"** 패턴이다. Clear 자체는 합성이 아니므로, 실제 동작은 **그 뒤에 어떤 텍스처를 그리느냐**가 결정한다. ❓


### RenderTarget의 이해
1. RT4개 역할 - PingPong 누적 매커니즘(시간 지남에 따른 누적/ 감쇠)
		RenderTarget의 역할
		ping-pong 순환구조란?




Player의 발자국을 찍어야할때, 4가지나 되는 RenderTarget을 활용한다.
`RT_Snow`, `RT_Capture`, `RT_Immediate`, `RT_SnowHistory`



### RenderTarget 매 프레임 단위 데이터 흐름

>**핑퐁(Ping-Pong) 순환 구조**: `RT_SnowHistory` + `RT_Capture` $\rightarrow$ `M_Merge` 연산 $\rightarrow$ `RT_Snow`에 출력 $\rightarrow$ 다음 프레임에서는 `RT_Snow`가 새로운 `RT_SnowHistory`가 됨.






#### Future Study
`RWTexture2D<float>`과 같은 것을 활용을 하면 UAV로 병렬 처리 가능

---
## Reference
SnowDeformable : https://illu.tistory.com/1496#footnote_link_1496_4
Deformable Snow System : https://www.fab.com/listings/e5fbb0e8-d234-414d-9599-b8d65e6d0517?lang=en

