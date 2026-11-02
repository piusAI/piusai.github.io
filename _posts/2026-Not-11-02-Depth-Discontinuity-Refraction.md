---
layout: post
published: false
title: DepthDiscontinuity-Refraction 비교 실험
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-11-02 14:22:00 +0900
description: Refraction
categories:
  - Graphics
tags:
  - Graphics
use_math: true
author: PIUS
---
Unreal Engine의 Refraction 방식과 비교하며, 최신 굴절 기술을 탐색하고, 새로운 Refraction을 조합하기 위해서 논문을 작성 했다.

---
### Architecture
- **Module** : 
- **.usf** UE RDG 아키텍쳐 활용 : Custom Depth Pass로 등록해서 usf로 굴절 쉐이더 제작
- **실패 사례 대응** : Depth Discontinuities나, 잘못된 초기 추즉핱은 실패 사례분석, 감지하고, 완화, 해결하기위한 실용적인 전략 제안한다.
- **Caustic 가속** : Refraction Ray에대한 이 접근 방식으로 Screen-space Caustic의 계산 속도를 향상 시킨다.

---
### 1. Class

#### 1.1 전체적인 UE RenderPipeline 아키텍쳐

Unreal engine의 RenderPipeline을 직접 수정하는것이 아니라, Engine이 열어둔 확장 지점에 끼어드는 방식으로 진행하였다.
`FSceneViewExtensionBase`를 상속한 클래스에서 아래 함수들을 override하면, Engine이 정해진 시점에 이 함수들을 호출해준다.

- `BeginRenderViewFamily()` : GT에서 호출, CVar를 읽어 `Cached*` 멤버에 저장
- `PrePostProcessPass_RenderThread()` : RT에서 호출, 엔진 Scene Rendering이 끝난 뒤, PostProcess가 시작되기 직전에 우리의 Pass를 추가
- `SetupViewFamily()`, `SetupView()` : 같은 확장 지점이지만 우리 구현에서는 사용 X

GT, RT사이에는 별도의 복사 커맨드 없이 `Cached*` 멤버를 공유한다 (GT가 쓰고, RT가 읽는다).  
`Enable`같은 일부 CVAR는 RT에서 `GetValueOnRenderThread()`로 직접 읽는다.

Module(Plugin) 형식으로 작업했기 떄문에, `StartupModule()`에서 가상 Shader경로를 `Shader`폴더로 Mapping해주어야 한다!

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Refraction/EngineFSceneViewExtension.png" alt="EngineFSceneViewExtension" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>FSceneViewExtension</strong> </td>  </tr> </table>
Engine이 넘겨준 RDG Graph에 우리의 Pass를 추가해서 SceneColor를 갱신하는 방식으로 진행하였다.

- 내가 `FRDGBuilder`를 만든게 아니라 Engine이 이미 생성해서 넘겨준 `GraphBuilder`에 Pass를 추가하는 경우, 직접 `Execute()`를 호출하면 안된다. 그 Graph의 소유자인 Engine이 나중에 알아서 `Execute`한다.
- `AddFullscreenPass`나 `AddCopyTexturePass`를 호출하면 Pass가 그 Graph에 기록만 되고, Engine이 Frame Rendering 마무리하면서 `GraphBuilder.Execute()` 한번 호출할때 같이 올라간 우리 Pass도 함께 Compile되고 실행된다.
- RDG는 결과를 아무도 읽지 않는 Pass를 자동으로 Cull(제거)한다. 그래서 Method Pixel Shader의 출력은 Composit Pass( Capture 복사)가 읽어야 살아남는다.

- 이 확장 클래스는 `UWorldSubsystem`의 `Initialize()`에서 `FSceneViewExtensions::NewExtension<>()`으로 생성한다. Subsystem의 Setter(`SetMethod`, `SetIOR` 등)는 멤버를 바꾸면서 해당 CVar도 함께 갱신하고, 렌더링 시점에는 이 CVar 값이 `BeginRenderViewFamily()`에서 읽힌다.


##### `NewExtension()`이란?

```cpp
SceneViewExtension = FSceneViewExtensions::NewExtension<FRefractSceneViewExtension>(GetWorld(), this);
```
- **생성과 등록**을 한방에 한다.
  Engine의 `SceneViewExtension`목록에 올려주어야하는데, Class 생성자 `FAutoRegister`가 이 등록을 위한 인자이다. 그래서 일반`New` / `MakeShared`로 만들면 엔진이 모른다.
- 엔진은 **약한 참조**만 갖는다.
  목록은 weakPointer로 들고있기에, 계속 객체를 살려두는 것은 Subsystem의 `SceneViewExtension` 멤버(`TSharedPtr`)이다. `Deinitilaize()`에서 `Reset()`을 하면 객체가 사라지고, Hook 호출도 끝난다.
- **World 단위**로 동작한다
  `FWorldSceneViewExtension`이라, `GetWorld()`로 넘긴 World 뷰에서만 훅이 호출된다.



#### 1.2. Data Structure

| 파일                                                        | 현재 책임                                   |
| --------------------------------------------------------- | --------------------------------------- |
| `Source/RefractMethod/Private/RefractMethod.cpp`          | `/Plugin/RefractMethod` 가상 셰이더 경로 등록    |
| `Source/RefractMethod/Private/RefractMethodSubsystem.cpp` | 월드별 View Extension 생성, Blueprint 설정 진입점 |
