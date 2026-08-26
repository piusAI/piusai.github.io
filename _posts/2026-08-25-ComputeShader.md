---
layout: post
published: true
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


|            | Graphics Pipeline (DrawCall)                    | Compute Shader Pipeline(Dispatch)                                                 |
| ---------- | ----------------------------------------------- | --------------------------------------------------------------------------------- |
| 시작 명령      | `Draw()`, `DrawIndexed()`                       | Dispatch()                                                                        |
| 입력 Data    | Vertex Buffer, Index Buffer                     | 임의의 Buffer/Texture(UAV, SRV)                                                      |
| Piepline순서 | IA→VS→(HS/DS/GS)→SO→RS→PS→OM                    | 없음<br>(Thread group이 곧 바로 실행)                                                     |
| 최종 출력      | RenderTarget  <br>(OM에서 Depth Test/Blend 거쳐 기록) | UAV(RWTexture2D, RWBuffer 등)<br>에 Shader가 직접 Write                                |
| 존재하는 것     | `SV_Position`, `SV_Target`, `SV_Depth`          | `SV_DispatchThreadID`,<br>`SV_GroupThreadID`,<br>`SV_GroupID`,<br>`SV_GroupIndex` |



### GPU Compute Shader

 1. VRAM에 Texture(RWTexture2D 등)나 Buffer(RWStructedBuffer)라는 판을 먼저 쫙 깔아둔다
 2. **CPU →GPU 지시**: CPU가 전체 가로 / 세로 크기에 맞춰 필요한 Group개수(`GroupCount`)를 계산하고 `Dispatch()`를 호출
 3. **GPU 병렬 연산** : 수천개의 GPU 코어가 각 Texel/Voxel 마다 전담하여 HLSL Shader Shader 코드를 수행한다.
    (VRAM의 위치 (DispatchThreadID.xyz)로 접근해서 모든 voxel이 동시에 연산 수행)

##### Compute Shader의 순서 간략히 정리 

> Dispatch() 호출 `cpu`
> → GPU가 Thread Group x numthreads개의 Thread를 실행  
> → 각 Thread가 UAV(OutTexture)에 직접 값 write 


##### Semantic
Shader의 입출력에 첨부되는 문자열로 Parameter의 사용 목적에 대한 정보를 전달한다

| Semantic            | 설명                                      |
| ------------------- | --------------------------------------- |
| SV_DispatchThreadID | 전체 Dispatch에서의 전역 좌표                    |
| SV_GroupThreadID    | Group내부에서의 Thread 로컬 좌표 (0~numthread-1) |
| SV_GroupID          | Thread가 속한 Group자체의 전역 좌표               |

## Compute Shader Thread 구조

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/SemanticHLSL/ThreadConfig.jpg" alt="CD001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>ThreadConfig</strong> </td>  </tr> </table>
#### 예시 Texture -> Thread 흐름
> Texture 256x256
>    ↓
> 총 65,536개의 Texel
>    ↓
> 보통 Texel 하나를 Thread 하나가 처리하도록 구성
>    ↓
> Threads 65,536개 필요

- 하나로 GPU로 처리 빡하면 좋겠지만, Thread Group으로 묶어서 실행을 한다

```
[numthreads(8,8,1)]
```
이라면 ThreadGroup이 (8 x 8 x 1) = 64개 Thread를 가진다

그리고 256x256의 텍스쳐를 처리하려면
> CPU ->GPU에 Dispatch 수치를 알려준다

```
TextureXsize / (GroupThread_X)
```
256 / 8 =32 

``` cpp
Dispatch(32, 32, 1)
```

Group수 : 32 x 32  
Group내 Threads 수 : 8 x 8  
총 Threads 수 : 256 x 256 (Texture와 1:1)

####  Dispatch Thread ID ?
진짜 내 Thread가 어디인가?
각 Thread의 전역 주소

``` cpp
(GroupID x NumThreads) + GroupThreadID
```
> 절대적인 방 주소 (울산 동구 일산지 자이 A동 B호)


####  비유로 알아보는 Thread

Group을 층으로 비유한다면
- Group ID : 현재 몇층?
- numthreads(= Group Threads) : 한 층당 총 방 개수
- Group Thread ID : 그 층에서의 방 번호
- DispatchThreadID :  * 동 * 호


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