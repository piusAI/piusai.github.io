---
layout: post
published: true
title:  "RenderPipeline?"
date:   2026-05-29 15:32:00 +0900
description: C++의 OOP를 제거하고 오직 그래픽스 렌더파이프라인에만 집중해본 연구 기록입니다.
img: pius-icon.png                  # 목록에 출력될 썸네일 이미지 (필요시 사용)
categories:
  - R&D
tags:
  - DX11
  - Graphics
  - RenderPipeline
author: PIUS
---
##  RenderPipeline?

![Render Pipeline Image]({{ '/assets/img/Renderpipeline.jpg' | relative_url }})
정확히는 Rasterizer RenderPipeline(DX11 Render Pipeline)이다.

>**모든 실시간 Grpahics의 목적**
>GPU 메모리 상의 BackBuffer에 현재 프레임을 생성하고, SwapChain을 활용해 이를 화면에 표시하는 것.
>RenderPipeline은 그 한장의 이미지를 그리기위해, Vtx를 Pixel로 변환하는 과정이다.

---
개별 Stage마다 작은 O, X를 표시해 두었는데 이는 프로그래밍 가능한지 불가능한지로 나뉘어 진다. 프로그래밍 불가능하다는것은 코드로 칠수없다는 뜻이 아닌, 프로그래머가 조작해서 커스텀하게 수정 불가능함을 의미한다. 그렇다는것은 그럼에도 불구하고 꼭 필요한 구조체들을 명시해줘야하는 것은 변함없다.

### KeyWord
아래는 DirectX11의 여정을 시작하기전에 짚고 가야하는 키워드들이다

**Shader** : 프로그래밍 가능한 단계, GPU에서 실행되는 프로그램
**Pipeline** : Shader들의 결합체
**HEL** : Hardware <u>Emulation</u> Layer ; Software Emulation
**HAL** : Hardware <u>Abstraction</u> Layer ; 프로그래밍 가능한 영역
**COM** : Component Object Models ;
예시, `ID3D11Device* Device', 'ID3D11DeviceContext* DeviceContext`,`IDXGISwapChain* SwapChain` 
실제 구현부는 DX 런타임 내부에 있고 Interface Pointer만 받는 구조

**FrameBuffer** :  GPU가 최종적으로 생성한 Pixel Data를 저장하는 Memory 영역이다.
각 픽셀은 화면 해상도(Resolution)에 대응하며, 픽셀마다 Color, Alpha, Depth등의 정보가 저장된다. 우리가 모니터를 통해 보는 화면은 결국 이 Frame Buffer에 기록된 데이터를 출력한 결과이다.

![mnist]({{ '/assets/postimg/RenderPipeline/mnist.jpg' | relative_url }})
위 MNIST 이미지는 0~255 범위의 8비트 값을 이용해 숫자를 표현한 예시이다. Frame Buffer 역시 본질적으로 pixel마다 수치 데이터를 저장 하고 있고, GPU는 이 수치들을 색상으로 해석하여 화면에 출력한다.

![MatrixFrameBuffer]({{ '/assets/postimg/RenderPipeline/MatrixFrameBuffer.jpg' | relative_url }})
영화 *The Matrix* 에서 실제 Frame Buffer가 저런 형태로 저장되는 것은 아니겠지만, "화면은 결국 수 많은 **데이터의 집합**으로 이루어져 있다"는 개념을 직관적으로 이해하는 비유로는 재미있다.

Frame Buffer는 GPU 메모리(VRAM)에 존재하는 Render Target으로, 화면에 출력될 픽셀 Data를 저장하는 Buffer이다.

### 📌 DCC Workflow

![Dcc Workflow]({{ '/assets/postimg/RenderPipeline/DccWorkFlow_001.jpg' | relative_url }})



| Step 01 | Step 02 |
|----------|----------|
| <img src="{{ '/assets/postimg/RenderPipeline/DccWorkFlow_002.jpg' | relative_url }}" width="250"> | <img src="{{ '/assets/postimg/RenderPipeline/DccWorkFlow_003.jpg' | relative_url }}" width="250"> | 

| Step 03 | Step 04 | 
|----------|----------|
| <img src="{{ '/assets/postimg/RenderPipeline/DccWorkFlow_004.jpg' | relative_url }}" width="250"> | <img src="{{ '/assets/postimg/RenderPipeline/DccWorkFlow_005.jpg' | relative_url }}" width="250"> | 


---
Art강의할때, 어떻게 하면 처음 3D를 접하는 사람들에게 쉽게 이해할수 있게 할까를 고민하고 고민한 결과 위 PDF가 가장 이해시키기 좋았다. Art작업에서의 Workflow이다. RenderPipeline과 대응하면서 이해할만큼의 매치가 되지는 않지만, RenderPiepeline에 들어가기전에 이 작업을 이해한 상태에서 들어갔기때문에 조금 더 쉽게 이해했던것 같다. 


그 기억을 되살려서, 간략화한 단계들을 먼저 이야기 해보겠다.

**IA->VS->HS(option)->T->DS(option)->GS(Optional)->SO->RS->PS->OM** 라는 단계가 있다. 

좀 더 간략화하자.

**IA->VS->T->SO->RS->PS->OM** 
이렇게 간략화 할 수있다.

그래픽스를 조금 더 거시적으로 본다면
> Graphics는 GPU가 메모리상의 Render Target에 이미지를 생성하고, 이를 Present()로 Display에 출력하는 과정이다.

`Scene Data → GPU가 Render Target(Back Buffer)에 그림 → 완성된 Back Buffer를 Present
→ Swap Chain이 화면에 표시 → 모니터가 스캔아웃`

위는 도식화하였다

### 01 IA :Input Assembler(X)
Input Assembler, Vtx Buffer, Index Buffer를 활용해서 위치를 들고있다.

### 02 VS : Vertex Shader(O)
최종 Camera에 projection시키기 위한 Vtx 행렬 계산
주요한 Matrix 변환 : (Local Transform -> World Transform -> ViewTransform -> ScreenProjection)


### 03 HS : Hull Shader(O)

### 04 T : Tessellator (X)

### 05 DS : DomainShader (O)

### 06 GS : Geometry Shader (O)

### 07 SO : Stream Output(X)

### 08 RS : Rasterizer(X)

### 09 PS : PixelShader(O)

### 10 OM : Output Merger(X)


--- 블로그 작성중

{% highlight cpp %}

#include <iostream>
using namespace std;
int main()
{
    cout << "Test Home" << endl;
    return 0;
}
{% endhighlight %}  

---


### 📌 학습 및 구현 참조 레퍼런스


* [Braynzar Soft DX11][braynzar] : C++의 OOP 구조를 걷어내고, 오직 렌더파이프라인 자체에만 집중해서 로직을 구현해 볼 수 있었습니다.

* [Rastertek DX11][Rastertek] : 전체적인 그래픽스 엔진 프레임워크를 설계하고 빌드하면서 깊이 있게 공부해볼 수 있습니다.

  

[braynzar]: https://www.braynzarsoft.net/viewtutorial/q16390-4-begin-drawing

[Rastertek]: https://www.rastertek.com/dx11win10tut04.html