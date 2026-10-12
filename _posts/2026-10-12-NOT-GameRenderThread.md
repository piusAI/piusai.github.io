---
layout: post
published: false
title: UE에서의 GT와 RT
subtitle: GameThread, RenderThread
date: 2026-10-3 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---

Unreal engine에서는 GameThread(GT)와 RenderThread(RT)를 따로 두어서 병렬적으로 처리한다.

이번에 주요한점은 Rendering에 필요한 데이터를 분리해서 안전하게 전달하는것이다.

- RT->GT로 보내는 통신은 성능상 이유로 지양하기에 이번 글에서는 RT->GT로 보내는 경우만 집중한다.
### GT와 RT의 이유
1.  로직을 담당하는 GT가 빠른 경우
2. 
그래픽처리를 담당하는 RT가 빠른 경우
이 둘 사이를 `async`하게 접근한다.

### GT->RT 전달 패턴 5가지

#### ① `UPrimitiveComponent` → `FRimitiveSceneProxy`
- mesh, Material 관련 정보, 렌더링에 필요한 parameter 등을 렌더링 전용 객체에 보관(보통 복사함)
    ex) customMesh, Terriain, `Primitive`Rendering

#### ② Render Command - 실행할 작업 전달
- GT → `ENEQUEUE_RENDER_COMMAND(...)` → RT
  Proxy를 새로 만들지 않고,  RT에서 실행할 작업을 예약하는 방식
  
```c++
ENQUEUE_RENDER_COMMAND(UpdateValue)(
[Value] (FRHICommandListImmediate& RHICmdList)
{
	// RT에서 실행할 작업
}
);
```

- `Value`는 GT에서 캡쳐한 값, 실제 코드에서 Capture한 객체와 리소스 수명 / 스레드 안정성 보장!
EX) 간단한 렌더링 상태 갱신, RT에서 실행할 작업 예약, Compute Shader Pass실행해서 RT Update시 - (GT에서 계산된 데이터를 capture해서 안전히 전달)

#### ③ Dynamic DataUpdate - 변경 데이터만 전달
- `MarkRenderDynamicDataDirty()` → `SendRenderDynamicData_Concurrnet()`계열
Proxy 전체를 재생성 하는 대신 기존 proxy에 번경된 렌더링 데이터를 갱신하는 패턴
ex) 매 프레임 변하는 parameter, 동적 지형 data


#### ⓸ GPU Resource Update - GPU 리소스 갱신
GT 측 데이터 → Rendering Resource update 경로 → GPU buffer / Texture

- 정점 데이터, instance data, texture처럼 GPU가 읽을 데이터를 업로드하는 방식
- 실제 업로드는 엔진의 리소스 관리 및 RHI 경로를 통해 이루어짐.
- ex - ComputeShader Inputbuffer, 변형 데이터, GPU instance data

#### ⑤RDG - RenderPass와 GPU 작업 구성
RT측 코드 → `FDGBuilder` → Pass 등록 → GPU `execute()`
- RDG자체는 GT→RT전달 방식은 아님.
- RT에 도착한 데이터로 GPU 작업을 구성, 리소스 의존성을 관리하는 프레임워크임
 ex) ComputeShader, UAV, RenderTarget, GPU 기반 tessellation

GT → `ENQUEUE_RENDER_COMMAND(...)` → RT

---

### GT->RT
- 렌더링에 필요한 데이터를 별도의 Proxy나 렌더링용 데이터 구조로 복사해서 전달한다.
- RT가 GT에서 관리하는 `UObject`/ `Component`데이터에 직접 접근을 막도록하기 위함

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/UEGraphics/GTRT/GTtoRT.png" alt="GTtoRT" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>GTtoRT</strong> </td>  </tr> </table>

`ENQUEUE_RENDER_COMMAND`로 RT로 보내며, Lambda로 캡쳐해서 보낸다.


``` cpp
//GameThread에서 보내는 방식
ENQUEUE_RENDER_COMMAND( class )([variable])(){ return;}

```
- 렌더링용 데이터 패킷을 구성해서 보낸다.


`Proxy`(RT)에서 받는 방식

`UpdateInteractionState_RenderThread` 와 같은 함수로 받는다

