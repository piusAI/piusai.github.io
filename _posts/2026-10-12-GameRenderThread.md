---
layout: post
published: true
title: GameThread → RenderThread 데이터 전달
subtitle: GameThread, RenderThread
date: 2026-10-12 22:24:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---

UE는 **GameThread(GT)** 와 **RenderThread(RT)** 를 따로 두어서 병렬로 처리한다.  
이글의 핵심은 Rendering에 필요한 데이터를 안전하게 분리해서 RT로 전달하는 방법이다.

> `RT->GT` 방향 통신은 성능상 이유로 지양하므로, 이 글은 **GT->RT** 방향만 다룬다.

---

### 1. GT와 RT를 왜 나누는가

| 상황               | 설명                                      |
| ---------------- | --------------------------------------- |
| GT(로직)가 더 빠른 경우  | GT가 다음 프레임 로직을 먼저 진행하고, RT는 이전 프레임을 렌더링 |
| RT(그래픽)가 더 빠른 경우 | RT는 GT가 넘겨줄 데이터를 기다리지 않고 자기 작업을 처리      |

두 스레드가 서로를 기다리지 않도록 **비동기(async)** 로 동작시켜 처리량을 높이는 것이 목적이다.

---

### 2. 왜 "복사"해서 전달하는가 (GT-> RT 전달 이유)

- RT가 GT에서 관리하는 `UObject`/ `Component`데이터에 **직접 접근을 막도록**하기 위함
- 두 thread가 같은 데이터를 동시에 읽고 쓰면 **Race Condition** 발생 -> Lock 없이 안전하게 하려면 **소유권을 분리**해야함
- `UObject`는 GC대상이라 GT에서 수명이 끝날 수 있음 ->RT가 포인터를 직접 들고있으면 **위험**
- 그래서 Rendering에 필요한 데이터만 **별도의 Proxy / Rendering 용 데이터 구조로 복사**하고, 전달은 `RENDER_COMMAND_QUEUE`로 한다

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/UEGraphics/GTRT/GTtoRT.png" alt="GTtoRT" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>GTtoRT</strong> </td>  </tr> </table>


### 3. GT->RT 전달 패턴 5가지

#### ① `UPrimitiveComponent` → `FPrimitiveSceneProxy`
- Mesh, Material 정보, 렌더링 파라미터 등을 **렌더링 전용 객체(Proxy)** 에 복사해서 보관
- Data를 서로 다른 Thread가 각자 소유하는 **별도 구조체** 에 저장하는 방식이라, 소유권이 더욱 명확함
- 예시 :  CustomMesh, Terriain, Primitive Rendering


#### ② Render Command - 실행할 작업 전달
- GT → `ENEQUEUE_RENDER_COMMAND(...)` → RT
- Proxy를 새로 만들지 않고,  **RT에서 실행할 작업을 예약**하는 방식
  
``` cpp
ENQUEUE_RENDER_COMMAND(UpdateValue)(
[Value] (FRHICommandListImmediate& RHICmdList)
{
	// RT에서 실행할 작업
}
);
```

- `Value`는 GT에서 **값으로 캡쳐**한 데이터. 캡쳐한 객체/Resource의 수명과 Thread 안정성을 직접 책임져야함( `UObject*`나 GT 객체의 참조를 캡쳐하지 말고, 값 복사본을 넘길 것)
- 예시 : 간단한 렌더링 상태 갱신, Compute Shader Pass 실행 시 GT에서 계산한 데이터를 안전하게 전달. -그래서 &Value로 안함(?)

#### ③ Dynamic DataUpdate - 변경 데이터만 전달
- `MarkRenderDynamicDataDirty()` → `SendRenderDynamicData_Concurrnet()`계열
- Proxy 전체를 재생성하지 않고, **기존 Proxy에 변경된 데이터만 갱신** 
- 예시 : 매 프레임 변하는 parameter, 동적 지형 데이터

#### ④ GPU Resource Update - GPU 리소스 갱신
- GT 측 데이터 → Rendering Resource update 경로 → GPU buffer / Texture
- Vertex Data, Instance Data, Texture 등 GPU가 읽을 데이터를 업로드
- 실제 업로드는 엔진의 리소스 관리 및 RHI 경로를 통해 수행됨
- 예시 : ComputeShader Inputbuffer, 변형 데이터, GPU instance data

#### ⑤RDG - RenderPass와 GPU 작업 구성
- RT 코드 → `FDGBuilder` → Pass 등록 → GPU `execute()`
- **RDG자체는 GT→RT전달 방식은 아니다**. 
  RT에 도착한 데이터로 GPU 작업을 구성, 리소스 의존성을 관리하는 프레임워크!!
 - 예시 : ComputeShader, UAV, RenderTarget, GPU 기반 Tessellation

GT → `ENQUEUE_RENDER_COMMAND(...)` → RT

---

### 4. 전달 흐름 정리
**GT측 (보내는 쪽)**:  
- `ENQUEUE_RENDER_COMMAND`로 RT로 보내며, Lambda로 캡쳐해서 전달
- 렌더링용 **Data Packet**을 구성해서 보낸다.
  (데이터를 별도의 Proxy나 렌더링용 데이터 구조로 복사해서) 전달한다.




``` cpp
//GameThread에서 보내는 방식
ENQUEUE_RENDER_COMMAND( CommandName )(
	[variable])(FRHICommandListImmediate& RHICmdList)
	{
		//RT에서 CapturedValue 사용
	});
```

**RT측 (받는 쪽)** :  
- `Proxy`(RT)에서 `UpdateInteractionState_RenderThread`같은 `_RenderThread`접미사 함수로 받아 처리

---

### 5. 한눈에 보기


|     | 패턴                     | 전달 대상              | 핵심                   |
| --- | ---------------------- | ------------------ | -------------------- |
| ①   | Component → SceneProxy | Mesh/Material/파라미터 | 렌더링 전용 객체로 복사        |
| ②   | Render Command         | 실행할 작업             | Lambda 값 캡처로 RT에서 실행 |
| ③   | Dynamic Data Update    | 변경분                | Proxy 재생성 없이 갱신      |
| ④   | GPU Resource Update    | Buffer/Texture     | RHI 경로로 업로드          |
| ⑤   | RDG                    | (전달 아님)            | RT에서 GPU Pass 구성     |
