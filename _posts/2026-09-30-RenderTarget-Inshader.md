---
layout: post
published: true
title: Snow Deformation ①  - RenderTarget 4개로 만드는 RealTime Deformation
subtitle: Constraint
date: 2026-09-30 15:50:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
RenderTaget은 GPU가 직접 그림을 그려 넣을 수 있는 Texture Buffer(GPU 메모리)이다.

위 한 문장에 모든 의미가 들어가 있지만, "**그래서 뭔데?**"에 답하려면 용도별로 내려가야한다.  

참고 : [BP로 RT Texture 생성하기](https://dev.epicgames.com/documentation/unreal-engine/creating-textures-using-blueprints-and-render-targets?application_version=4.27)

지금까지 RenderTarget(이하 RT)를 써 본 용도는 4가지다.
1. Dx에서 최종 BackBuffer와 SwapChain을 거쳐 RT으로 보내기
2. RT로 Texture 생성하기
3. XR screen에 projection하기 위한 RT
4. **RT에 그린 Texture를 Shader로 보내 Mesh Deformation에 쓰기**


이번에는 4번에 집중해 보려한다.

---

### 1. 오프라인 렌더링의 Interaction 흐름과 RT가 필요한 이유
오프라인 (Houdini)에서 발자국과 같은 VTX 이동을 어떤식으로 표현할까?
대략적인 흐름은 아래와 같다
```
1. 발(표면과 interaction할 영역) VDB volume 생성
2. Ground nearpoint 탐색
3. @P.y -= @dist; //displace
```
여기에 추가로 **누적(Accumulation)** 이 필요하면 Houdini에서는 `WetMap`을 만들어야한다.  
이는 `SopSolver`단계에서 Simulaiton한다.  
이 단계에서 항상 swapchain 이슈가 많아서

1. 0 Frame으로 꼭 돌려야 하거나
2. Sop Level 왔다 갔다 해야 하거나
3. SceneView를 다시 열어야 한다.

게다가 이 시뮬레이션은 이전 프레임에 의존하는 *Determine simulation*이고,  
CPU에서 처리하기때문에 **실시간성이 없**다.  
그래픽스의 꽃은 역시 최적화인데, 이걸 실시간에서 가능하다는것이 경이로웠다.

그렇다면 실시간 엔진에서는 어떻게할까? 답이 **RT**다.

다시 한번 강조하자면 RenderTarget은 [HeightField](https://piusai.github.io/graphics/2026/09/27/animationSandMud.html)를 저장하는 GPU 데이터 버퍼이다.  
겉보기엔 그냥 Texture2D로 이미지로 보이지만 틀린 말도 아니다.  
이 Deformation 에서의 RT 역할은 다음과 같다.
> (Interaction할)플레이어의 정확한 World Location / Geometry를 받아서 Shader에서 쓸 수있도록 하는것

RT의 크기와 정밀도는 제한되어 있으므로, **RT가 플레이어와 함께 움직이도록** 만든다.

---
### 2. 현재 포스팅내 실험 구성 :
원본  [DeformableSnowSystem](https://www.fab.com/listings/e5fbb0e8-d234-414d-9599-b8d65e6d0517?lang=en)의 BP를 분석하며 재구성 하였다.

| 원본                        | 재구현                                  |
| ------------------------- | ------------------------------------ |
| `BP_SnowGenerator`        | `SnowGenerator.cpp` (CPP Conversion) |
| `BP_CustomTraceComponent` | `HJCS_CustomTraceComponent` (BP)     |
| `BP_FootprintComponent`   | `HJCJ_FootTraceComponent` (BP)       |

먼저 최소 수준, `SnowGenerator`에서의 RT와 `BP_CustomTraceComponent`에서의 RT만 뜯어 이해!
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/DeformationSnow.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> DeformationSnow</strong> </td>  </tr> </table>
최종 Mateiral의 결과 화면이다.

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/OnlyRenderTarget.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> RenderTarget - noMaterial</strong> </td>  </tr> </table>
하지만 위와같이 Material까지 연동하지 않고 RenderTarget 흐름에만 먼저 집중해보자.

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/OnlyMaterial.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Only RenderTarget - Emission</strong> </td>  </tr> </table>
지금 RenderTarget위치를 Screen Position과 UV를 기준으로 Step 처리했기 때문에,  
월드의 정확한 위치가 아닌 Texture (0~1) 정규화된 좌표로 Texture가 나온다. 

---

### 3. Render Target 4개

#### 3.1 한눈에 보기

| RT                | 크기 / 포맷             | 역할                                               | 읽는(R) 곳 / 쓰는(W) 곳                                              |
| ----------------- | ------------------- | ------------------------------------------------ | -------------------------------------------------------------- |
| `RT_Capture`      | 256×256,  R16f      | `E`키 누른 이후, **Top View**를 캡쳐한 Render Texture     | `BP_CustomTraceComponent`(R)                                   |
| `RT_Intermediate` | 2048×2048, RGBA16f  | **이번 프레임** 발자국 / CustomTrace를 Canvas로 그려두는 중간 RT | `BP_SnowGenerator`(W),`M_Merge`(R)                             |
| `RT_Snow`         | 2048×2048, RGBA 16f | **현재** 눈 변형 상태. 머티리얼이 읽는 최종 결과                   | `BP_SnowGenerator`(W), `M_SnowSideHeap`(R), `M_SnowDebug`(R)   |
| `RT_SnowHistory`  | 2048×2048, RGBA 16f | **직전** 상태. 다음 프레임 합성의 기준 (Accumulation)          | `BP_SnowGenerator`(W), `M_SnowSideHeap`(R), `M_Snow`(Final R), |

**비유**:
- `RT_Capture` = 이번에 찍은 도장 하나
- `RT_Intermediate` = 도장을 종이에 찍기 전 작업대
- `RT_Snow` = 지금 종이(눈)에 찍혀 있는 상태
- `RT_SnowHistory` = 방금 전 종이의 사본

발자국 (`FootPrint` / `Brush` )은 `M_Footprint`(depth, Hardness)를 Canvas에 직접 Stamp하고 `RT_Capture`는 CustomTrace(액터 실루엣)에만 쓰인다.
```
0 0 0 0 0
0 1 1 1 0
0 1 1 1 0
0 1 1 1 0
0 0 0 0 0
```
와 같이 밟았음을 `RT_Intermediate`에 Stamp된다.

#### 3.2 Material 연결

| Material         | 읽는 RT                               | 하는 일                                                              |
| ---------------- | ----------------------------------- | ----------------------------------------------------------------- |
| `M_Merge`        | `RT_SnowHistory`, `RT_Intermediate` | 둘을 합성해 `RT_Snow`에 Draw. `SetSnowAttenuation` 파라미터로 History 감쇠량 조절 |
| `M_SnowSideHeap` | `RT_Snow`                           | `RT_SnowHistory`로 Draw (눈이 눌린 주변으로 쌓이는 Heap 처리 + History 갱신)      |
| `M_Snow`         | `RT_SnowHistory`                    | `MF_DeformationProcess`의 In Texture로 사용해 실제 메시 변형                 |
| `M_SnowDebug`    | `RT_Snow`                           | Emissive로 출력하는 디버그용                                               |
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_Merge.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> M_Merge </strong> </td>  </tr> </table>

#### 3.3 Material Instance Dynamic 연결

| MID                | UMaterialInterface* Parent | Paramter                          | 그리는 방식 / 대상                                                          |
| ------------------ | -------------------------- | --------------------------------- | -------------------------------------------------------------------- |
| `M_MergeMID`       | `M_Merge`                  | Attenuation, Offset               | `DrawMaterialToRenderTarget`→ `RT_Snow`                              |
| `M_FootprintMID`   | `M_Footprint`              | Depth, Hardness                   | `Canvas`→`K2_DrawMaterial` → `RT_Intermediate` (`DrawFootprint()`)   |
| `M_CustomTrailMID` | `M_CustomTrail`            | CaptureRT, Hardness, Compensation | `Canvas`→`K2_DrawMaterial` → `RT_Intermediate` (`DrawCustomTrace()`) |

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/MF_OffsetUV.png"  alt="ShouldWorkOff" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>MF_OffsetUV</strong> </td> <td width="37%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/M_Merge.png" alt="M_Merge.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>M_Merge</strong> </td> </tr> </table>

#### 3.4 RT가 플레이어를 따라다니는 방법
RT는 World 전체가 아닌, Player주변 `SnowFieldSize`만큼만 듶는다

``` cpp
//UpdatePos() - 매 Tick
OldPos = Pos;
Pos = PixelAlign(PlayerPosition);
DeltaOffset = Pos - OldPos;

UKismetMaterialLibrary::SetVectorParameterValue(
		this, MPC_Snow, FName("PivotAndSize"),
		FLinearColor(Pos.X, Pos.Y, SnowFieldSize.X, SnowFieldSizes.Y)
	);

//PixelAlign - RT 1pixel 단위로 위치 Floor(필드가 Pixel단위로만 이동)
res = RTResolution / SnowFieldSize;  //World 1unit당 픽셀수
Aligned = floor(pos*res) / rest;

//WorldposToUV - World>RT UV(Player RT가 중앙 0.5)
UV = (WorldXY - Pos) / SnowFieldSize +0.5;

//MergeRT - History를 이동량만큼 UV로 밀어서 이전 자국이 World에 고정된것처럼
Offset = DeltaOffset / SnowFieldSize;

```
- `PixelAlign` : 이동량이 정확히 Pixel 정수배가 되어, History를 밀때 Sampling Blur나 흔들림이 생기지 않는다.
- MPC_Snow : `PivotAndSize`, `SnowHeight`를 Material쪽에서 Word->RT UV 변환에 쓸 수 있도록 전달. 



---

### 4. RT별 흐름
##### 4.1 RT_Capture

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/DeformationSnow/ShouldWorkOff.png"  alt="ShouldWorkOff" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ShouldWorkOff</strong> </td> <td width="25%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/DeformationSnow/ShouldWorkOn.png" alt="ShouldWorkOn.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ShouldWorkOn</strong> </td> </tr> </table>

왼쪽과 같이 `ShouldWork`멤버 변수 초기값: Off로 두어야 `E`로 죽었을때 RT가 On되며 올라옴
오른쪽과 같이 켜두면 계속 Topview에서 RT 찍음

- `CustomTraceComponent`:: **Begin Play** :  
	Actor 위치를 받아 저장해두고  
- CustomTraceComponent :: **Tick** :  
	`Should Work`가 On일때만 `LineTraceForObject`로 AddCustom Trace로 받아 구조체에 담고, `SnowGenerator::CursomTrace` 함수 호출하여 `CustomTrace` 배열에 넣는다.
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/CustomTrace.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> CustomTrace </strong> </td>  </tr> </table>

실제 캡쳐는 `CustomTraceComponent`가 아니라 `SnowGenerator::SceneCaptureForEachActor()`가 한다. *4-3에 이어서 설명*

##### 4.2 공통 초기화 (RT_Snow, RT_SnowHistory, Intermediate)
- `SnowGenerator`의 **BeginPlay/EndPlay** 에서 `ClearRenderTarget2D`로 아래 세RT를 비운다
- BeginPlay는 시 작전 Asset 유효성 검사를 하고, 하나라도 없다면 Tick을 끈다.  
	통과시 MID 3개 생성 → RT Clear -> `UpdatePos()` → MPC `SnowHeight`설정 → `bSnowGeneratorReady = true`순

##### 4.3 RT_Intermediate
Foot, CustomTrace를 그린다
`SnowGenerator`::**Tick** 에서 `DrawCurrentAllFootPrint()`, `DrawCurrentAllCurrentTrace()`를 호출.

**DrawCurrentAllFootPrint**, **DrawCurrentAllCustomTrace**에서 Canvas에 그린다.  
발자국, CustomTrace가 여기서 그려진다.
```cpp
//DrawCurrentAllFootprint, DrawCurrentAllCustomTrace
UKismetRenderingLibrary::BeginDrawCanvasToRenderTarget(this, RT_Intermediate, Canvas, inSize, Context);
...
// Canvas에
UKismetRenderingLibrary::EndDrawCanvasToRenderTarget(this, Context);
```

###### Footprint
- `RT_Intermediate`를 먼저 Clear. (Footprint 0개라도 Clear 매프레임)
- 각 발자국 `WorldPosToUV`→`ComputeScreenPosandSize(UV, 2.0, ContactRadius)`로 RT 픽셀 위치 / 크기 수정. 크기 `2 x ContactRadius`  world unit → pixel.
- `Depth = max((SnowHeight-Distance) / SnowHeight, 0)` : 발이 지면에 가까울수록(Distance↓) 깊게 눌린다
- `M_FootprintMID(Depth, Hardness)`를 `Rotation`과 함께 `K2_DrawMaterial`로 Stamp한다.

###### CustomTrace(Clear 없이 Footprint 위에 이어 그림)
- Trace마다 `SceneCaptureForEachActor`로 `RT_Capture`에 캡쳐
	- `CaptureRT` Clear →하나의 `SceneCapture`Component를 `CaptureLocation`으로 옮김
	-  `OrthoWidth = RT_Size`, `ShowOnlyActors = Trace.ActorToDraw` (해당 액터만 촬영)
	- `CaptureSource = SCS_BaseColor` (**그림자가 안 찍히게** 하는 설정)
	- `CaptureScene()`(`bCaptureEveryFrame = false`라서 수동 호출)
- Capture 영역(World `RT_Size`)과 Stamp 크기(`ComputeScreenPosandSize(UV, 1.0, RT_Size))`가 일치해 픽셀 1:1대응
- `DrawCustomTrace` : `K2_DrawMaterial(M_CustomTrailMID)`이후 `K2_DrawTexture(CaptureRT, v반전)`을 같은 위치에 그림

##### 4.4 RT_Snow - 합성 결과
`SnowGenerator`::**Tick** 에서 `MergeRT()`를 호출.
```cpp
//ASnowGenerator:MergeRT()
UKismetRenderingLibrary::ClearRenderTarget(this, RT_Snow);
UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_Snow, M_MergeMID);
}
```
- `RT_Snow`에서는 Clear를 한번 해주고, `M_MergeMID`로 다시 그린다(DrawMaterialToRT).

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_MergeRT.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> M_Merge내부의 RT </strong> </td>  </tr> </table>
- `M_MergeMID` <- `M_Merge` ![](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)⊃ `RT_History` + `RT_InterMediate`로 구성되어있음
 - `Attenuation = clamp(DeltaTime * TrailAttenuation, 0,1)`은 `SetSnowAttenuation`내 수식이다.
   dt에 비례하므로, FrameRate와 관계없이 감쇠 속도 일정

##### 4.5 RT_Snowhistory - Accumulate(누적)
`RT_SnowHistory`는 한 Tick에서 **두 번** 쓰인다.
(1) Tick 시작 : `CopyToHistory()` - 합성 입력 준비
``` cpp

//ASnowGenerator::CopyToHistory
UKismetRenderingLibrary::BeginDrawCanvasToRenderTarget(this, RT_SnowHistory, Canvas, ScreenSize, Context);
...
Canvas->K2_DrawTexture(RT_Snow, FVector2D::ZeroVector, ScreenSize, FVector2D::ZeroVector,FVector2D::UnitVector, FLinearColor::White, BLEND_Opaque, 0.f, FVector2D(0.5f, 0.5f));
```

- Clear 직후 **직전 프레임**의 `RT_Snow`를 그대로 복사한다.
  -> Clear하더라도 누적 사라지지 않음
`snowGenerator`::**CopyToHistory**에서 `RT_Snow`의 Texture를 `RT_SnowHistory`에 그려줌


(2) Tick 마지막 : `DrawMaterialtoRT`를 호출.
```cpp
//ASnowGenerator::Tick
UKismetRenderingLibrary::DrawMaterialToRenderTarget(this, RT_SnowHistory, M_SnowSideHeap);

```
- `M_SnowSideHeap`을 거쳐 `RT_SnowHistory`로 복사한다

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/M_Snow.png" alt="DeformationSnow" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> RT_SnowHistory in M_Snow</strong> </td>  </tr> </table>

- `M_Snow`는 `MF_DeformationProcess`의 inTexture로 `RT_SnowHistory`를 사용
- 다음 프레임의 (1)이 `RT_Snow`를 raw로 다시 복사하므로, **누적 경로에는 SideHeap**결과가 되 먹임되지 않음. SideHeap은 표시용.



---

### 5. 프레임 단위 데이터 흐름
`SnowGenerator::Tick` 코드 순서

```
[BeginPlay] 에셋 검증 → MID 3개 생성 → RT 3개 Clear → UpdatePos → MPC.SnowHeight 설정

(별도 Tick) Footprint/CustomTrace 컴포넌트가 AddFootPrintHJ / AddCustomTraceHJ로 배열에 요청을 쌓음

[Tick]
① SetSnowAttenuation(dt)
     M_MergeMID.Attenuation = clamp(dt × TrailAttenuation, 0, 1)
② UpdatePos()
     Pos = PixelAlign(플레이어 위치), DeltaOffset = Pos - OldPos
     MPC_Snow.PivotAndSize = (Pos.xy, SnowFieldSize.xy)
③ CopyToHistory()
     Clear(RT_SnowHistory) → RT_Snow(직전 프레임 결과)를 RT_SnowHistory에 복사
④ DrawCurrentAllFootPrint()
     Clear(RT_Intermediate) → Footprints 각각을 M_FootprintMID로 stamp → Footprints.Reset()
⑤ DrawCurrentAllCustomTrace()          (Clear 없음, ④ 위에 이어서 그림)
     Trace마다: SceneCaptureForEachActor → RT_Capture,
                DrawCustomTrace → RT_Intermediate  → CustomTrace.Reset()
⑥ MergeRT()
     Clear(RT_Snow) → M_MergeMID.Offset = DeltaOffset / SnowFieldSize
     → M_Merge(RT_SnowHistory + RT_Intermediate) → RT_Snow
⑦ DrawMaterialToRenderTarget(RT_SnowHistory, M_SnowSideHeap)
     RT_Snow → SideHeap 처리 → RT_SnowHistory에 덮어씀

[렌더링] M_Snow가 RT_SnowHistory(⑦ 결과)를 읽어 메시 변형
```


-  **③이 ⑥보다 먼저**인 이유 : ⑥에서 `RT_Snow`를 Clear하고 다시 그리기 때문에, 이전 상태를 읽으려면 그 전에 `RT_SnowHistory`로 빼 두어야 한다.
- **RT_Snow는 Tick 안에서 한 번 Clear → 한 번 Draw**되고, ③에서 읽히는 시점은 "직전 프레임의 완성본"이다.
- Footprint, CustomTrace는 처리 후 각 배열을 `Reset()`하므로 **요청이 프레임 단위로 소비**된다.
- **고정 비용** : 2048² 규모의 패스가 매 프레임 ③ 복사, ④ Clear, ⑥ 합성, ⑦ SideHeap으로 4번 돌고, CustomTrace 개수만큼 SceneCapture가 추가된다.


---
### 6. RT가 왜 4개나 필요한가 : Ping-Pong 누적
핵심 제약은 **GPU는 같은 RT를 동시에 읽고 쓸 수 없다**는 것이다.  
이전 상태를 읽으면서 같은 RT에 새 결과를 쓸 수 없으므로 "읽는 용도"와 "쓰는 용도"를 분리한다.

> Ping-Pong 순환 : `RT_SnowHistory`(읽기) + `RT_Intermediate`(새자국)
> `M_Merge` → `RT_Snow`(쓰기) → 프레임 끝에 `RT_Snow`를 `RT_SnowHistory`에 복사 (`CopytoHistory()` 역할) → 다음 프레임의 읽기 소스

- 두 RT를 Pointer로 `Swap`하는 Ping-pong이 아니라, 매 프레임 **복사**하는 방식.
	복사 비용이 발생함.
- 누적 : History가 이전 자국을 기억하므로 발자국이 남는다.
- 감쇠 : 오래된 자국은 `Attenuation`으로 줄어든다.
- 이동 : 필드가 player를 따라가므로 history는 `offset`만큼 UV 이동해서 읽는다
- 분리 이유 :
  `RT_Capture`- 256²의 SceneCapture  
  `RT_Intermediate`- World Coordinate로 배치된 현재 프레임 자국
  `RT_Snow/History` - 플레이어 주변 전체 필드


---

### 7. cpp 메모 - 구현하며 생긴 의문
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/DeformationSnow/Outparameter.png" alt="의문1" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> RT_SnowHistory in M_Snow</strong> </td>  </tr> </table>
- `BeginDrawCanvasToRenderTarget(this, RT, Canvas, Size, Context)`의 뒤 세 인자는 전부 `OutParamter`이다. `UCanvas*&는` 함수가 Pointer자체를 채워넣어야해서 "포인터의 참조".

- `FDrawToRenderTargetContext`는 Being이 채운 상태를 End가 다시 받아 정리하는 구조체라, RValue가 아니라 **같은 객체(Lvalue)**를 Begin/End에 넘겨야한다.
- `TArray<AActor*>&`는 배열 복사하지않고 배열 자체를 참조로 넘기는 것이다.  
  배열안의 포인터는 그대로이다.

--- 

#### Future Study
- `RWTexture2D<float>`과 같은 UAV를 활용하면 ComputeShader로 병렬 처리할 수있다.
- RT가 World 어느 영역을 덮는지
- HeightField 변경 영역에만 Tessellation / Subdivision 적용 → 논문의 아이디어
- 제안하는 Concept중 HeightField가 실제 바뀌는 부분에만 Tessellation / Subdivision을 활용한다 → 논문의 아이디어


---
## Reference
- [SnowDeformable](https://illu.tistory.com/1496#footnote_link_1496_4)
- [Deformable Snow System](https://www.fab.com/listings/e5fbb0e8-d234-414d-9599-b8d65e6d0517?lang=en)

