---
layout: post
published: false
title: Compute Shader의 이해
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-07-07 15:32:00 +0900
description: Compute Shader의 이해
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---
Compute Shader Pipeline과 Graphics Pipeline과 비교를 먼저해보자.


|            | Graphics Pipeline (DrawCall)              | Compute Pipeline(Dispatch)                     |
| ---------- | ----------------------------------------- | ---------------------------------------------- |
| 시작 명령      | `Draw()`, `DrawIndexed()`                 | Dispatch()                                     |
| 입력         | Vertex Buffer, Index Buffer               | 임의의 Buffer/Texture(UAV, SRV)                   |
| Piepline순서 | IA->VS->(HS/DS/GS)->SO->RS->PS->OM        | 없음, Thread group이 곧 바로 실행                      |
| 최종 출력      | RenderTarget, OM에서 Depth Test/Blend 거쳐 기록 | UAV(RWTexture2D, RWBuffer 등)에 Shader가 직접 Write |
| 존재하는 것     | SV_Position, OM, DepthStencil             | SV_DispatchThreadID, SV_Dispatch*              |


### GPU Compute Shader

 VRAM에 Texture(RWTexture2D 등)나 Buffer(RWStructedBuffer)라는 판을 먼저 쫙 깔아둔다
 연산 - 수천개의 GPU 코어가 각 voxel 마다 전담하여 HLSL Shader 코드가 VRAM의 위치 (DispatchThreadID.xyz)로 접근해서 모든 voxel이 동시에 연산 수행한다

Compute Shader의 순서는 다음가ㅗ 같다

> Dispatch() 호출 `cpu` ->GPU가 Thread Group x numthreads개의 Thread를 실행 ->각 Thread가 UAV(OutTexture)에 직접 값 write 



이를 읽기위해 쓰이는 Semantic
Shader의 입출력에 첨부되는 문자열로 Parameter의 사용 목적에 대한 정보를 전달한다

| Semantic            | 설명                                      |
| ------------------- | --------------------------------------- |
| SV_DispatchThreadID | 전체 Dispatch에서의 전역 좌표                    |
| SV_GroupThread      | Group내부에서의 Thread 로컬 좌표 (0~numthread-1) |
| SV_GroupID          | Thread가 속한 Group자체의 전역 좌표               |


##  Dispatch Thread ID?

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/SemanticHLSL/DispatchThreadID.png" alt="CD001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Dispatch ThreadID</strong> </td>  </tr> </table>


GPU에서 Thread 하나는 **단일 실행**되는 Shader Instance이다.

>이 **Mesh (vtx data)** 를 Shader / Texture / Material 을 써서 이 위치에 그려라


---

### Compute Shader, GPU로 병렬적으로 처리하는 이유와 활용도

#### 001 왜 굳이 Thread를 활용해서 GPU로 병렬적으로 활용해야할까?
이는 너무 간단하다. GPU를 Shader로 가져와야하는 이유다.  
Material의 각 Texel로 가져와서 개별당 하나씩 단일 실행되게 만들기 위헤서이다.

1. `CPU` : "이 Texture는 512x512이니, 512x512개의 thread 필요하다" ->Dispatch group 개수 계산 후 Dispatch() 호출
2. `GPU` : numthreads크기만큼 묶어서 필요한만큼 Thread를 병렬로 띄운다
3. 각 Thread는 `DispatchThreadID`로 처리할 "Texel좌표"를 자동으로 받음
4. 그 대응하는 좌표 하나만 계산
5. 모든 Thread가 끝나면 Texture 전체가 완성

Material Node에 있는 Pixel Shader나 Vertex Shader는 GPU로 병렬적으로 처리안하는것일까?  
또 그렇지 않다, 대신 RenderPipeline을 거치니, DrawCall()을 꼭 한다.
#### 002 왜 굳이 Compute Shader를 사용해야하나?
그렇다면 Pixel Shader나 Vertex Shader와 다른 점은 무엇인가?  
앞서 말한 것과 같이, Compute Shader는 `DrawCall()`을 하지 않기에 그와 완전히 무관하게 `Dispatch(GroupCountx, GroupCountY, 1)`로 원하는 개수만큼 Thread를 실행해줘라고 GPU에 지정할 수있다 (Drawcall이라고 명명 x) Mesh도 필요없고 화면에 뭘 그리는 것도 아니다,
이 Texture/Buffer는 이런 규칙으로 채워라는 범용 병렬 연산이다.

