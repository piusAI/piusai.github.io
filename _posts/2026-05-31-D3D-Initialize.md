---
layout: post
published: true
title: DirectX3D 초기화
date: 2026-05-31 15:32:00 +0900
description: D3D 초기화 7전8기 공부!
img: pius-icon.png
categories:
  - R&D
tags:
  - DX11
  - Graphics
  - RenderPipeline
author: PIUS
---

## Direct3D의 초기화

DirectX를 처음 접하게 되면 여러 큰 벽들을 마주하게 되는데, 그중 하나가 D3D의 초기화이다.

Rastertek으로 이해도 되지 않는 긴 글을 읽으면서 코드를 따라 치다 보면, 지금 내가 어딘가 사경을 헤매고 있다는 사실을 깨닫게 된다. 내 손은 하염없이 구조체를 옮겨 적으며 _'꼭 필요하다고 하니 그렇겠지'_ 하면서 그냥 따라치고 있다. 그래서 지금 어디쯤인가 스스로에게 반문할 때 그 답을 할 수가 없다.

나는 이 지점, **"필요하겠으니 복사 붙여넣기 하자"** 에서 탈피하고자 여러 번 설명을 읽고 이해하려 노력했다. 크게 네 가지 학습으로 나누었다.

1. DirectX 학부 수업
2. RasterTek D3D Tutorial
3. DirectX11 게임 프로그래밍 입문 서적 Chapter4
4. BryanzarSoft D3D tutorial


### D3D 초기화 정리
  ![D3DInitIMG]({{ '/assets/postimg/D3DInit/D3DInitialize001.jpg' | relative_url }})

그리고 마침내 나만의 정리를 했다.

---
어떤 구조체가 "무조건 있어야 한다"는 말은 가볍게 _그렇구나_ 하고 넘겼다.
RenderPipeline에서 중요한 단계가 **'IA->VS->HS->T->DS->GS->RS->PS->OM'이다** 인 것처럼, D3D init에서도 핵심 흐름만 머릿속에 남겼다.

|단계     |  내용   |
| --- | -------- |
|  01 02   |   Interface 및 사용자 Device 검사   |
|  03 04   |   Swapchain 준비, Backbuffer를 확보   |
|  05   |   확보된 backbuffer로 RTV생성  |
|  06   |   DSV 준비  |
|  07   |   RenderPipeline의 마지막 OM에 등록  |
|  08   |  Viewport설정   |


결국 삼각형을 여러 번 띄워보는 프레임워크를 직접 만들고 나서야 비로소 전체 흐름이 눈에 들어왔다.

--

  

### 📌 학습 레퍼런스

[braynzar : D3D 초기화][braynzar-D3Dinit] : OOP없이 D3D초기화 자체에만 집중 가능
[Rastertek : D3D 초기화][Rastertek-D3Dinit] : FrameWork를 기반으로 이식성 높게 작업 가능


[braynzar-D3Dinit]: https://www.braynzarsoft.net/viewtutorial/q16390-3-initializing-direct3d-11

[Rastertek-D3Dinit]: https://www.rastertek.com/dx11win10tut04.html
